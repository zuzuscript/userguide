# Chapter 14: Concurrency, Tasks, and Workers

Chapter 13 showed how to make data flow readable in one expression.

This chapter is about letting more than one piece of work be in progress
at the same time.

Concurrency is not always about making a CPU run harder. Often it is
about not sitting idle while something else is waiting:

- a timer,
- a message,
- a child operation,
- or work that belongs in another runtime.

ZuzuScript has two related tools:

- cooperative tasks, from the language and `std/task`,
- isolated workers, from `std/worker`.

Tasks are for structured asynchronous work in one runtime. Workers are for
shared-nothing parallel work in another runtime.

This chapter covers:

- `async function`,
- `await { ... }`,
- `spawn { ... }`,
- the `Task` type,
- `std/task` helpers such as `sleep`, `yield`, `all`, `race`, `timeout`,
  `Channel`, and `CancellationSource`,
- when cooperative scheduling does and does not interleave work,
- and `std/worker` for CPU-heavy or isolated work.

We will avoid deep HTTP and filesystem examples here. Later chapters cover
IO and network APIs in their own right.


## 14.1 The small model

The basic model is:

- an `async function` returns a `Task`,
- `await { ... }` waits for a task and unwraps its result,
- `spawn { ... }` starts a new task,
- `std/task` provides helpers for timing, coordination, cancellation, and
  message channels.

`await` and `spawn` always take blocks:

```zzs
let answer := await {
	some_task;
};

let task := spawn {
	"work result";
};
```

The block is expression-valued. The last expression in the block is the
value the block produces.

For `await`, that value must be awaitable:

```zzs
from std/task import resolved;

let answer := await {
	resolved(42);
};
```

For `spawn`, the block is the body of the new task:

```zzs
let task := spawn {
	let name := "Zia";
	`${name} is sleepy`;
};

say await {
	task;
};
```


## 14.2 Async functions return tasks

An async function looks like an ordinary function with `async` in front:

```zzs
from std/task import sleep;

async function answer_later () {
	await {
		sleep(0.01);
	};

	return 42;
}
```

Calling it gives you a `Task`:

```zzs
let task := answer_later();
say typeof task;             // Task
```

Awaiting the task gives you the function's return value:

```zzs
let answer := await {
	task;
};

say answer;                  // 42
```

For script entrypoints that need async work, write `async function
__main__ ( argv )`. The command-line runner passes the argument list and
awaits it after loading the script.

```zzs
from std/task import sleep;

async function __main__ ( argv ) {
	await {
		sleep(0.01);
	};

	say "ready";
}
```


## 14.3 Await points are where other tasks get a turn

ZuzuScript task scheduling is cooperative.

That means a task keeps running until it:

- reaches an `await`,
- returns,
- throws,
- yields explicitly,
- or is cancelled.

Other tasks do not interrupt arbitrary expressions halfway through.

```zzs
from std/task import sleep;

async function greet_later ( name ) {
	say `${name}: start`;

	await {
		sleep(0.01);
	};

	say `${name}: end`;
}

async function __main__ ( argv ) {
	let a := spawn {
		await {
			greet_later("Zia");
		};
	};

	let b := spawn {
		await {
			greet_later("Zenia");
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

The tasks can overlap while they are awaiting `sleep`. A CPU-heavy loop
with no await or yield point still blocks the runtime until it finishes.


## 14.4 `yield()` is a deliberate checkpoint

Import `yield` from `std/task` when a long-running async function should
give the scheduler a chance to run other tasks.

```zzs
from std/task import yield;

async function count_work ( Number limit ) {
	let total := 0;
	let i := 0;

	while ( i < limit ) {
		total += i;

		if ( i mod 100 = 0 ) {
			await {
				yield();
			};
		}

		i += 1;
	}

	return total;
}
```

This is still cooperative concurrency, not true CPU parallelism. Use a
worker when the work should run outside the current runtime.


## 14.5 `spawn { ... }` starts independent work

Use `spawn` when the current task should continue while another task runs.

```zzs
from std/task import sleep;

async function __main__ ( argv ) {
	let background := spawn {
		await {
			sleep(0.05);
		};

		"done";
	};

	say "started";

	let result := await {
		background;
	};

	say result;               // done
}
```

Important rule: if the spawned work matters, keep the task value and
observe it. A spawned task's failure is noticed when you await it or pass
it to a combinator such as `all`, `race`, or `timeout`.

Dropping a task value does not mean "wait for it before continuing".


## 14.6 Task status and cancellation

Tasks are objects. You can inspect and cancel them:

```zzs
from std/task import sleep;

let task := sleep(60);

say task.status();           // sleeping
say task.done();             // false

task.cancel("not needed");

say task.status();           // cancelled
say task.done();             // true
```

Awaiting a cancelled task throws:

```zzs
try {
	await {
		task;
	};
}
catch ( CancelledException e ) {
	say e.to_String();
}
```

Use cancellation when the work no longer has a useful result.


## 14.7 `all` waits for everything

Import `all` from `std/task` when you need several tasks to complete
before moving on.

```zzs
from std/task import all, sleep;

async function label_after ( String label, Number seconds ) {
	await {
		sleep(seconds);
	};

	return label;
}

async function __main__ ( argv ) {
	let results := await {
		all( [
			label_after( "Zia", 0.02 ),
			label_after( "Zenia", 0.01 ),
			label_after( "Zachary", 0.03 ),
		] );
	};

	say results;              // [ "Zia", "Zenia", "Zachary" ]
}
```

`all(tasks)` returns a task that:

- waits for every input task,
- resolves to an Array of results,
- preserves input order,
- fails or cancels if one of the input tasks fails or is cancelled.

The result order is the input order, not completion order.


## 14.8 `race` waits for the first completion

Import `race` when the first completed task is the only result you need.

```zzs
from std/task import race, resolved, sleep;

async function __main__ ( argv ) {
	let winner := await {
		race( [
			sleep(1),
			resolved("ready"),
		] );
	};

	say winner;               // ready
}
```

`race(tasks)` returns a task that:

- resolves or fails with the first completed input task,
- cancels unfinished losing tasks,
- propagates the winning result or error.

Loser cancellation is part of the deal. If later completion matters,
do not put that task in a race.


## 14.9 `timeout` is clearer than a timeout race

Use `timeout(seconds, task)` when one task must finish within a time
limit.

```zzs
from std/task import sleep, timeout;

async function slow_answer () {
	await {
		sleep(10);
	};

	return 42;
}

async function __main__ ( argv ) {
	try {
		let answer := await {
			timeout( 0.25, slow_answer() );
		};

		say answer;
	}
	catch ( TimeoutException e ) {
		say "too slow";
	}
}
```

Use `race` when several alternatives are genuinely competing. Use
`timeout` when time is the constraint.


## 14.10 Coordinated cancellation

For one task, `task.cancel(reason)` is enough.

For several related tasks, use `CancellationSource`.

```zzs
from std/task import CancellationSource, sleep;

async function __main__ ( argv ) {
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

The source owns the cancellation decision. The token is the signal other
code can watch.

Useful token methods are:

- `cancelled()` to check whether cancellation was requested,
- `reason()` to inspect the reason,
- `throw_if_cancelled()` to fail immediately if cancelled,
- `watch(task)` to cancel a task when the token is cancelled.


## 14.11 Channels

`std/task`'s `Channel` is a small FIFO message queue for tasks in the same
runtime.

```zzs
from std/task import Channel;

async function producer ( ch ) {
	await {
		ch.send("Zia");
	};
	await {
		ch.send("Zenia");
	};

	ch.close();
}

async function __main__ ( argv ) {
	let ch := new Channel();

	let task := spawn {
		await {
			producer(ch);
		};
	};

	say await {
		ch.recv();
	};
	say await {
		ch.recv();
	};
	say await {
		ch.recv();
	};                         // null

	await {
		task;
	};
}
```

Channel rules:

- `send(value)` returns a task,
- `recv()` returns a task,
- values are received in FIFO order,
- `close()` closes the channel,
- sending after close throws `ChannelClosedException`,
- receiving after a closed channel is drained resolves to `null`.

Channels are for message passing. They are often clearer than sharing and
mutating the same collection from several tasks.


## 14.12 Resolved and failed tasks

`std/task` also exports helpers for building already-completed tasks:

```zzs
from std/task import failed, resolved;

let ready := resolved("ready");
say await {
	ready;
};

try {
	await {
		failed("not ready");
	};
}
catch ( Exception e ) {
	say e.to_String();
}
```

These are useful in tests, adapters, and code that sometimes has an
immediate answer but needs to return a task-shaped value.


## 14.13 When tasks are not enough

Tasks are cooperative. They are excellent for async workflows and for
work that has natural await points.

They are not a magic way to make CPU-heavy code use multiple cores:

```zzs
let task := spawn {
	let total := 0;
	let i := 0;

	while ( i < 100000000 ) {
		total += i;
		i += 1;
	}

	total;
};
```

That task is independent, but while it is running a tight CPU loop, the
runtime cannot interleave other task work. Add `yield()` checkpoints when
cooperative interleaving is enough. Use workers when the work belongs in
another runtime.


## 14.14 Workers for parallel work

Import `Worker` from `std/worker` when work should run in an isolated
runtime.

```zzs
from std/worker import Worker;

async function __main__ ( argv ) {
	let task := Worker.spawn(
		function ( n ) {
			let total := 0;
			let i := 1;

			while ( i <= n ) {
				total += i;
				i += 1;
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

`Worker.spawn(callable, args?, ...options)` returns a normal `Task`.
Awaiting that task gives the worker's return value. If the worker throws,
cannot receive its input, cannot return its result, or is cancelled,
awaiting the task throws.

Workers are shared-nothing:

- parent and worker do not share mutable ZuzuScript values,
- arguments and return values are copied through `std/marshal`,
- mutations inside the worker do not mutate the parent's original value,
- live runtime resources such as tasks, channels, open files, and host
  handles are not generally transferable.

This copy boundary is a feature. It makes worker code easier to reason
about than shared-memory threads.


## 14.15 Workers are not an untrusted-code sandbox

Workers run in a separate runtime, but they are intended for cooperating
trusted code.

The worker boundary copies values with `std/marshal`, and marshalling may
carry code records that need evaluation. Do not treat a worker as a safe
place to run hostile data or hostile code.

Workers can deny capabilities, and denied capabilities are inherited from
the parent runtime:

```zzs
let task := Worker.spawn(
	function () {
		return __system__{deny_fs};
	},
	[],
	deny_fs: true,
);
```

Supported denial options are:

- `deny_fs`,
- `deny_net`,
- `deny_proc`,
- `deny_db`,
- `deny_clib`,
- `deny_gui`,
- `deny_worker`,
- `deny_js`,
- `deny_perl`.

Passing `false` does not restore a capability that the parent already
denied.


## 14.16 Worker message handles

Use `Worker.spawn_handle` when parent and worker need to exchange more
than one message.

```zzs
from std/worker import Worker;

async function __main__ ( argv ) {
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
	};                         // 42

	say await {
		handle.result();
	};                         // done
}
```

`spawn_handle` returns a `WorkerHandle` in the parent. The worker function
receives an `inbox`.

The parent-side handle provides:

- `send(value)` to send a marshal-copied value to the worker,
- `recv()` to receive the next value from the worker,
- `close()` to close the parent-to-worker send direction,
- `cancel(reason?)` to request cancellation,
- `result()` to get the worker result task,
- `status()` and `done()` to inspect the result task.

The worker-side inbox provides `send`, `recv`, and `close`.

`spawn_handle` is available in implementations that support worker message
passing. Portable code can check:

```zzs
if ( Worker can "spawn_handle" ) {
	// use Worker.spawn_handle(...)
}
```


## 14.17 Choosing the right tool

Use ordinary synchronous code when:

- there is one simple sequence of work,
- waiting is not a problem,
- or clarity is better without task machinery.

Use tasks when:

- work has natural await points,
- several operations can be in progress at once,
- you need timers, channels, cancellation, or task composition.

Use workers when:

- CPU-heavy work should leave the cooperative scheduler,
- parent and worker can communicate by copied values,
- isolation is useful and shared mutable state is not needed.

Avoid using workers just to make code look asynchronous. A worker is a
runtime boundary, not a lightweight function call.


## 14.18 Chapter recap

You now have the concurrency model:

- `async function` returns a `Task`,
- `await { ... }` waits for a task and unwraps its result,
- `spawn { ... }` creates independent task work,
- tasks interleave at await and yield points,
- `all`, `race`, and `timeout` cover common coordination shapes,
- cancellation should have a clear owner,
- channels provide FIFO message passing inside one runtime,
- workers provide shared-nothing parallel work in another runtime.

Next up: IO, files, and directories.
