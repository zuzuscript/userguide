# Chapter 10: Errors, Exceptions, and Regrettable Decisions

In Chapter 9, we learned how to query deeply nested data without getting
lost in traversal boilerplate.

Now we need the companion skill: what to do when reality does not follow
our assumptions.

Files are missing.
Input is weird.
A module is optional.
Zia drank the deployment coffee and now every config key is
`"definitely-fine"`.

This chapter is about writing code that fails *usefully*.

In the Perl implementation, that means getting comfortable with:

- ordinary runtime failures,
- explicit exception flow with `try`, `catch`, `throw`, and `die`,
- typed catches,
- expression-form error handling,
- and practical debugging habits.

The goal is not “never fail”.
The goal is “fail in a way future-you can repair quickly”.


## 10.1 Two kinds of bad news: compile-time vs runtime

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


## 10.2 The `try` / `catch` model in this language

ZuzuScript has explicit exception handling via `try` and `catch`.

Basic shape:

```zzs
try {
	risky_call();
}
catch ( Exception e ) {
	say "something went wrong: " _ e{message};
}
```

How it works:

- Code in `try` runs normally until it finishes or throws.
- If a value is thrown, catch clauses are checked top-to-bottom.
- First matching `catch` runs.
- If nothing matches, the throw continues outward.

This left-to-right catch ordering matters. Put more specific catches
before broader ones.


## 10.3 Throwing values: `throw` and `die`

You have two common ways to signal failure, but they have different
intent in this guide:

### `throw expr` (object/value throw)

Use `throw` when you want to propagate a structured object/value.

```zzs
class Boom {
	let message;
}

throw new Boom( message: "kapow" );
```

### `die "message"` (string shorthand)

Use `die` as a shorthand for a message-style failure.

```zzs
die "config file missing";
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
	note := e{message};
}

say note;   # "retry me"
```

One stylistic guideline:

- use `throw` for structured domain exceptions,
- use `die` for quick message-level fatal branches.

Both are valid; just keep the distinction consistent in a module.


## 10.4 Catch signatures you can use

From language tests in this repository, catch supports these forms.

### Full signature: `catch ( Type name )`

```zzs
try {
	throw new Exception( message: "full" );
}
catch ( Exception e ) {
	say e{message};
}
```

### Name-only shortcut: `catch (name)`

Defaults to `Exception` type.

```zzs
try {
	throw new Exception( message: "name-only" );
}
catch (err) {
	say err{message};
}
```

### Signature-less shortcut: `catch { ... }`

Defaults to `Exception e`.

```zzs
try {
	throw new Exception( message: "default binding" );
}
catch {
	say e{message};
}
```

If you use shortcut catches, keep blocks short and obvious. In larger
code, explicit `catch ( Exception e )` is often easier to scan.


## 10.5 Typed catches and class hierarchies

Chapter 7 introduced inheritance. Error handling can benefit from the
same structure.

You can catch specific classes first, then general classes:

```zzs
class ConfigError {
	let message;
}

class MissingConfig : ConfigError {}

try {
	throw new MissingConfig( message: "config.toml missing" );
}
catch ( MissingConfig e ) {
	say "create file: " _ e{message};
}
catch ( ConfigError e ) {
	say "config issue: " _ e{message};
}
catch ( Exception e ) {
	say "fallback: " _ e{message};
}
```

Order this from narrow to broad.
If you put broad first, specific handlers may never run.

Also useful: `Any` can be used as a broad catch type when you truly want
“catch whatever was thrown here and translate it now”.

Use broad catches sparingly. They are powerful, but can hide design
mistakes if overused.


## 10.6 `try/catch` as an expression (not only a statement)

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
}
;
```

That semicolon after the closing catch block matters in declaration
contexts.

Another compact example:

```zzs
const label := try {
	"ready";
}
catch {
	"fallback";
}
;
```

Expression form is ideal when:

- you need a default value,
- you want one local “attempt + fallback” unit,
- turning the logic into a whole helper function would be overkill.

If it becomes visually dense, split it into a named helper function.


## 10.7 Rethrow vs recover: choose intentionally

Inside a catch block, you generally have three choices:

1. **Recover locally** and continue.
2. **Translate** to a domain-specific value/error.
3. **Rethrow** and let higher-level code decide.

### Recover locally

```zzs
function read_retry_limit ( Dict cfg ) -> Int {
	return try {
		to_Int( cfg{retry_limit} );
	}
	catch ( Exception e ) {
		3;
	}
	;
}
```

### Translate

```zzs
class StartupError {
	let message;
}

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
	log_error( e{message} );
	throw e;
}
```

A good rule: low-level code may translate technical details, high-level
orchestration decides whether to stop the program.


## 10.8 Optional imports and graceful degradation

The Perl implementation supports `try import`, which is excellent for
feature flags and optional dependencies.

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

But remember one parser guardrail:

- wildcard import (`*`) cannot be combined with `try` import.

So this is rejected:

```zzs
# invalid
# from extras/math try import *;
```

Why this matters for error design:

- **required module**: regular `import` and fail fast,
- **optional capability**: `try import` and branch on `null`.

That distinction makes startup behaviour obvious to readers.


## 10.9 Common pitfalls (and how to avoid them)

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
if ( report @? "/team/members/#0" ) {
	process_first_member();
}
else {
	say "No members yet";
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


## 10.10 Debugging workflows that actually help

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
}
;
```

### 5) Prefer deterministic fallback values

If you recover, recover to a known, documented default.


## 10.11 Mini lab: sleepy raccoon release pipeline

Let’s wire several ideas together.

```zzs
class ConfigError {
	let message;
}

class DeployError {
	let message;
}

function read_timeout ( Dict cfg ) -> Int {
	return try {
		to_Int( cfg{timeout_ms} );
	}
	catch ( Exception e ) {
		1500;
	}
	;
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
			message: "unexpected deploy failure: " _ e{message}
		);
	}
	;
}
```

What this does:

- Uses expression-form `try/catch` for value fallback.
- Uses `try import` for optional capability detection.
- Uses typed domain exceptions.
- Keeps broad catch at the end, translating with context.

This is the “cozy but robust” style we want.


## 10.12 Practical checklist for production-ish scripts

Before shipping a script, ask:

- Which failures should stop execution immediately?
- Which failures should degrade with defaults?
- Which module imports are truly optional?
- Are catch clauses ordered from specific to broad?
- Do translated/rethrown errors keep useful context?
- Are we avoiding giant `try` blocks that blur root causes?

A tiny checklist like this saves real debugging hours.


## 10.13 Chapter recap

In this chapter, you learned how error flow works in the Perl
implementation:

- compile-time and runtime failures have different jobs,
- `throw` carries thrown objects/values, while `die` is string shorthand,
- `catch` supports full and shortcut signatures,
- typed catches should be ordered narrow-to-broad,
- `try/catch` works as both statement and expression,
- `try import` enables graceful optional-module behaviour.

Chapter 9 gave you power to ask nested data precise questions.
Chapter 10 gives you the resilience to survive imperfect answers.

Next: Chapter 11, where we organize code into modules and connect your
scripts to the outside world.
