# Chapter 11: Modules, Packages, and the Outside World

In Chapter 10 we practiced failing gracefully.

Now it is time to zoom out.
Most interesting programs are not one giant file.
They are small files with clear boundaries:

- reusable modules,
- imports at the top,
- explicit side-effect points,
- and deliberate interactions with files, processes,
  environment variables, and networks.

This chapter focuses on how that works in the Perl implementation.
Examples are based on current parser/runtime tests and module docs
in this repository.


## 11.1 Why modules matter (especially after chapter 10)

Good module structure makes error handling easier:

- boundaries are clearer,
- failure ownership is clearer,
- and optional features can be isolated.

Zia's rule of thumb:

> If a file does more than one "job story", split it.

A tiny split like this already helps:

- `app/config.zzm` loads and validates config,
- `app/http.zzm` does HTTP calls,
- `main.zzs` wires them together.

When something explodes at 2:17 AM,
you want to know *which* raccoon to blame.


## 11.2 Module files and import syntax

In this implementation, imports use slash-separated module paths.

```zzs
from extras/math import add_x, x;
```

The runtime resolves module identifiers to `.zzm` files in lib paths.
So `extras/math` maps to `extras/math.zzm` under a configured
module root.

### Import forms you should know

Named import:

```zzs
from extras/math import add_x, x;
```

Wildcard import:

```zzs
from extras/math import *;
```

Conditional import:

```zzs
let enabled := true;
from extras/math import add_x if enabled;
from extras/math import add_x unless false;
```

Try import (optional module):

```zzs
from extras/not_real try import MaybeFeature;

if ( MaybeFeature ≡ null ) {
	say "feature unavailable";
}
```

Important parser rule from integration tests:

- `try import` cannot be combined with wildcard `*`.
- postfix `if/unless` also cannot be combined with wildcard `*`.

So these are invalid:

```zzs
# invalid
# from extras/math try import *;
# from extras/math import * if true;
```

### Lexical scope behaviour

Imports are lexical: a symbol imported inside one scope does not
magically leak into outer scope.

That means this pattern is expected to fail outside `foo`:

```zzs
function foo () {
	from extras/math import bar;
	return bar();
}

foo();
# bar();   # not visible here
```


## 11.3 Export behaviour and naming conventions

From tests, wildcard import brings in public exports.
Underscore-prefixed names are treated as private-ish by convention,
but can still be imported explicitly.

```zzs
from extras/private import *, _hidden;
```

Practical style:

- use normal names for intended public API,
- prefix internal helpers with `_`,
- only explicitly import `_private` names when truly needed.

That keeps your module's "front door" obvious.


## 11.4 Search paths and package layout

There are two common ways modules become discoverable:

1. runtime lib paths,
2. script-relative discovery.

Integration tests show both patterns.

### CLI include paths

The CLI supports adding include dirs using `-I`:

```bash
zuzu -I./modules app/main.zzs
```

### Relative app layout

If your script lives in an app directory, module resolution can also
search a script-relative `lib/` tree (for example `app/lib/local`).

A practical project layout:

```text
my-app/
  main.zzs
  lib/
    app/
      config.zzm
      deploy.zzm
    extras/
      formatting.zzm
```

Then in `main.zzs`:

```zzs
from app/config import load_config;
from app/deploy import run_release;
```

### Safety guardrail

Import module paths cannot contain `..` parent segments.
This is rejected by parser/runtime rules for safety and clarity.


## 11.5 Builtin modules: a quick field guide

The Perl implementation exposes a sizable builtin standard library
through module ids under `std/...`.

You do not need to memorize everything.
Start with a few high-value areas:

- **Config**: `std/config`
- **Data**: `std/data/json`, `std/data/xml`, `std/data/yaml`
- **I/O and paths**: `std/io`
- **Process & env**: `std/proc` (includes `Proc`, `Env`)
- **HTTP / networking**: `std/net/http`
- **CLI option parsing**: `std/getopt`
- **Logging**: `std/log`
- **Time**: `std/time`
- **Hashing/encoding**:
  `std/digest/md5`, `std/digest/sha`, `std/string/base64`
- **Dynamic evaluation**: `std/eval`

A tiny starter combo:

```zzs
from std/data/json import JSON;
from std/io import Path;
from std/log import Log;
```

Use module-level imports near the top of the file so dependencies are
visible in one glance.


## 11.6 File I/O with `std/io` and `Path`

For file work, prefer `Path` objects from `std/io`.
Several APIs intentionally expect `Path`, not plain strings.

```zzs
from std/io import Path;

let out := Path.tempfile();
out.spew("sleepy raccoon report\n");
let text := out.slurp();
```

When an API expects `Path`, passing a string can raise a type error.
For example `XML.load(...)` and `XML.dump(...)` are tested with that
contract.

```zzs
from std/data/xml import XML;
from std/io import Path;

let file := Path.tempfile();
let doc := XML.parse("<root><a>1</a></root>");
XML.dump(file, doc);
let loaded := XML.load(file);
```

Design tip:

- use `Path` at module boundaries,
- convert raw user string input to `Path` once,
- then pass typed values through the rest of your code.


## 11.7 Binary snapshots with `std/marshal`

`std/marshal` turns supported live ZuzuScript values into a compact
`BinaryString`, and loads that binary form back into values. It is useful
for trusted persistence, worker handoff, and runtime interoperability.

```zzs
from std/marshal import dump, load, safe_to_dump;
from std/io import Path;

let state := {
    name: "Ada",
    jobs: [ "parse", "compile", "test" ],
};

if ( safe_to_dump(state) ) {
    let blob := dump(state);
    let path := Path.tempfile();
    path.spew(blob);

    let restored := load(path.slurp());
    say restored{name};
}
```

Marshal is not a general text interchange format like JSON. It preserves
more Zuzu runtime structure:

- scalar values: `null`, Booleans, finite Numbers, Strings, and
  BinaryStrings,
- collections: Pairs, Arrays, Dicts, PairLists, Sets, and Bags,
- selected runtime values: Times and Paths,
- user functions, classes, traits, user objects, and bound methods when
  their source, captures, dependencies, and slots are marshalable,
- shared references and supported cycles for non-scalar values.

Objects may participate in the process:

- `__on_dump__` runs before an object is encoded,
- `__on_load__` runs after the graph is reconstructed,
- `__build__` is not called by `load`.

That makes marshal powerful, but it also defines a trust boundary.
`load` is not safe for untrusted blobs in Zuzu Marshal CBOR v1. A blob
can contain code records used to rebuild functions, classes, and traits,
and loaded objects can run `__on_load__`. Treat loading an untrusted
marshal blob like executing untrusted code. For data from outside your
trust boundary, use `std/data/json`, `std/data/cbor`, or another
data-only format and validate the result.

Common dump failures raise `MarshallingException`: unsupported native
functions, regexes, file or database handles, unsupported runtime-backed
objects, non-finite numbers, non-scalar function captures, non-const
code dependencies, and failing `__on_dump__` hooks.

Common load failures raise `UnmarshallingException`: invalid CBOR, a
wrong envelope or version, invalid object/code references, duplicate
Dict keys, duplicate object slot names, malformed code records, failing
`__on_load__` hooks, and malformed or misplaced weak storage records.

Version 1 uses weak storage records to preserve weak object slots and
weak collection entries. A loader must not quietly turn a weak record
into a strong reference, because that would change the graph being
loaded. A dead weak reference, or a weak reference whose target is not
otherwise strongly reachable from the dumped root, loads as a weak
`null` record.


## 11.8 Environment variables and processes (`std/proc`)

When touching the outside world, make side effects explicit and local.

`std/proc` provides useful classes including `Env` and `Proc`.

```zzs
from std/proc import Env, Proc;

function read_mode () -> String {
	return Env.get("APP_MODE", "dev");
}

function current_pid () -> Int {
	return Proc.pid();
}
```

Pattern for clean architecture:

- read env in one place (startup/config module),
- validate and normalize,
- pass plain values downward.

This avoids scattered `Env.get(...)` calls all over your codebase.


## 11.9 Command-line arguments and `__main__`

When run as a script, the CLI calls `__main__(args)` if present.
`args` is an array of positional CLI arguments.

```zzs
function __main__ (args) {
	if ( args[0] ≡ null ) {
		die "usage: app <name>";
	}
	say "hello " _ args[0];
}
```

For richer option parsing, use `std/getopt`.

```zzs
from std/getopt import Getopt;

function __main__ (argv) {
	let parsed := Getopt.parse(
		argv,
		[ "verbose|v", "config|c=s" ]
	);

	if ( not parsed{ok} ) {
		die "option parse failed: " _ parsed{error};
	}

	let opts := parsed{options};
	let rest := parsed{argv};

	if ( opts{verbose} ) {
		say "verbose mode on";
	}

	say "remaining args: " _ rest.size();
}
```

This API is intentionally explicit: you pass argv in,
then receive `options` plus remaining positional args.


## 11.10 Networking and capability-aware imports

Some builtin modules are capability-sensitive in the runtime policy.
From tests, denied capabilities can hide modules from import resolution.

Examples:

- deny `fs` can block `std/io`,
- deny `net` can block `std/net/http`.

This is useful for sandboxing untrusted scripts or enforcing strict
execution profiles.

For HTTP usage, keep networking code in a dedicated module:

```zzs
from std/net/http import UserAgent;

function fetch_status ( String url ) -> Int {
	let ua := new UserAgent();
	let res := ua.get(url);
	return res.status();
}
```

Then higher-level code can choose whether network capability should be
available at all.


## 11.11 Dependency management in practice

ZuzuScript itself does not have an in-language package manager command
in the syntax covered so far.
In the Perl implementation, dependency management is mostly a project
and runtime-path concern.

Practical habits:

1. Keep app modules in a local `lib/` tree.
2. Load that tree in development/CI via `-I` or runtime config.
3. Treat builtin `std/...` modules as platform dependencies.
4. Use `try import` for optional features and feature probes.
5. Fail fast for truly required internal modules.

Optional-feature pattern:

```zzs
from app/metrics try import emit_metric;

function maybe_emit ( String name, Any value ) {
	if ( emit_metric ≡ null ) {
		return null;
	}
	return emit_metric(name, value);
}
```

Required-feature pattern:

```zzs
from app/core/config import load_config;
```

If this import fails, startup should fail loudly.
That is often the right choice.


## 11.12 Managing packages with Zuzuzoo

`zuzuzoo` installs Zuzu distributions into a module directory, a script
directory, and an installed-metadata directory. The command-line tool is
a wrapper around `std/zuzuzoo`, so the same behaviour is available to
ZuzuScript code.

Common commands:

```sh
zuzuzoo install ./example-1.0.0.tar.gz
zuzuzoo install example/module
zuzuzoo remove example/module
zuzuzoo remove --dist example-dist
zuzuzoo list
zuzuzoo query example/module
zuzuzoo query --dist example-dist
zuzuzoo verify example/module
zuzuzoo latest example/module
```

Use `--dry-run` with `install` or `remove` to print the plan without
changing files. Use `--json` with `list`, `query`, `verify`, and
`latest` when another program needs structured output.

During install, Zuzuzoo fetches or reads the source archive, extracts it
to a temporary work directory, validates `zuzu-distribution.json`, runs
`Build.zzs` if present, plans dependencies, optionally runs packaged
tests, writes module and script files, and writes installed metadata
last. Module and script files are copied through temporary sibling
files, hashed after the final move, and then recorded in metadata.
Metadata is also written through a temporary sibling JSON file and read
back before it is moved into place.

The installed metadata lives under the configured metadata directory,
normally `~/.zuzu/meta`. Each distribution has one JSON file named from
its distribution name and version. That metadata records the installed
module and script paths, hashes, sizes for newly installed files, source
information, and the installation root. `verify` checks that recorded
files still exist and that their current hashes, and sizes where
recorded, match the metadata. This catches missing, modified, partial,
and truncated installed files.

Install and remove operations lock the metadata directory for the whole
plan, test, and mutate window. The lock is a directory named
`.zuzuzoo.lock` inside the metadata directory, with `owner.json`
describing the owning process, creation time, metadata directory, and
operation. The default timeout is 30 seconds with a 0.1 second poll
interval. Use `--lock-timeout SECONDS` to fail sooner or wait longer.
Timeout errors include the lock path and readable owner metadata.

Temporary source and extraction directories are cleaned after successful
installs, dry-runs, test failures, corrupt archives, and thrown
exceptions. The API option `keep_work_dirs` keeps them for debugging.
Tests can also set `temp_root` to force Zuzuzoo work directories under a
known parent.

Archive caching is opt-in. Use `--cache-dir DIR`, or pass `cache_dir` to
`std/zuzuzoo`, to cache HTTP and module-name downloads. Local archive
files are never cached. Cache entries use the SHA-256 of the original
URL as `<hash>.archive`, with a `<hash>.json` sidecar containing the
original URL, resolved URL, download time, archive SHA-256, and byte
size. Zuzuzoo validates cached archives before reuse. If the sidecar
hash does not match, the sidecar is corrupt, or the cached archive
cannot be decoded, the cache entry is deleted and downloaded once more.

Failure behaviour is intentionally conservative:

- corrupt downloads and archives report the target, source type, URL or
  file path, and underlying archive error,
- dependency conflicts report the requested dependency, minimum version,
  requester chain, planned or provided version, and conflicting version,
- failed installs do not write metadata claiming files were installed,
- interrupted installs may leave already-written files, but without the
  final installed metadata those files are not treated as installed.

Programmatic use:

```zzs
from std/zuzuzoo import Zuzuzoo;

let zoo := new Zuzuzoo(
	lib_dir: "local/modules",
	bin_dir: "local/bin",
	meta_dir: "local/meta",
);

let result := zoo.install(
	"./example-1.0.0.tar.gz",
	{
		no_test: true,
		cache_dir: ".zuzu-cache",
		lock_timeout: 10,
	},
);
```


## 11.13 Organizing a real script: Zia's sleepy deploy tool

Let's combine module structure, imports, env, file I/O,
and optional integrations.

`main.zzs`:

```zzs
from app/config import load_config;
from app/release import run_release;

function __main__ (args) {
	let cfg := load_config(args);
	let result := run_release(cfg);
	say result;
}
```

`lib/app/config.zzm`:

```zzs
from std/proc import Env;
from std/io import Path;
from std/data/json import JSON;

function load_config (Array args) -> Dict {
	let path_text := Env.get("ZIA_CONFIG", "config.json");
	let path := new Path(path_text);
	let raw := path.slurp();
	let json := new JSON();
	let cfg := json.decode(raw);

	if ( cfg ≡ null ) {
		die "config decode failed";
	}

	return cfg;
}
```

`lib/app/release.zzm`:

```zzs
from app/deployer try import deploy;

function run_release (Dict cfg) -> String {
	if ( deploy ≡ null ) {
		return "deploy integration unavailable";
	}
	return deploy(cfg);
}
```

Notice what this buys us:

- imports tell dependency story quickly,
- optional integration is explicit via `try import`,
- and `main` remains tiny and readable.

Future-you is now less likely to hiss at past-you.


## 11.14 Checklist and pitfalls

Before finishing a module, ask:

- Did I keep imports at top-level and readable?
- Are required vs optional modules clearly distinguished?
- Did I avoid wildcard imports in complex modules?
- Do APIs expecting `Path` receive `Path` objects?
- Is environment/process access centralized?
- Are side effects and external boundaries easy to test?

Common pitfalls:

1. **Using `*` everywhere**
   Nice for spikes, noisy for maintained code.
2. **Hidden optional dependencies**
   If something can be null from `try import`, name that in code.
3. **Stringly-typed file boundaries**
   Convert to `Path` once, then stay typed.
4. **Scattered env reads**
   Pull env at startup, not in random deep helpers.


## 11.14 Wrap-up

Chapter 11 is where scripts become systems.

You now have the core tools to:

- structure code into modules,
- control imports and search paths,
- use builtin standard modules intentionally,
- and integrate with files, env, CLI, process, and network boundaries.

In the next chapter, we'll make those outside-world boundaries
concurrent:

- async functions,
- `await { ... }`,
- `spawn { ... }`,
- task combinators,
- cancellation,
- and awaitable HTTP, process, and file APIs.
