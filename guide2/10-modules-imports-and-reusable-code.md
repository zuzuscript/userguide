# Chapter 10: Modules, Imports, and Reusable Code

Chapter 9 was about handling failure well.

Now we move from individual scripts to small systems.

Most useful programs eventually need the same move:

- split reusable code into modules,
- import only the names a script actually uses,
- put module roots somewhere predictable,
- and document public APIs near the code that provides them.

In ZuzuScript, module files usually use the `.zzm` extension. Scripts
usually use `.zzs`. The language does not make those formats radically
different, but the convention matters:

- `.zzs` means "run this as a program",
- `.zzm` means "import this as reusable code".

This chapter covers:

- importing named exports,
- wildcard imports and aliases,
- optional and conditional imports,
- how module search paths work,
- the `-I` command-line option,
- writing modules,
- what gets exported by default,
- and documenting modules with POD.


## 10.1 Importing from modules

The most common import form is:

```zzs
from std/string import trim, split, join;
```

Read it as:

> From the module `std/string`, bring these names into the current scope.

Once imported, those names behave like ordinary names:

```zzs
from std/string import trim, split, join;

let raw := "  red, green, blue  ";
let parts := split( trim(raw), "," );

say join( " / ", parts );
```

So this import:

```zzs
from app/config import load_config;
```

looks for a module file named `app/config.zzm` (with a fallback to
`app/config.zzs`) under one of the configured module search roots.


## 10.2 Importing several names

Use commas to import several things from the same module:

```zzs
from std/string import trim, starts_with, ends_with;
```

That keeps dependency lists readable and makes missing names fail early.

If the module does not export one of the names you asked for, the import
fails instead of quietly creating a useless value.

This is usually what you want:

```zzs
from app/report import build_report, render_report;

let report := build_report();
say render_report(report);
```

Prefer named imports for application code. They make it obvious which
parts of another module this file depends on.


## 10.3 Import aliases with `as`

Use `as` when two modules export the same name, or when a short local
name would make code clearer.

```zzs
from std/string import index as string_index;

say string_index( "abcdef", "cd" );
```

Aliases are especially useful around classes:

```zzs
from app/html/report import Renderer as HtmlRenderer;
from app/text/report import Renderer as TextRenderer;

let html := new HtmlRenderer();
let text := new TextRenderer();
```

The imported export is the same value; only the local name changes.

Previously it was mentioned that `instanceof` is more reliable than
`typeof`. The `typeof` operator would report both of the above
objects as having type "Renderer"; `instanceof` allows you to check
which class(es?) an object belongs to.


## 10.4 Wildcard imports

You can import every public export with `*`:

```zzs
from std/string import *;
```

That is convenient for small scripts, tests, and facade modules.

For example, tests often do this:

```zzs
from test/more import *;

ok( true, "the test framework is loaded" );
done_testing();
```

For larger application files, use wildcard imports carefully. They hide
which names came from which module, and they make accidental name
collisions harder to see.

A good practical rule:

- use named imports in normal application code,
- use wildcard imports in tests and small throwaway scripts,
- use wildcard imports in facade modules when you intentionally re-export
  another module's API.

Wildcard import skips names that begin with `_`.

```zzs
from app/config import *;
```

That imports `load_config`, `ConfigError`, and other public-looking names,
but it does not import `_parse_line`.

If you really need an underscore-prefixed export, import it explicitly:

```zzs
from app/config import _parse_line;
```

Do that rarely. Leading `_` is the module author's way of saying
"internal helper".


## 10.5 Import scope

Imports are lexical.

An import at the top of a file is available to the rest of that file.
An import inside a function or block is local to that function or block.

```zzs
function normalize_name (name) {
	from std/string import trim;

	return trim(name);
}

say normalize_name("  Ada  ");
// trim("  Ada  ");  // Error: trim is not in this outer scope.
```

Local imports are useful when:

- an optional feature is only used in one branch,
- importing at the top would create a name collision,
- or keeping a dependency near its only use makes the code easier to read.

For ordinary modules, top-level imports are still the default style.
They make dependencies easy to scan.


## 10.6 Optional imports with `try import`

Sometimes a module is optional.

Maybe it depends on a host feature. Maybe it is provided by a third-party
package. Maybe your script has a useful fallback.

Use `try import` for that:

```zzs
from std/worker try import Worker;

if ( Worker ≡ null ) {
	say "Workers are not available here.";
}
else {
	say "Workers are available.";
}
```

If the import succeeds, `Worker` is bound to the export.
If it fails, `Worker` is bound to `null`.

That means optional imports are good for feature detection:

```zzs
from app/fast_json try import FastJSON;
from std/data/json import JSON;

let CodecClass := FastJSON ?: JSON;
```

Important restriction: `try import` works with named imports, not wildcard
imports.

This is valid:

```zzs
from app/feature try import Feature;
```

This is not:

```zzs
// Invalid.
from app/feature try import *;
```

If a feature is optional, name the pieces you expect from it.


## 10.7 Conditional imports with `if` and `unless`

Imports may also have a postfix condition:

```zzs
from app/debug import DebugTools if debug_mode;
from app/prod import ProdTools unless debug_mode;
```

If the condition says the import should not run, the requested names are
bound to `null`.

```zzs
let use_colour := false;

from app/colour import ColourPrinter if use_colour;

if ( ColourPrinter ≡ null ) {
	say "plain output";
}
else {
	ColourPrinter.say("colour output");
}
```

Conditional imports can combine with `try import`:

```zzs
from app/native try import NativeSpeedup if use_native;
```

That means:

1. if `use_native` is false, bind `NativeSpeedup` to `null`,
2. otherwise try the import,
3. if the import fails, also bind `NativeSpeedup` to `null`.

Like `try import`, conditional imports are for named imports, not wildcard
imports.


## 10.8 Module search paths

When ZuzuScript sees this:

```zzs
from app/config import load_config;
```

it does not search the whole filesystem.

It checks configured module roots in order. Under each root, it looks for:
`app/config.zzm` or `app/config.zzs`.

The search order is:

1. directories passed with `-I`, in command-line order,
2. entries from `ZUZULIB`,
3. the user third-party module directory, if it exists,
4. the system third-party module directory, if it exists,
5. the standard-library module directory.

`ZUZULIB` is a path list. On Unix-like systems, entries are separated with
`:`. On Windows, entries are separated with `;`.

`ZUZU_STDLIB` is different: it is a single directory that replaces the
installed standard-library module directory.

Relative entries in `-I`, `ZUZULIB`, and `ZUZU_STDLIB` are resolved
against the interpreter's initial current working directory. They do not
change meaning if your program later changes directory.

You can inspect the effective include path with:

```bash
zuzu -V
```

At runtime, the final path is also available as:

```zzs
__system__{inc}
```

That is mainly useful when debugging module loading.


## 10.9 The `-I` command-line option

For your own projects, `-I` is the tool you will use most often.

Suppose your project looks like this:

```text
my-app/
  main.zzs
  modules/
    app/
      config.zzm
      report.zzm
```

Run it like this:

```bash
zuzu -Imy-app/modules my-app/main.zzs
```

or, if your runtime accepts the separated form:

```bash
zuzu -I my-app/modules my-app/main.zzs
```

Then `main.zzs` can import modules under `modules/`:

```zzs
from app/config import load_config;
from app/report import build_report;

let config := load_config();
say build_report(config);
```

The module root is `my-app/modules`.
The module path is `app/config`.
The file is `my-app/modules/app/config.zzm`.

Keep that distinction clear:

- `-I` points at a module root,
- `from ... import ...` names a module under that root,
- `.zzm` is added by the module loader.

You can pass more than one `-I`:

```bash
zuzu -Imodules -Ivendor/modules main.zzs
```

Earlier entries win when two roots contain the same module path.


## 10.10 Module paths are not filesystem paths

Module paths deliberately have a restricted shape.

They are made of identifiers separated by `/`:

```zzs
from app/config import load_config;
from std/data/json import JSON;
```

Do not write relative filesystem paths in imports:

```zzs
// Invalid.
from ./app/config import load_config;

// Invalid.
from ../shared/config import load_config;
```

Parent-directory segments such as `..` are rejected.

That keeps imports portable and makes module lookup depend on configured
module roots instead of whatever directory the process happens to be in.


## 10.11 Writing a module

A module is just a ZuzuScript file intended to be imported.

Create `modules/app/greeting.zzm`:

```zzs
function greeting (name) {
	return "Hello, " _ name _ ".";
}

class Greeter {
	let prefix := "Hello";

	method greet (name) {
		return prefix _ ", " _ name _ ".";
	}
}
```

Then import it from a script:

```zzs
from app/greeting import greeting, Greeter;

say greeting("Ada");

let greeter := new Greeter( prefix: "Good morning" );
say greeter.greet("Lin");
```

Run it with the module root:

```bash
zuzu -Imodules main.zzs
```

Modules may import other modules too:

```zzs
from std/string import trim;

function clean_greeting (name) {
	return "Hello, " _ trim(name) _ ".";
}
```

Keep module-level side effects small.

This is good module behaviour:

- define functions,
- define classes and traits,
- define constants,
- import dependencies,
- prepare cheap reusable data.

Be cautious with this at module load time:

- reading or writing files,
- opening network connections,
- starting long-running work,
- printing user-facing output.

If a module needs to do real work, expose a function and let the importing
script decide when to call it.


## 10.12 What modules export by default

ZuzuScript does not require a separate export list.

Top-level names in a module become exports by default:

```zzs
const DEFAULT_PORT := 8080;

function parse_port (value) {
	return value + 0;
}

class ConfigError extends Exception;

function _parse_line (line) {
	return line;
}

{
	// Start a new scope
	function this_is_no_longer_top_level () {
		...;
	}
}
```

Other files can explicitly import those names:

```zzs
from app/config import DEFAULT_PORT, parse_port, ConfigError;
```

Even `_parse_line` is an export in the technical sense:

```zzs
from app/config import _parse_line;
```

But wildcard import treats leading `_` as private and skips it:

```zzs
from app/config import *;

// DEFAULT_PORT, parse_port, and ConfigError are imported.
// _parse_line is not.
```

Use that convention deliberately:

- public API names should be ordinary names,
- internal helpers should begin with `_` or not be on the top level.
- callers should not import `_` names unless they are doing something
  unusual and tightly coupled.

Imported top-level names can also be re-exported.

That allows facade modules:

```zzs
// modules/app/api.zzm
from app/config import ConfigError, load_config;
from app/report import build_report;
```

Then callers can import from one place:

```zzs
from app/api import ConfigError, load_config, build_report;
```

Facade modules are useful, but keep them boring. They should gather a
stable public surface, not hide complicated startup work.


## 10.13 Mutable exports are shared bindings

When you import a mutable top-level variable from a module, you are not
getting a disconnected copy.

You are referring to the module's binding.

```zzs
// modules/app/counter.zzm
let counter := 0;

function bump () {
	counter += 1;
	return counter;
}
```

```zzs
from app/counter import counter, bump;

say counter;  // 0
say bump();   // 1
say counter;  // 1
```

That can be useful for shared module state, but do not overuse it.

For most public APIs, prefer functions or objects:

```zzs
function current_count () {
	return counter;
}
```

That gives you room to change the internal representation later.


## 10.14 Import cycles

Avoid circular imports.

This is a cycle:

```text
app/a.zzm imports app/b.zzm
app/b.zzm imports app/a.zzm
```

Runtimes detect this and report an import-cycle style error.

When you hit a cycle, usually one of these fixes is best:

- move shared types into a third module,
- move constants into a small `app/constants.zzm`,
- pass dependencies as arguments instead of importing them,
- or split one module into a lower-level helper and a higher-level facade.

For example:

```text
app/model.zzm      // shared classes
app/load.zzm       // imports app/model
app/render.zzm     // imports app/model
app/api.zzm        // imports and re-exports load/render pieces
```

That shape keeps dependencies pointing in one direction.


## 10.15 Documenting modules with POD

ZuzuScript module files may contain POD documentation.

POD begins with headings such as `=head1`, uses paragraphs of plain text,
and ends before the code resumes with `=cut`.

Here is a small module with documentation:

```zzs
=encoding utf8

=head1 NAME

app/greeting - Greeting helpers for the application.

=head1 SYNOPSIS

  from app/greeting import greeting, Greeter;
  
  say greeting("Ada");
  let greeter := new Greeter( prefix: "Hello" );
  say greeter.greet("Lin");

=head1 DESCRIPTION

This module provides a simple function and class for producing greeting
strings.

=head1 EXPORTS

=head2 Functions

=over

=item C<greeting(String name)>

Parameters: C<name> is the display name. Returns: C<String>. Returns a
short greeting.

=back

=head2 Classes

=over

=item C<Greeter>

Configurable greeter object.

=over

=item C<< Greeter.greet(String name) >>

Parameters: C<name> is the display name. Returns: C<String>. Returns a
greeting using the object's C<prefix>.

=back

=back

=cut

function greeting (name) {
	return "Hello, " _ name _ ".";
}

class Greeter {
	let prefix := "Hello";

	method greet (name) {
		return prefix _ ", " _ name _ ".";
	}
}
```

Notice the indentation in the POD `SYNOPSIS`: code samples inside POD use
spaces, not tabs.


## 10.16 A practical POD structure

For user-facing modules, this structure works well:

```text
=encoding utf8

=head1 NAME

module/path - Concise purpose.

=head1 SYNOPSIS

Short runnable example.

=head1 DESCRIPTION

What the module does, what it does not do, and any important semantics.

=head1 EXPORTS

Public functions, classes, traits, constants, and variables.

=cut
```

For larger modules, group `EXPORTS` with second-level headings:

```text
=head2 Functions

=head2 Classes

=head2 Constants
```

Under each group, use `=over`, `=item`, and `=back`:

```text
=over

=item C<load_config(Path path)>

Parameters: C<path> is the config file. Returns: C<Dict>. Throws
C<ConfigError> when the file cannot be read or parsed.

=back
```

Document the public contract:

- parameters,
- return value,
- important thrown exceptions,
- side effects,
- runtime or platform differences,
- and examples that users can copy.

Do not document every private helper. If a helper starts with `_`, it
usually does not belong in `EXPORTS`.


## 10.17 Putting it together

A small project can now have a clean shape:

```text
todo/
  todo.zzs
  modules/
    todo/
      model.zzm
      store.zzm
      format.zzm
      api.zzm
```

`modules/todo/model.zzm`:

```zzs
=encoding utf8

=head1 NAME

todo/model - Todo item model.

=head1 SYNOPSIS

  from todo/model import TodoItem;

  let item := new TodoItem( title: "Write docs" );

=head1 EXPORTS

=head2 Classes

=over

=item C<TodoItem>

Todo item with a C<title> field and C<done> flag.

=back

=cut

class TodoItem {
	let title with get, set;
	let done with get, set := false;
}
```

`modules/todo/format.zzm`:

```zzs
from todo/model import TodoItem;

function format_item (item) {
	let mark := item.get_done() ? "x" : " ";
	return "[" _ mark _ "] " _ item.get_title();
}
```

`modules/todo/api.zzm`:

```zzs
from todo/model import TodoItem;
from todo/format import format_item;
```

`todo.zzs`:

```zzs
from todo/api import TodoItem, format_item;

let item := new TodoItem( title: "Write chapter 10" );
say format_item(item);
```

Run it:

```bash
zuzu -Imodules todo.zzs
```

That is the module system doing its main job:

- each file has a clear purpose,
- imports describe dependencies,
- `-Ilib` defines where project modules live,
- and POD travels with the module API.


## 10.18 Chapter summary

You now know how to:

- import named exports with `from module/path import name`,
- rename imports with `as`,
- use wildcard imports when appropriate,
- use `try import` for optional modules,
- use postfix `if` and `unless` for conditional imports,
- add project module roots with `-I`,
- understand the search order used by the module loader,
- write `.zzm` files,
- use default exports and underscore-private conventions,
- re-export names through facade modules,
- and document modules with POD.

Modules are the point where scripts start becoming systems.

Keep module boundaries small, name exports plainly, document the contract,
and make startup paths explicit with `-I`.

Once code is split into reusable pieces, text processing becomes a common
job for those pieces. Chapter 11 focuses on regular expressions: compact
patterns for recognizing and transforming strings.
