# Chapter 16: Processes, Environment, and System State

Chapter 15 covered files and structured data.

This chapter covers the other common way a script talks to the outside
world: processes.

ZuzuScript's `std/proc` module provides:

- `Env` for environment variables,
- `Proc.pid()` for the current process id,
- `Proc.run(...)` for one external command,
- `Proc.pipeline(...)` for command pipelines,
- `Proc.run_async(...)` and `Proc.pipeline_async(...)`,
- `Proc.kill(...)` and `Proc.onsignal(...)`,
- `Proc.exit(...)`,
- `sleep(...)` and `sleep_async(...)`,
- and helpers for interpreting process results.

This chapter also introduces the global `__system__` dictionary, which
describes the current runtime and its available capabilities.

`std/proc` is supported in the command-line runtimes and in host
JavaScript runtimes with process support, such as Node and Electron. It is
not available in a browser page.


## 16.1 Importing `std/proc`

The usual imports are:

```zzs
from std/proc import Proc, Env;
```

Import `sleep` or `sleep_async` only when you need them:

```zzs
from std/proc import sleep, sleep_async;
```

`Proc` and `Env` are classes with static methods. You do not create
instances of them:

```zzs
say Proc.pid();
say Env.get("HOME", "");
```


## 16.2 The process boundary

Running a process is different from calling a function.

An external command has:

- a program name,
- an argument list,
- standard input,
- standard output,
- standard error,
- an environment,
- a working directory,
- and an exit status.

`std/proc` keeps those pieces explicit. It does not ask a shell to parse
one large string for you.

This is deliberate:

```zzs
Proc.run("echo", [ "Zia is sleepy" ]);
```

The command is `echo`. The argument is one string. There is no shell
interpolation, no wildcard expansion, and no pipe character with special
meaning. If you want a pipeline, use `Proc.pipeline(...)`.


## 16.3 Environment variables with `Env`

Environment variables are inherited from the process that started your
script. They are often used for configuration:

```zzs
from std/proc import Env;

let mode := Env.get("APP_MODE", "development");
let token := Env.get("API_TOKEN", null);
```

`Env.get(name, default?)` returns the variable value when it is set. If it
is not set, it returns the fallback value. Without a fallback, the result
is `null`:

```zzs
let name := Env.get("ZIA_NAME", "Zia");
let missing := Env.get("NO_SUCH_VARIABLE");
```

Use `Env.set(name, value)` to set a variable in the current process:

```zzs
Env.set("APP_MODE", "test");
say Env.get("APP_MODE");      // test
```

Use `Env.remove(name)` to remove it:

```zzs
Env.remove("APP_MODE");
```

Changes made with `Env.set` and `Env.remove` affect the current process
and child processes started after the change. They do not rewrite the
environment of the parent shell that launched your script.

For one child process, prefer the `env` option to `Proc.run`. That keeps
the change local:

```zzs
let result := Proc.run(
	"perl",
	[ "-e", "print $ENV{FRIEND};" ],
	{
		env: {
			FRIEND: "Zenia",
		},
	},
);

say result{stdout};           // Zenia
```

The `env` option can also set a variable to `null` to remove it for that
child process.


## 16.4 Current process id

`Proc.pid()` returns the current process id:

```zzs
from std/proc import Proc;

say Proc.pid();
```

The value is mainly useful for diagnostics and signals:

```zzs
say `running as pid ${Proc.pid()}`;
```


## 16.5 Running one command

Use `Proc.run(command, argv?, options?)` to run one external command and
wait for it to finish:

```zzs
from std/proc import Proc;

let result := Proc.run(
	"perl",
	[ "-e", "print qq<hello from a child process\\n>;" ],
);

say result{stdout};
```

The result is a dictionary. The important fields are:

- `ok`: true when the command succeeded,
- `stdout`: captured standard output,
- `stderr`: captured standard error,
- `exit_code`: the command's numeric exit code,
- `signal`: the signal number when the process ended by signal,
- `core_dump`: whether the process produced a core dump,
- `error`: an execution error, or `null`,
- `timed_out`: true when a timeout stopped the command,
- `command`: the command array that was run.

Always check `ok` before trusting the output:

```zzs
let result := Proc.run("zuzu-lint", [ "notes.zzs" ]);

if ( result{ok} ) {
	say "lint passed";
}
else {
	say `lint failed: ${Proc.status_text(result)}`;
	say result{stderr};
}
```

`Proc.is_success(result)` performs the same success check:

```zzs
if ( Proc.is_success(result) ) {
	say "success";
}
```

`Proc.status_text(result)` formats the status for humans:

```zzs
say Proc.status_text(result);     // exit 0, exit 2, signal 15, ...
```


## 16.6 Passing input and capturing output

Use the `stdin` option to pass text to the child process:

```zzs
let result := Proc.run(
	"perl",
	[ "-e", "my $line = <STDIN>; print uc($line);" ],
	{
		stdin: "zia\n",
	},
);

say result{stdout};           // ZIA
```

Output is captured by default:

```zzs
let result := Proc.run("perl", [ "-e", "print qq<ok\\n>;" ]);
say result{stdout};           // ok
```

The main output options are:

- `capture_stdout`: defaults to true,
- `capture_stderr`: defaults to true,
- `merge_stderr`: defaults to false.

When `merge_stderr` is true, stderr is combined into stdout:

```zzs
let result := Proc.run(
	"perl",
	[ "-e", "print qq<out\\n>; print STDERR qq<err\\n>;" ],
	{
		merge_stderr: true,
	},
);

say result{stdout};
```

Use this when the order of diagnostic and normal output matters more than
keeping the streams separate.


## 16.7 Working directory and timeouts

The `cwd` option runs the child process from a different directory:

```zzs
from std/io import Path;
from std/proc import Proc;

let dir := new Path("examples");

let result := Proc.run(
	"perl",
	[ "-e", "print qq<running here\\n>;" ],
	{
		cwd: dir.to_String(),
	},
);
```

The parent process's working directory is restored after the command.

Use `timeout` to stop a command that takes too long:

```zzs
let result := Proc.run(
	"perl",
	[ "-e", "sleep 10; print qq<late\\n>;" ],
	{
		timeout: 0.5,
	},
);

if ( result{timed_out} ) {
	say "command timed out";
}
```

Timeouts are measured in seconds, and fractional seconds are allowed.


## 16.8 Command arrays

`Proc.run` has two equivalent forms.

The first form separates the command and arguments:

```zzs
Proc.run("perl", [ "-e", "print qq<ok\\n>;" ]);
```

The second form passes the whole command as one array:

```zzs
Proc.run( [ "perl", "-e", "print qq<ok\\n>;" ] );
```

The second form is useful when you already have a command specification in
an array:

```zzs
let command := [ "perl", "-e", "print qq<Zachary\\n>;" ];
let result := Proc.run(command);
```

Do not join a command into one string unless the program name really is
that whole string. `Proc.run("git status")` looks for a program literally
called `git status`; it does not run `git` with the argument `status`.


## 16.9 Pipelines

Use `Proc.pipeline(commands, options?)` when the stdout of one command
should become the stdin of the next:

```zzs
from std/proc import Proc;

let result := Proc.pipeline(
	[
		[ "perl", "-e", "print qq<zia\\nzenia\\nzachary\\n>;" ],
		[ "perl", "-ne", "print uc($_);" ],
	],
);

say result{stdout};
```

The `commands` argument is an array of command specs. Each command spec is
itself an array:

```zzs
[
	[ "command-one", "arg" ],
	[ "command-two", "arg" ],
]
```

The pipeline result has the same top-level fields as `Proc.run`, plus
`steps`:

```zzs
for ( const step in result{steps} ) {
	say Proc.status_text(step);
}
```

`result{stdout}` is the final command's stdout. Each item in
`result{steps}` is the result dictionary for one command.

If an earlier step fails, the pipeline result is not `ok`.

Pipeline options are the same kind of options used by `Proc.run`:

```zzs
let result := Proc.pipeline(
	[
		[ "perl", "-pe", "s/Zia/Zenia/g" ],
		[ "perl", "-ne", "print if /Zenia/" ],
	],
	{
		stdin: "Zia is sleepy\nFenn is alert\n",
		timeout: 1,
	},
);
```

This is the process equivalent of the chain operators from Chapter 13:
data flows from one step to the next. The difference is that each step is
an external program rather than a ZuzuScript expression.


## 16.10 Async process execution

`Proc.run` and `Proc.pipeline` block until the child process is finished.
That is fine in a short synchronous script.

In async code, prefer the awaitable versions:

```zzs
from std/proc import Proc;

async function run_tool () {
	let result := await {
		Proc.run_async(
			"perl",
			[ "-e", "print qq<async ok\\n>;" ],
		);
	};

	return result{stdout};
}
```

`Proc.run_async(command, argv?, options?)` resolves to the same kind of
result dictionary as `Proc.run`.

`Proc.pipeline_async(commands, options?)` resolves to the same kind of
result dictionary as `Proc.pipeline`:

```zzs
async function uppercase_names () {
	let result := await {
		Proc.pipeline_async(
			[
				[ "perl", "-e", "print qq<zia\\nzenia\\n>;" ],
				[ "perl", "-ne", "print uc($_);" ],
			],
		);
	};

	return result{stdout};
}
```

Async process helpers compose with the `std/task` helpers from Chapter 14:

```zzs
from std/proc import Proc;
from std/task import all;

async function run_two () {
	let results := await {
		all( [
			Proc.run_async( "perl", [ "-e", "print qq<Zia>" ] ),
			Proc.run_async( "perl", [ "-e", "print qq<Zenia>" ] ),
		] );
	};

	return results[0]{stdout} _ " and " _ results[1]{stdout};
}
```

The async forms still run operating-system processes. They simply let the
ZuzuScript scheduler make progress while the process is running.


## 16.11 Sleeping

`std/proc` exports a synchronous `sleep(seconds)`:

```zzs
from std/proc import sleep;

say "before";
sleep(0.5);
say "after";
```

This blocks the current process. It is useful in simple synchronous
scripts, but avoid it inside async workflows.

For async code, use `sleep_async(seconds)` from `std/proc`:

```zzs
from std/proc import sleep_async;

async function pause_then_return () {
	await {
		sleep_async(0.5);
	};

	return "done";
}
```

`sleep_async` returns a `Task`, so it can be combined with `all`, `race`,
or `timeout` from `std/task`.

Chapter 14 also introduced `sleep` from `std/task`. Both are awaitable
ways to pause in async code. The important distinction in this chapter is:

- `std/proc.sleep(seconds)` blocks,
- `std/proc.sleep_async(seconds)` is awaitable.


## 16.12 Signals

Signals are operating-system notifications sent to a process. Common
signals include `INT` and `TERM`. Some systems also support signals such
as `USR1` and `USR2`.

Use `Proc.onsignal(signal, callback)` to register a callback:

```zzs
from std/proc import Proc;

let shutting_down := false;

Proc.onsignal(
	"TERM",
	function () {
		shutting_down := true;
	},
);
```

The method is spelled `onsignal` in the API. Read it as "on signal".
If you think of the operation as `on_signal`, remember that the exported
method name does not contain an underscore.

Use `Proc.kill(signal, pid?)` to send a signal:

```zzs
Proc.kill("TERM", some_pid);
```

If `pid` is omitted, the current process is used:

```zzs
Proc.onsignal(
	"USR1",
	function () {
		say "received USR1";
	},
);

Proc.kill("USR1");
```

`Proc.kill` returns the number of signalled processes reported by the
runtime.

Signal support depends on the operating system and host runtime. Keep
signal callbacks short. Set a flag, write a small message, or trigger
cleanup; do not put large application workflows inside the handler.


## 16.13 Exiting the current process

Use `Proc.exit(code?)` to terminate the current process:

```zzs
from std/proc import Proc;

if ( not ok ) {
	Proc.exit(1);
}

Proc.exit(0);
```

An exit code of `0` conventionally means success. Non-zero values normally
mean failure.

`Proc.exit` does not return. Code after it will not run:

```zzs
Proc.exit(2);
say "this will not be printed";
```

Because it ends the current process immediately, reserve `Proc.exit` for
script entrypoints and command-line tools. Library code should usually
throw an exception or return an error value instead.


## 16.14 The `__system__` global

`__system__` is a read-only global dictionary describing the current
runtime:

```zzs
say __system__{runtime};
say __system__{platform};
```

Common keys include:

- `language_version`,
- `runtime`,
- `runtime_version`,
- `platform`,
- `inc`,
- `deny_fs`,
- `deny_net`,
- `deny_perl`,
- `deny_js`,
- `deny_proc`,
- `deny_db`,
- `deny_clib`,
- `deny_gui`,
- `deny_worker`.

Some runtimes add extra keys. For example, the JavaScript Node host may
include `nodejs_version`, and the Perl runtime includes `perl_version`.

Use `in` or `.get(...)` when reading optional keys:

```zzs
if ( "nodejs_version" in __system__ ) {
	say __system__{nodejs_version};
}

let platform := __system__.get("platform", "unknown");
```

`inc` is the module search path, as introduced in Chapter 10:

```zzs
for ( const dir in __system__{inc} ) {
	say dir;
}
```

The `deny_*` keys describe runtime policy and host capabilities. For
process support, check `deny_proc`:

```zzs
if ( __system__{deny_proc} ) {
	die "process execution is not available here";
}
```

When a capability is denied, importing or using the module that needs it
may fail. This is why browser examples avoid `std/proc`: a normal browser
host does not have process execution.

`__system__` is read-only:

```zzs
__system__{deny_proc} := false;     // invalid
```

It is information, not permission. A script can inspect it, but cannot
grant itself or its child processes new capabilities by changing it.


## 16.15 A complete small example

This script runs a formatter command, reports errors clearly, and uses an
environment variable for configuration.

```zzs
from std/proc import Env, Proc;

function formatter_command () {
	let command := Env.get("ZIA_FORMATTER", "perl");

	if ( command eq "perl" ) {
		return [
			"perl",
			"-pe",
			"s/\\bzia\\b/Zia/g; s/\\bzenia\\b/Zenia/g;",
		];
	}

	return [ command ];
}

function format_text ( String text ) -> String {
	let result := Proc.run(
		formatter_command(),
		[],
		{
			stdin: text,
			timeout: 2,
			capture_stderr: true,
		},
	);

	if ( not result{ok} ) {
		die "formatter failed: "
			_ Proc.status_text(result)
			_ "\n"
			_ result{stderr};
	}

	return result{stdout};
}

say format_text("zia and zenia are planning a careful walk\n");
```

The important pattern is:

- keep command arguments as arrays,
- pass input through `stdin`,
- use `env` for per-child configuration,
- check `ok`,
- use `Proc.status_text` in diagnostics,
- and use the async forms when the surrounding workflow is async.

At this point we have files, structured data, environment, and external
commands. Chapter 17 pulls those pieces together into a full command-line
tool with options, config, validation, and exit codes.
