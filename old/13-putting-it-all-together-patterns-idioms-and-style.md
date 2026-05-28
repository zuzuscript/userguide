# Chapter 13: Putting It All Together: Patterns, Idioms, and Style

In Chapter 12, we made scripts concurrent without giving up clear
module, file, process, and network boundaries.

Now we get to the part that makes code feel *pleasant* after month three:
idioms.

This chapter is about writing ZuzuScript in ways that fit the Perl
implementation's parser, runtime behavior, and stdlib conventions,
while still staying friendly for future readers.

Think of this as Zia's long-term maintainability guide:

- what to reach for first,
- what to avoid even when it "works",
- how to shape code so errors are boring,
- and how to keep performance reasonable without turning your file into
  a puzzle box.

Examples in this chapter are aligned with parser/runtime integration
tests and module POD contracts in this repository.


## 13.1 Idiomatic code has three jobs

A good ZuzuScript file usually does three things clearly:

1. **Name intent** with readable functions and values.
2. **Contain side effects** near boundaries.
3. **Prefer data transforms** over hidden mutable state.

A tiny non-idiomatic sketch:

```zzs
from std/proc import Env;
from std/io import Path;

function run () {
	let x := Env.get("CFG", "config.json");
	let y := (new Path(x)).slurp();
	# ... decode, validate, call network, write file, print, all here ...
}
```

An idiomatic refactor:

```zzs
from app/config import load_config;
from app/report import build_report;
from app/output import write_report;

function __main__ (args) {
	let cfg := load_config(args);
	let report := build_report(cfg);
	write_report(cfg, report);
}
```

The second version is not "fancier".
It is just easier to test and easier to panic-debug with coffee.


## 13.2 Collections: array vs set vs bag (the practical decision)

By chapter 8, you know semantics.
Now let's make the decision fast in real code.

Use an **Array** when order matters:

- CLI args,
- event timelines,
- "top N" output.

Use a **Set** when uniqueness is the whole point:

- seen IDs,
- enabled feature flags,
- quick membership checks.

Use a **Bag** when counts matter:

- word frequencies,
- repeated tags,
- vote or reaction tallies.

### Zia's checklist

Ask: "What information must survive this step?"

- If duplicates are meaningful, do **not** choose Set.
- If order is meaningful, do **not** choose Set or Bag as your final
  display structure.
- If you keep hand-counting in a Dict, consider Bag first.

A frequency-style sketch:

```zzs
function tally_snacks (Array snacks) -> Bag {
	let counts := <||>;
	for ( item in snacks ) {
		counts := counts ∪ <| item |>
	}
	return counts;
}
```

Then convert for output only when needed.
That keeps core logic semantically honest.


## 13.3 Query-driven style: pull shape out of nested data

Path operators are one of the biggest readability wins when used
with discipline.

A common anti-pattern is manual deep indexing everywhere:

```zzs
# noisy and brittle
let city := payload{user}{profile}{address}{city};
```

Query-driven style makes intent obvious and composes with filtering.

```zzs
let city := payload @ "/user/profile/address/city";
```

When the shape may vary, make fallback behavior explicit:

```zzs
function user_city (Any payload) -> String {
	let city := payload @ "/user/profile/address/city";
	if ( city ≡ null ) {
		return "unknown-city";
	}
	return city;
}
```

### Idiom: query once, normalize once

Do not repeat the same query in five places.
Query near a boundary, normalize, then pass plain values.

```zzs
function normalize_event (Dict raw) -> Dict {
	let id := raw @ "/event/id";
	let kind := raw @ "/event/kind";
	let ts := raw @ "/event/timestamp";

	return {
		id: id,
		kind: kind,
		timestamp: ts,
	};
}
```

This keeps business logic independent from deep external payload shapes.


## 13.4 Imports and module boundaries: opinionated defaults

From chapter 11 and import tests, we get useful defaults:

- prefer named imports over wildcard `*` in maintained code,
- use `try import` only for genuinely optional features,
- keep imports at top-level for quick dependency scanning,
- use slash-based module ids that map clearly to `.zzm` files.

### Good optional integration pattern

```zzs
from app/metrics try import emit_metric;

function record_latency (String key, Int ms) {
	if ( emit_metric ≡ null ) {
		return null;
	}
	return emit_metric(key, ms);
}
```

### Anti-pattern: pretend optional dependencies are required

```zzs
# risky style
from app/metrics try import emit_metric;

function done () {
	emit_metric("job.done", 1);   # may explode when null
}
```

If a symbol may be `null`, code should read like you know that.


## 13.5 Error style that stays readable

By now you have runtime errors, exceptions, and module boundaries.
The idiom is to keep error ownership local and visible.

### Pattern: fail fast at startup, degrade gracefully at edges

- Required config/module? fail fast.
- Optional integration? degrade with explicit fallback.
- User input issue? return a helpful message, not parser soup.

```zzs
from app/core/config import load_config;
from app/notify try import send_notification;

function boot (Array args) -> Dict {
	# load_config is required; if it fails, startup should stop.
	let cfg := load_config(args);
	return cfg;
}

function maybe_notify (Dict payload) {
	if ( send_notification ≡ null ) {
		return "notify disabled";
	}
	return send_notification(payload);
}
```

### Pattern: keep thrown messages actionable

Prefer messages that say what and where:

- "config decode failed in app/config.zzm"
- "missing required option --project"

Avoid:

- "bad value"
- "oops"
- "error!!!"

Future Zia deserves better breadcrumbs.


## 13.6 Performance, but calm

Most scripts do not need heroic micro-optimization.
They need sane data choices and clear boundaries.

### High-value habits

1. **Choose the right collection first.**
   Wrong structure costs more than tiny expression overhead.
2. **Avoid repeated deep queries in hot loops.**
   Pull fields once outside inner loops when possible.
3. **Avoid repeated decode/encode churn.**
   Decode JSON once, pass structured data onward.
4. **Separate I/O from transforms.**
   Pure transforms are easier to optimize and benchmark later.

### Example: avoid repeated JSON parser setup

```zzs
from std/data/json import JSON;

function decode_many (Array lines) -> Array {
	let json := new JSON();
	let out := [];

	for ( line in lines ) {
		out.push(json.decode(line));
	}

	return out;
}
```

If you instead instantiate `new JSON()` for every line,
you pay overhead and make intent noisier.


## 13.7 Style conventions that age well

Style is not about looking clever.
It is about reducing mental branch mispredictions.

### Naming

- functions: verb-ish (`load_config`, `run_release`),
- values: noun-ish (`cfg`, `report`, `status_code`),
- booleans: question-ish (`is_ready`, `has_token`).

### Function size

If a function cannot fit on a screen without scrolling,
ask whether it has multiple responsibilities.

### Side-effect discipline

A useful split:

- `parse_*`, `normalize_*`, `build_*` are pure-ish,
- `read_*`, `write_*`, `fetch_*`, `run_*` may have side effects.

This naming alone helps readers predict risk.

### Imports

Prefer grouped, stable imports over local surprise imports.
Lexical import scope is real, but top-level imports remain clearer for
most code.


## 13.8 Common anti-patterns (and better replacements)

### Anti-pattern 1: wildcard everywhere

```zzs
from std/io import *;
from std/proc import *;
from std/data/json import *;
```

This hides what you actually depend on.

Better:

```zzs
from std/io import Path;
from std/proc import Env;
from std/data/json import JSON;
```

### Anti-pattern 2: path/string confusion at boundaries

If API expects `Path`, pass `Path`.
Do conversion once near input handling.

```zzs
function config_path_from_env () -> Path {
	let raw := Env.get("APP_CONFIG", "config.json");
	return new Path(raw);
}
```

### Anti-pattern 3: deep env reads in random helpers

Reading env in ten places makes behavior invisible.
Read once in startup/config module and pass concrete values.

### Anti-pattern 4: null-unsafe optional integration

If import used `try`, write code as if feature may be absent.
Because it may.


## 13.9 A full walkthrough: Zia's sleepy issue triager

Let's stitch everything together in one small program.

Goal:

- read issue events from JSON lines,
- normalize fields with path operators,
- count labels with a bag,
- optionally emit metrics,
- write a summary file.

### Project layout

```text
sleepy-triager/
  main.zzs
  lib/
    app/
      input.zzm
      normalize.zzm
      stats.zzm
      metrics.zzm
      output.zzm
```

### `main.zzs`

```zzs
from app/input import load_events;
from app/normalize import normalize_events;
from app/stats import summarize_labels;
from app/metrics import maybe_emit_counts;
from app/output import write_summary;

function __main__ (args) {
	let events := load_events(args);
	let normalized := normalize_events(events);
	let summary := summarize_labels(normalized);
	maybe_emit_counts(summary);
	write_summary(summary);
	say "triage complete";
}
```

### `lib/app/input.zzm`

```zzs
from std/io import Path;
from std/data/json import JSON;
from std/proc import Env;

function load_events (Array args) -> Array {
	let source := args[0];
	if ( source ≡ null ) {
		source := Env.get("ZIA_EVENTS", "events.jsonl");
	}

	let path := new Path(source);
	let text := path.slurp();
	let lines := text.split("\n");
	let json := new JSON();
	let out := [];

	for ( line in lines ) {
		if ( line = "" ) {
			next;
		}
		out.push(json.decode(line));
	}

	return out;
}
```

### `lib/app/normalize.zzm`

```zzs
function normalize_one (Dict e) -> Dict {
	return {
		id: e @ "/id",
		title: e @ "/title",
		state: e @ "/state",
		labels: e @ "/labels",
		assignee: e @ "/assignee/login",
	};
}

function normalize_events (Array events) -> Array {
	let out := [];
	for ( e in events ) {
		out.push(normalize_one(e));
	}
	return out;
}
```

### `lib/app/stats.zzm`

```zzs
function summarize_labels (Array events) -> Dict {
	let label_counts := <||>;
	let open_count := 0;
	let closed_count := 0;

	for ( e in events ) {
		if ( e{state} = "open" ) {
			open_count := open_count + 1;
		}
		else {
			closed_count := closed_count + 1;
		}

		for ( label in e{labels} ) {
			label_counts := label_counts ∪ <| label |>
		}
	}

	return {
		open: open_count,
		closed: closed_count,
		labels: label_counts,
	};
}
```

### `lib/app/metrics.zzm`

```zzs
from app/telemetry try import emit_metric;

function maybe_emit_counts (Dict summary) {
	if ( emit_metric ≡ null ) {
		return null;
	}

	emit_metric("issues.open", summary{open});
	emit_metric("issues.closed", summary{closed});
	return emit_metric("labels.total", summary{labels}.size());
}
```

### `lib/app/output.zzm`

```zzs
from std/io import Path;
from std/data/json import JSON;
from std/proc import Env;

function write_summary (Dict summary) {
	let out_file := Env.get("ZIA_SUMMARY", "summary.json");
	let path := new Path(out_file);
	let json := new JSON(pretty: true, canonical: true);
	path.spew(json.encode(summary));
}
```

### Why this design is idiomatic

- each module has one job,
- imports tell a clear dependency story,
- optional integration is explicit and null-safe,
- path-based extraction is centralized,
- side effects live in input/output/metrics edges,
- core summarization logic is readable and testable.

And most importantly: when Zia finds a bug at 1:04 AM,
she knows where to start.


## 13.10 Weak references and ownership

Use weak references to document non-owning relationships. The rule of
thumb is simple: the object that owns another value should keep a strong
reference; values that only point back to an owner, parent, or observer
should usually be weak.

```zzs
class Node {
	let parent with get, set, clear, has but weak;
	let children := [];

	method add_child (Node child) {
		self.children.push(child);
		child.set_parent(self);
		return self;
	}
}
```

Here, `children` is strong because the parent owns its children.
`parent` is weak because a child should not keep its parent alive.
Generated field setters preserve the weak-storage rule, so callers do
not need to write a separate `clear_parent` method just to break a
reference cycle.

Use weak collection methods for observer-style relationships:

```zzs
class Broadcaster {
	let listeners := [];

	method listen (Function callback) {
		self.listeners.push_weak(callback);
		return self;
	}
}
```

Keep ordinary strong references for retained callbacks, owned children,
caches that are meant to keep values alive, and any value whose lifetime
is part of the current object's responsibility. Do not use `but weak`
as an argument marker; collection APIs expose weak storage explicitly
through method names such as `push_weak`, `set_weak`, and `add_weak`.


## 13.11 Chapter-end checklist

Before you call a script "done", run this checklist:

- **Collections**: did I choose Array/Set/Bag by semantics, not habit?
- **Queries**: did I normalize nested data once instead of poking deep
  structures everywhere?
- **Imports**: are required and optional dependencies clearly separated?
- **Ownership**: are parent links, owner back-references, and observer
  lists weak when they are not lifetime-owning?
- **Boundaries**: are I/O, env, network, and process effects localized?
- **Error messages**: do failures help the next reader recover quickly?
- **Readability**: would a teammate understand this without a tour?

If yes, you are writing ZuzuScript in the spirit of this
Perl implementation: expressive, pragmatic, and maintainable.


## 13.12 Wrap-up and where to go next

You now have the full beginner-to-intermediate path:

- values and types,
- control flow and functions,
- objects and collections,
- query navigation,
- error handling,
- modules and external boundaries,
- concurrency,
- GUI construction for small tools,
- and idioms that keep code healthy over time.

From here, keep Appendix A/B/C nearby for quick syntax and library
lookups, use Appendix F when building GUI tools, and build small scripts
end to end.

One final Zia guideline:

> Prefer clear code you can improve tomorrow over clever code you must
> explain forever.

Next up: use that principle to write your own guide-quality script, or
give it a small GUI from Chapter 14, ship it, and reward yourself with
coffee.
