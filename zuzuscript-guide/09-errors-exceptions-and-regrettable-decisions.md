# Chapter 9: Errors, Exceptions, and Regrettable Decisions

Chapter 8 introduced objects, classes, inheritance, and traits. Those tools
give you useful vocabulary for ordinary data, and they also give you useful
vocabulary for failure.

Now we need the next skill: what to do when reality does not follow our
assumptions.

- Files are missing.
- Input is weird.
- A module is optional.
- A config value says `"true"` where you needed a number.

This chapter is about writing code that fails *usefully*.

In ZuzuScript, that means getting comfortable with:

- ordinary runtime failures,
- explicit exception flow with `try`, `catch`, `throw`, and `die`,
- typed catches,
- expression-form error handling,
- explicit success/failure values with `std/result`,
- and practical debugging habits.

The goal is not “never fail”.
The goal is “fail in a way future-you can repair quickly”.


## 9.1 Two kinds of bad news: compile-time vs runtime

A helpful first split:

1. **Compile-time errors**: the script cannot be parsed/validated enough
   to run.
2. **Runtime errors**: the script starts, then something goes wrong while
   evaluating code.

### Compile-time examples

These fail before your program can do useful work:

- syntax mistakes,
- using undeclared names,
- impossible import forms,
- malformed module paths.

Think “the program text is invalid for execution”.

### Runtime examples

These happen during evaluation:

- calling something that is not callable,
- failing conversions,
- missing data in a strict code path,
- exceptions you throw yourself.

Think “the text compiled, but execution hit a problem”.

A practical habit:

- fix compile errors immediately,
- design runtime failures so they are catchable, local, and clear.


## 9.2 The `try` / `catch` model in this language

ZuzuScript has explicit exception handling via `try` and `catch`.

Basic shape:

```zzs
try {
	risky_call();
}
catch ( Exception e ) {
	say "something went wrong: " _ e.message;
}
```

How it works:

- Code in `try` runs normally until it finishes or throws.
- If a value is thrown, catch clauses are checked top-to-bottom.
- First matching `catch` runs.
- If nothing matches, the throw continues outward.

This left-to-right catch ordering matters. Put more specific catches
before broader ones.


## 9.3 Throwing values: `throw` and `die`

You have two common ways to signal failure.

### `throw expr` (object/value throw)

Use `throw` when you want to propagate a structured object/value.

```zzs
class Boom extends Exception;

throw new Boom( message: "kapow" );
```

### `die "message"` (string shorthand)

Use `die` as a shorthand for a message-style failure.

```zzs
die "config file missing";

// That is a shortcut for:
// throw new Exception( message: "config file missing" );
```

In other words:

- `throw` is for explicit thrown objects/values,
- `die` is the shorthand for string fatal messages.

Both can be handled with matching `catch` clauses:

```zzs
let note := "";

try {
	throw new Exception( message: "retry me" );
}
catch ( Exception e ) {
	note := e.message;
}

say note;   # "retry me"
```

One stylistic guideline:

- use `throw` for structured domain exceptions,
- use `die` for quick message-level fatal branches.

Both are valid; just keep the distinction consistent in a module.


## 9.4 Catch signatures you can use

From language tests in this repository, catch supports these forms.

### Full signature: `catch ( Type name )`

```zzs
try {
	throw new Exception( message: "full" );
}
catch ( Exception e ) {
	say e.message;
}
```

### Name-only shortcut: `catch (name)`

Defaults to `Exception` type.

```zzs
try {
	throw new Exception( message: "name-only" );
}
catch (err) {
	say err.message;
}
```

### Signature-less shortcut: `catch { ... }`

Defaults to `Exception e`.

```zzs
try {
	throw new Exception( message: "default binding" );
}
catch {
	say e.message;
}
```

If you use shortcut catches, keep blocks short and obvious. In larger
code, explicit `catch ( Exception e )` is often easier to scan.


## 9.5 Typed catches and class hierarchies

Chapter 8 introduced inheritance. Error handling can benefit from the
same structure.

You can catch specific classes first, then general classes:

```zzs
class ConfigError extends Exception;
class MissingConfig extends ConfigError;

try {
	throw new MissingConfig( message: "config.toml missing" );
}
catch ( MissingConfig e ) {
	say "create file: " _ e.message;
}
catch ( ConfigError e ) {
	say "config issue: " _ e.message;
}
catch ( Exception e ) {
	say "fallback: " _ e.message;
}
```

Order this from narrow to broad.
If you put broad first, specific handlers may never run.

Also useful: `Any` can be used as a broad catch type when you truly want
“catch whatever was thrown here and translate it now”.

Use broad catches sparingly. They are powerful, but can hide design
mistakes if overused.


## 9.6 `try/catch` as an expression (not only a statement)

This is a great feature for ergonomic code.

`try/catch` can evaluate to a value:

- if `try` succeeds, expression value is from the `try` block,
- if matched catch runs, expression value is from that catch block.

```zzs
const port := try {
		parse_port( env{PORT} );
	}
	catch ( Exception e ) {
		8080;
	};
```

That semicolon after the closing catch block matters in declaration
contexts.

Expression form is ideal when:

- you need a default value,
- you want one local “attempt + fallback” unit,
- turning the logic into a whole helper function would be overkill.

If it becomes visually dense, split it into a named helper function.


## 9.7 Rethrow vs recover: choose intentionally

Inside a catch block, you generally have three choices:

1. **Recover locally** and continue.
2. **Translate** to a domain-specific value/error.
3. **Rethrow** and let higher-level code decide.

### Recover locally

```zzs
function read_retry_limit ( Dict cfg ) → Number {
	return try {
			// Unary plus coerces config value to a Number, but
			// some values cannot be coerced to numbers.
			+cfg{retry_limit};
		}
		catch ( Exception e ) {
			// Default is the `try` block threw an exception.
			3;
		};
}
```

### Translate

```zzs
class StartupError extends Exception;

function load_required_module () {
	try {
		from app/critical import boot;
		return boot;
	}
	catch ( Any e ) {
		throw new StartupError(
			message: "Critical startup module unavailable"
		);
	}
}
```

### Rethrow

```zzs
try {
	sync_once();
}
catch ( Exception e ) {
	log_error( e.message );
	throw e;
}
```

A good rule: low-level code may translate technical details, high-level
orchestration decides whether to stop the program.


## 9.8 Returning structured errors with `std/result`

Exceptions are not the only way to represent failure. The `std/result`
module provides a small `Result` class for functions that should return
either a success value or an error value without throwing.

```zzs
from std/result import Result;

function parse_timeout ( value ) {
	let parsed := try {
			Result.ok( +value );
		}
		catch ( Exception e ) {
			Result.err({
				code: "invalid-timeout",
				reason: e.message,
				input: value,
			});
		};

	if ( parsed.is_err ) {
		return parsed;
	}

	let timeout := parsed.unwrap;

	if ( timeout < 1 ) {
		return Result.err({
			code: "timeout-too-small",
			reason: "timeout must be positive",
			input: value,
		});
	}

	return Result.ok(timeout);
}

let result := parse_timeout("2500");

if ( result.is_ok ) {
	say "timeout: " _ result.unwrap;
}
else {
	let err := result.error;
	say err{code} _ ": " _ err{reason};
}
```

A `Result` has these basic operations:

- `Result.ok(value)` wraps a successful value.
- `Result.err(error)` wraps an error value.
- `result.is_ok` and `result.is_err` tell you which side you have.
- `result.value` returns the ok value, or `null` for an error result.
- `result.error` returns the error value, or `null` for an ok result.
- `result.unwrap` returns the ok value, but throws if the result is an
  error.
- `result.unwrap_err` returns the error value, but throws if the result
  is ok.

Use exceptions when failure should interrupt the current flow: violated
invariants, impossible states, required startup dependencies, or a low-level
operation that cannot sensibly continue. Exceptions keep the happy path
clear, work well with typed catches, and let high-level orchestration decide
where recovery belongs. Their cost is that control flow is less visible at
the call site, broad catches can hide mistakes, and callers must remember
which operations may throw.

Use `Result` when failure is an expected part of the function's contract:
validation, parsing, optional lookups, worker/task responses, or APIs where
the caller should inspect structured error data. Result objects make success
and failure explicit, travel through ordinary return values, and can carry
domain-specific error objects. Their cost is extra checking at every call
site. If callers ignore the result, or call `unwrap()` without checking,
the code becomes noisy or just moves the exception to a later line.

A practical rule:

- throw for exceptional interruption,
- return `Result` for expected, recoverable failure that the caller should
  handle directly.


## 9.9 Optional imports and graceful degradation

ZuzuScript supports `try import`, which is excellent for feature flags
and optional dependencies.

```zzs
from extras/not_real try import MaybeFeature;

if ( MaybeFeature ≡ null ) {
	say "Optional feature unavailable; continuing.";
}
```

This lets you represent “not found” as `null` binding instead of a hard
compile-stop for that specific import request.

You can combine with postfix conditions:

```zzs
let enabled := true;
from extras/not_real try import Maybe if enabled;
```

But remember one guardrail:

- wildcard import (`*`) cannot be combined with `try` import.

So this is rejected:

```zzs
// invalid
from extras/math try import *;
```

Why this matters for error design:

- **required module**: regular `import` and fail fast,
- **optional capability**: `try import` and branch on `null`.

That distinction makes startup behaviour obvious to readers.


## 9.10 Common pitfalls (and how to avoid them)

### Pitfall 1: Catching too broadly too early

```zzs
# less ideal
try { risky(); }
catch ( Any e ) { say "oops"; }
catch ( Exception e ) { ... }
```

The second catch is unreachable by design. Keep broad catch last.

### Pitfall 2: Swallowing errors silently

```zzs
# risky style
try { write_config(); }
catch ( Exception e ) { }
```

Always do *something* explicit:

- log,
- increment a failure counter,
- convert to safe fallback,
- or rethrow.

### Pitfall 3: Using exceptions for normal control flow

If a condition is expected and frequent, prefer normal branching.

```zzs
if ( items.length() > 0 ) {
	process_first_item( items.get(0) );
}
else {
	say "No items yet";
}
```

Reserve exceptions for truly exceptional or boundary-failure states.

### Pitfall 4: Losing context on rethrow/translate

When translating, preserve useful context in message fields.

```zzs
throw new StartupError(
	message: "config load failed in profile=night"
);
```

The future debug session will thank you.


## 9.11 `debug`, `assert`, and always-on warnings

ZuzuScript has two debugging keywords that are controlled by the runtime
debug level: `debug` and `assert`.

`debug level, expr` writes a diagnostic line to standard error when
`level` is less than or equal to the current runtime debug level:

```zzs
debug 1, "loaded config";
debug 2, "config detail: " _ config{name};
```

At the default debug level `0`, those two lines do nothing. At `-d1`, the
first line prints. At `-d2`, both lines print.

```sh
zuzu -d1 script.zzs
zuzu -d2 script.zzs
```

The message expression is not evaluated when the line is filtered out.
That makes `debug` useful for temporary trace points that would otherwise
be expensive or noisy:

```zzs
debug 2, "full payload: " _ build_large_diagnostic(payload);
```

Use `debug` for developer-facing trace output. Do not use it for messages
that users need to see during normal execution.

`assert expr` checks an internal assumption only when debugging is enabled:

```zzs
function average ( Array values ) {
	assert values.length() > 0;
	return values.sum() / values.length();
}
```

At debug level `0`, the assertion expression is not evaluated. With
debugging enabled, a false assertion throws `AssertionException`. That
makes `assert` good for invariants that should never be false if the
program is correct.

Do not use `assert` for ordinary input validation:

```zzs
// Good: user-facing validation.
if ( values.length() = 0 ) {
	die "cannot average an empty list";
}

// Good: internal sanity check after validation.
assert values.length() > 0;
```

Use `warn` when the diagnostic should always be emitted. `warn` writes to
standard error and adds a newline; it is not gated by the debug level.
That makes it suitable for deprecations, fallback notices, and operational
messages that should not disappear in normal mode.


## 9.12 Debugging workflows that actually help

When something breaks, calm, repeatable habits beat heroics.

### 1) Reproduce with a tiny input

Minimize script state until the failure is stable and quick to rerun.

### 2) Keep failure messages specific

Prefer messages that include identifiers, path fragments, and stage
names, not just “failed”.

```zzs
die "user import failed at row " _ row_index;
```

### 3) Guard unsafe assumptions early

Use fast checks before deep logic:

```zzs
if ( data ≡ null ) {
	die "data cannot be null";
}
```

### 4) Isolate risky calls in narrow `try` blocks

Smaller `try` scopes make root cause clearer.

```zzs
let parsed := try {
		parse_json(raw);
	}
	catch ( Exception e ) {
		die "invalid JSON payload";
	};
```

### 5) Prefer deterministic fallback values

If you recover, recover to a known, documented default.


## 9.13 Mini lab: release pipeline

Let’s wire several ideas together.

```zzs
class ConfigError extends Exception;
class DeployError extends Exception;

function read_timeout ( Dict cfg ) -> Number {
	return try {
			+cfg{timeout_ms};
		}
		catch ( Exception e ) {
			1500;
		};
}

function load_deployer () {
	from tools/deploy try import run_deploy;
	if ( run_deploy ≡ null ) {
		throw new DeployError(
			message: "deploy module missing"
		);
	}
	return run_deploy;
}

function run_release ( Dict cfg ) -> String {
	if ( cfg ≡ null ) {
		throw new ConfigError( message: "cfg is null" );
	}

	let timeout := read_timeout(cfg);
	let deploy_fn := load_deployer();

	return try {
			deploy_fn( timeout );
			"ok";
		}
		catch ( DeployError e ) {
			"degraded";
		}
		catch ( Exception e ) {
			throw new DeployError(
				message: "unexpected deploy failure: " _ e.message
			);
		};
}
```

What this does:

- Uses expression-form `try/catch` for value fallback.
- Uses `try import` for optional capability detection.
- Uses typed domain exceptions.
- Keeps broad catch at the end, translating with context.

This is the robust style we want.


## 9.14 Practical checklist for production-ish scripts

Before shipping a script, ask:

- Which failures should stop execution immediately?
- Which failures should degrade with defaults?
- Which module imports are truly optional?
- Are catch clauses ordered from specific to broad?
- Do translated/rethrown errors keep useful context?
- Should this API throw, or return a `Result` the caller can inspect?
- Are we avoiding giant `try` blocks that blur root causes?

A tiny checklist like this saves real debugging hours.


## 9.15 Chapter recap

In this chapter, you learned how error flow works in ZuzuScript:

- compile-time and runtime failures have different jobs,
- `throw` carries thrown objects/values, while `die` is string shorthand,
- `catch` supports full and shortcut signatures,
- typed catches should be ordered narrow-to-broad,
- `try/catch` works as both statement and expression,
- `std/result` lets APIs return explicit success or failure values,
- `try import` enables graceful optional-module behaviour.
- `debug`, `assert`, and `warn` give you different levels of diagnostic
  output and invariant checking.

This chapter gives you the resilience to survive imperfect inputs, missing
dependencies, and decisions that did not age well.

Next we will use that resilience while splitting code into modules. Good
module boundaries make failure easier to localize, and optional imports give
you one more way to degrade gracefully.
