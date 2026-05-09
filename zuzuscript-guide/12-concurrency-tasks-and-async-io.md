# Chapter 12: Concurrency, Tasks, and Async I/O

In Chapter 11, we moved side effects to clean module boundaries:
files through `std/io`, processes through `std/proc`, and HTTP through
`std/net/http`.

That is the right setup for concurrency.

Most script concurrency is not about doing heavy CPU work at the same
time. It is about not waiting idle while the runtime is already waiting
for:

- an HTTP response,
- a child process,
- a timer,
- or a file operation.

ZuzuScript's first concurrency model is cooperative async I/O:
tasks share one runtime thread, and they switch at explicit `await`
points. That keeps the model close to JavaScript's async functions and
Promises, but with ZuzuScript's block syntax.

This chapter describes the portable task model and the worker API that
builds on it for CPU-heavy work.


## 12.1 The small model

The main pieces are:

- `async function` for functions that return a `Task`,
- `await { ... }` for waiting on a task and unwrapping its result,
- `spawn { ... }` for starting concurrent task work,
- `std/task` helpers and classes such as `sleep`, `all`, `race`,
  `timeout`, `Channel`, and `CancellationSource`,
- awaitable standard-library APIs such as HTTP `_async` methods,
  `Proc.run_async`, and `Path.slurp_utf8_async`.

`await` and `spawn` always take blocks.

```zzs
let response := await {
	ua.get_async(url);
};

let worker := spawn {
	let page := await {
		ua.get_async(url);
	};
	page.status();
};
```

The block is expression-valued. Its last expression is the value of the
block, like `do { ... }`.

For `await`, that last expression must be awaitable:

```zzs
let text := await {
	file.slurp_utf8_async();
};
```

For `spawn`, the block is the task body:

```zzs
let task := spawn {
	await {
		sleep(0.05);
	};
	"ready";
};

say await {
	task;
};
```

That prints `ready`.


## 12.2 Async functions return tasks

Calling an async function starts task-shaped work and gives you a
`Task` value.

```zzs
from std/task import sleep;

async function answer_later () {
	await {
		sleep(0.01);
	};
	return 42;
}

async function __main__ () {
	let task := answer_later();

	let answer := await {
		task;
	};

	say answer;
}
```

The important split is:

- calling `answer_later()` gives you a task,
- awaiting that task gives you the value `42`.

Inside named functions, use `await` from async functions. At script
entry, use `async function __main__ ( argv )` for async code; the CLI
awaits it after loading the script. Top-level `await` is not the
documented script entrypoint style.


## 12.3 Await is where interleaving happens

ZuzuScript Phase A scheduling is cooperative.

That means a task keeps running until it reaches an await point, returns,
throws, or is cancelled. Other tasks do not interrupt arbitrary
expressions.

```zzs
from std/task import sleep;

async function one () {
	say "one: start";
	await {
		sleep(0.01);
	};
	say "one: end";
}

async function two () {
	say "two: start";
	await {
		sleep(0.01);
	};
	say "two: end";
}

async function __main__ () {
	let a := spawn {
		await {
			one();
		};
	};

	let b := spawn {
		await {
			two();
		};
	};

	await {
		a;
	};
	await {
		b;
	};
}
```

The tasks can overlap while they are sleeping. A CPU-heavy loop with no
await points still blocks the runtime until it finishes. Worker support
is the better fit for CPU parallelism.


## 12.4 `spawn { ... }` creates independent work

Use `spawn` when the current task should keep moving while another
piece of work runs.

```zzs
from std/task import sleep;

async function __main__ () {
	let background := spawn {
		await {
			sleep(0.10);
		};
		"done";
	};

	say "started";

	let result := await {
		background;
	};

	say result;
}
```

Spawned tasks are independent until you await them, cancel them, or pass
them to a helper such as `all`, `race`, or `timeout`.

This follows JavaScript's Promise style closely:

- a spawned task's failure is observed when the task is awaited or
  composed,
- dropping the task value does not automatically await it,
- the runtime may still cancel unfinished background work during
  shutdown.

So do not casually ignore spawned task values. If the task matters,
store it and observe it.


## 12.5 `all` waits for every task

Import `all` from `std/task`.

```zzs
from std/net/http import UserAgent;
from std/task import all;

async function fetch_many (Array urls) {
	let ua := new UserAgent(timeout: 5);
	let tasks := [];

	for ( let url in urls ) {
		tasks.push(ua.get_async(url));
	}

	return await {
		all(tasks);
	};
}
```

`all(tasks)` returns one task that:

- waits for every input task,
- resolves to an array of results,
- preserves the input order,
- fails if any input task fails.

This is useful when you need the whole set before continuing:

- fetch three URLs, then compare them,
- start two processes, then combine their output,
- read config and data files, then build one object.

Without `all`, every script would need to hand-write "wait for N things"
logic. That logic is easy to get subtly wrong, especially around
failures and cancellation.


## 12.6 `race` waits for the first completion

Import `race` from `std/task`.

```zzs
from std/task import race, sleep;

async function with_fallback ( work ) {
	return await {
		race( [
			work,
			sleep(1),
		] );
	};
}
```

`race(tasks)` returns one task that:

- resolves or fails with the first task that completes,
- cancels unfinished losing tasks,
- propagates the winning result or error.

Loser cancellation is part of the contract. If later completion matters,
keep those tasks out of `race` and await them explicitly.

This is useful when the first answer is the only answer you need:

- use whichever mirror responds first,
- stop waiting once one strategy succeeds,
- enforce a "first event wins" workflow,
- combine a real task with a timeout-like task.

ZuzuScript also provides `timeout(seconds, task)` for the common timeout
case. Prefer `timeout` when you mean "this one task must finish soon";
use `race` when multiple alternatives are genuinely competing.


## 12.7 Why `all` and `race` are not keywords

`await` and `spawn` are language forms because they affect evaluation:
they create or suspend task frames.

`all` and `race` are task combinators. They are imported from
`std/task`:

```zzs
from std/task import all, race;
```

Keeping them in a module has practical advantages:

- code only imports task helpers when it uses them,
- the API can grow without expanding the keyword set,
- runtimes can still implement them efficiently behind the module,
- user code reads like ordinary function composition.

They are still necessary because they need well-defined shared behavior
across runtimes: input ordering for `all`, loser cancellation for
`race`, and consistent error propagation for both.


## 12.8 Async HTTP

`std/net/http` exposes awaitable methods on `UserAgent` and `Request`.

```zzs
from std/net/http import UserAgent;
from std/task import all;

async function load_pair () {
	let ua := new UserAgent(timeout: 5);

	let responses := await {
		all( [
			ua.get_async("https://example.com/a.json"),
			ua.get_async("https://example.com/b.json"),
		] );
	};

	return [
		responses[0].expect_success().json(),
		responses[1].expect_success().json(),
	];
}
```

Available async request methods include:

- `send_async(request)`,
- `request_async(method, url, data?, headers?)`,
- `get_async(url, headers?)`,
- `head_async(url, headers?)`,
- `delete_async(url, headers?)`,
- `options_async(url, headers?)`,
- `post_async(url, data?, headers?)`,
- `put_async(url, data?, headers?)`,
- `patch_async(url, data?, headers?)`,
- `request.send_async(user_agent)`.

The async methods resolve to the same response objects as the
synchronous methods.


## 12.9 Async process helpers

`std/proc` keeps the existing synchronous process APIs and adds
awaitable twins for process execution.

```zzs
from std/proc import Proc;
from std/task import all;

async function run_tools () {
	let results := await {
		all( [
			Proc.run_async( "perl", [ "-e", "print qq<left\\n>;" ] ),
			Proc.run_async( "perl", [ "-e", "print qq<right\\n>;" ] ),
		] );
	};

	return results[0]{stdout} _ results[1]{stdout};
}
```

The main awaitable process APIs are:

- `Proc.run_async(command, argv?, options?)`,
- `Proc.pipeline_async(commands, options?)`,
- `sleep_async(seconds)`.

`std/proc` also exports synchronous `sleep(seconds)`. Use that only in
synchronous scripts. In async code, prefer `sleep(seconds)` from
`std/task` or `sleep_async(seconds)` from `std/proc` so the scheduler
can run other tasks while waiting.


## 12.10 Async file operations

`std/io` `Path` objects include awaitable file I/O methods.

```zzs
from std/io import Path;
from std/task import all;

async function load_config_and_data () {
	let config := new Path("config.json");
	let data := new Path("data.txt");

	let pair := await {
		all( [
			config.slurp_utf8_async(),
			data.slurp_utf8_async(),
		] );
	};

	return {
		config: pair[0],
		data: pair[1],
	};
}
```

Text helpers:

- `slurp_utf8_async()`,
- `lines_utf8_async()`,
- `spew_utf8_async(text)`,
- `append_utf8_async(text)`.

Binary helpers:

- `slurp_async()`,
- `lines_async()`,
- `spew_async(bytes)`,
- `append_async(bytes)`.

The async methods return tasks. Await them directly or compose them with
`all`, `race`, or `timeout`.


## 12.11 Timeouts

Use `timeout(seconds, task)` when one awaitable operation has a maximum
acceptable wait.

```zzs
from std/net/http import UserAgent;
from std/task import timeout;

async function fetch_quickly (String url) {
	let ua := new UserAgent(timeout: 10);

	try {
		return await {
			timeout( 2, ua.get_async(url) );
		};
	}
	catch ( TimeoutException e ) {
		warn "HTTP request timed out";
		return null;
	}
}
```

The HTTP client's own timeout setting still matters for the underlying
network operation. The task timeout is the ZuzuScript-level limit around
the awaitable task.


## 12.12 Cancellation

Tasks can be cancelled directly.

```zzs
from std/task import sleep;

async function __main__ () {
	let task := sleep(60);
	task.cancel("not needed");

	try {
		await {
			task;
		};
	}
	catch ( CancelledException e ) {
		say e.to_String();
	}
}
```

For coordinated cancellation, construct `CancellationSource`.

```zzs
from std/task import CancellationSource, sleep;

async function __main__ () {
	let source := new CancellationSource();
	let task := spawn {
		await {
			sleep(60);
		};
		"finished";
	};

	source.token().watch(task);
	source.cancel("stopping early");

	try {
		await {
			task;
		};
	}
	catch ( CancelledException e ) {
		say e.to_String();
	}
}
```

The source owns the cancellation decision. The token is the signal that
other code can watch or query.

Cancellation unwinds task cleanup. If a task has cleanup guards or other
finalization work in scope, that cleanup runs as the task exits.


## 12.13 Channels

Channels are simple FIFO message queues from `std/task`.

```zzs
from std/task import Channel;

async function producer ( ch ) {
	await {
		ch.send("ready");
	};
	ch.close();
}

async function __main__ () {
	let ch := new Channel();

	let task := spawn {
		await {
			producer(ch);
		};
	};

	say await {
		ch.recv();
	};

	await {
		task;
	};
}
```

Channel rules in Phase A:

- `send(value)` returns an awaitable task,
- `recv()` returns an awaitable task,
- `close()` closes the channel,
- sending to a closed channel throws `ChannelClosedException`,
- receiving from a closed and drained channel resolves to `null`.

That `null` result is the documented end-of-stream value for a closed,
empty channel.

Channels are useful when tasks should communicate by messages instead
of sharing mutable state.


## 12.14 Workers for CPU-heavy work

Import `Worker` from `std/worker` when the work should run in an
isolated runtime instead of the current cooperative scheduler.

```zzs
from std/worker import Worker;

async function __main__ () {
	let task := Worker.spawn(
		function ( n ) {
			let total := 0;
			for ( let i := 0; i < n; i++ ) {
				total += i;
			}
			return total;
		},
		[ 100000 ],
	);

	say await {
		task;
	};
}
```

If `std/worker` imports successfully, workers are available in that
host.

`Worker.spawn(callable, args?, ...options)` returns a normal `Task`.
Awaiting that task gives the worker's return value. If the worker
throws, cannot unmarshal its input, cannot marshal its result, or is
cancelled, awaiting the task throws.

Workers are shared-nothing. Values crossing the boundary are copied
through `std/marshal`, so ordinary collections, classes, traits,
functions, methods, and user objects can move when marshal supports
them. Live runtime resources such as `Task`, `Channel`, open files,
sockets, process handles, HTTP clients, database handles, and native
host objects are not transferable unless `std/marshal` explicitly gains
support for them.

The worker boundary is for cooperating trusted code. It uses
`std/marshal.load`, which may evaluate marshalled code records, so it is
not an untrusted data sandbox.

Workers inherit denied capabilities from the parent runtime and may
deny more:

```zzs
let task := Worker.spawn(
	function () {
		from std/io import Path;
		return new Path("secret.txt").slurp_utf8();
	},
	[],
	deny_fs: true,
);
```

Supported denial options match the runtime capability flags:
`deny_fs`, `deny_net`, `deny_proc`, `deny_db`, `deny_clib`,
`deny_gui`, `deny_worker`, `deny_js`, and `deny_perl`. Passing `false`
never restores a capability denied by the parent.


## 12.15 Worker message handles

Use `Worker.spawn_handle` when a worker needs an explicit parent/worker
message channel.

```zzs
from std/worker import Worker;

async function __main__ () {
	let handle := Worker.spawn_handle(
		async function ( inbox ) {
			let value := await {
				inbox.recv();
			};

			await {
				inbox.send( value * 2 );
			};

			inbox.close();
			return "done";
		},
		[],
	);

	await {
		handle.send(21);
	};

	say await {
		handle.recv();
	};

	say await {
		handle.result();
	};
}
```

`Worker.spawn_handle(callable, args?, ...options)` calls
`callable(inbox, ...args)` inside the worker and returns a
`WorkerHandle` in the parent.

The parent side provides:

- `handle.send(value)` to send a marshal-copied message to the worker,
- `handle.recv()` to receive the next worker message,
- `handle.close()` to close the parent-to-worker send direction,
- `handle.cancel(reason?)` to request cancellation,
- `handle.result()` to get the worker result task,
- `handle.status()` and `handle.done()` to inspect the result task.

The worker-side `inbox` provides `send`, `recv`, and `close`.

Unlike `std/task.Channel.recv`, worker `recv()` rejects with
`ChannelClosedException` after the peer has closed and all queued
messages are drained. This is because `null` is a valid worker message.

Workers may return any marshalable value. `std/result` provides a
subclassable `Result` class for scripts that want a conventional
`ok`/`err` payload shape, but workers are not required to return a
`Result`.


## 12.16 Debugging async scripts

The usual `debug` statement and global `DEBUG` value still apply.

Run a script with debug enabled:

```sh
bin/zuzu -d script.zzs
```

Use level 2 when you want blocking-operation warnings from async tasks:

```sh
bin/zuzu -d2 script.zzs
```

When debug mode is enabled, the Perl scheduler records task trace
information such as:

- task id,
- parent task id,
- task name,
- creation file and line,
- status changes,
- blocking native operations used from async task context.

The most common warning means:

> This async task called a synchronous operation that may block the
> scheduler.

The fix is usually to switch to the awaitable API:

- `Path.slurp_utf8_async()` instead of `Path.slurp_utf8()`,
- `Proc.run_async(...)` instead of `Proc.run(...)`,
- `ua.get_async(...)` instead of `ua.get(...)`,
- `sleep(...)` from `std/task` or `sleep_async(...)` from `std/proc`
  instead of synchronous `sleep(...)` from `std/proc`.


## 12.17 Checklist and pitfalls

Before finishing async code, ask:

- Did every `await` and `spawn` use a block?
- Did I await or otherwise observe every important spawned task?
- Did I use `all` when I need every result?
- Did I use `race` only when the first completion really wins?
- Did I use `timeout` for ordinary per-task time limits?
- Did I avoid synchronous file, HTTP, process, and sleep calls inside
  async code?
- Did I move CPU-heavy work from cooperative tasks into workers where
  host support is available?
- Did I only send marshalable values across the worker boundary?
- Did I close or cancel long-lived worker handles?
- Did cancellation have a clear owner?

Common pitfalls:

1. **Forgetting that async functions return tasks**
   Call the function to create work, then `await { task; }` to get its
   value.
2. **Spawning and ignoring failures**
   If the work matters, keep the task and await it or compose it.
3. **Using `race` as a casual timeout**
   Use `timeout(seconds, task)` when timeout is the real intent.
4. **Blocking the scheduler**
   Synchronous APIs still work, but they are the wrong tool inside async
   task code.
5. **Expecting CPU parallelism**
   Async I/O overlaps waiting. It does not make CPU loops run on
   multiple cores. Use `std/worker` for isolated CPU-heavy work.
6. **Treating workers as a sandbox**
   Worker capability denial is a Zuzu runtime policy, not portable OS
   isolation for untrusted code.


## 12.18 Wrap-up

Concurrency in ZuzuScript starts with a small rule:

> Start tasks explicitly, and wait explicitly.

`async function` gives you task-returning functions.
`spawn { ... }` starts independent work.
`await { ... }` unwraps results at clear suspension points.
`all`, `race`, and `timeout` cover the common coordination patterns.
Channels and cancellation give longer-running workflows a way to stay
orderly. `std/worker` adds shared-nothing parallel work when CPU-heavy
code needs to leave the cooperative scheduler.

In the next chapter, we'll pull the whole guide together:
collections, queries, errors, modules, concurrency, and the everyday
style choices that keep scripts readable after they become useful.
