# Chapter 18: Testing, Packaging, and Sharing Code

Chapter 17 built a useful command-line script.

This chapter turns that kind of code into something other people can use:

- tests for modules,
- a distribution archive,
- upload to Zuzulang.org,
- and installation with Zuzuzoo.

The example is a small distribution called `naplog-tools`. It provides a
module for parsing nap log lines and a script that can use that module.

The shape should feel familiar by now: clear module boundaries from Chapter
10, useful failure behaviour from Chapter 9, and the command-line edge from
Chapter 17.


## 18.1 Why distributions start with modules

A script is easy to run, but a module is easier to test and reuse.

Instead of putting all of `naplog` into one file, move the parts that make
sense on their own into a module:

```text
naplog-tools-1.0.0/
  modules/
    naplog/
      parse.zzm
  scripts/
    naplog-summary.zzs
```

The module path is `naplog/parse`. The file path under the module root is
`modules/naplog/parse.zzm`.

Here is a deliberately small module:

```zzs
from std/string import trim, split;

function parse_entry (line) {
	let parts := split(line, "|");

	if ( parts.length() < 3 ) {
		die `invalid naplog entry: ${line}`;
	}

	return {
		name: trim(parts[0]),
		minutes: trim(parts[1]) + 0,
		note: trim(parts[2]),
	};
}

function entry_summary (entry) {
	return `${entry{name}} slept for ${entry{minutes}} minutes`;
}
```

Top-level names are exports, so another file can import those functions:

```zzs
from naplog/parse import parse_entry, entry_summary;
```

That is the boundary you want to test. The script can stay thin: it reads
arguments and files, then calls the module.


## 18.2 A first test with `test/more`

ZuzuScript tests commonly use `test/more`.

Create `tests/parse.zzs`:

```zzs
from test/more import *;
from naplog/parse import parse_entry, entry_summary;

let entry := parse_entry("Zia | 42 | rainy afternoon");

is(entry{name}, "Zia", "parse_entry reads the name");
is(entry{minutes}, 42, "parse_entry reads the minutes");
is(entry{note}, "rainy afternoon", "parse_entry reads the note");
is(
	entry_summary(entry),
	"Zia slept for 42 minutes",
	"entry_summary formats a short sentence",
);

done_testing();
```

Run it with the distribution's `modules` directory in the module search
path:

```sh
zuzu -Imodules tests/parse.zzs
```

The output is TAP:

```tap
ok 1 - parse_entry reads the name
ok 2 - parse_entry reads the minutes
ok 3 - parse_entry reads the note
ok 4 - entry_summary formats a short sentence
1..4
```

A passing test file should:

- print a valid plan line, such as `1..4`,
- contain no `not ok` lines,
- and exit with status `0`.

`done_testing()` is convenient while a test file is growing. It prints the
plan after all tests have run.

If you already know the exact number of tests, you can put the plan first:

```zzs
from test/more import *;

plan(2);

ok(true, "truth is truthy");
is(2 + 2, 4, "addition still works");
```

Use one style per file: either call `plan(n)` before the tests, or call
`done_testing()` after the tests.


## 18.3 Useful `test/more` functions

The most common assertions are small:

```zzs
ok(value, "value is truthy");
is(got, expected, "values are equal");
isnt(got, unexpected, "values are different");
like(text, /pattern/, "text matches a pattern");
unlike(text, /pattern/, "text does not match a pattern");
pass("this point was reached");
fail("this point should not be reached");
```

Use `diag` when a failure needs more context:

```zzs
diag(`parsed entry: ${entry}`);
```

Use `exception` when a function should fail:

```zzs
let error := exception(function () {
	parse_entry("not enough fields");
});

isnt(error, null, "bad input throws an exception");
like(error{message}, /invalid naplog entry/, "exception explains the input");
```

Use `subtest` to group related checks:

```zzs
subtest("invalid input", function () {
	let error := exception(function () {
		parse_entry("Zia only");
	});

	isnt(error, null, "invalid input throws");
	like(error{message}, /invalid naplog entry/, "message names the problem");
});
```

Do not call `done_testing()` inside a `subtest`. The outer test file owns
the final plan.

If a whole file cannot run in the current environment, skip it before
running assertions:

```zzs
from test/more import *;

skip_all("filesystem support is not available")
	unless requires_capability("fs");
```

That is better than producing misleading failures on a runtime that cannot
possibly support the feature being tested.


## 18.4 Distribution layout

Zuzu distributions use ZDF-1, the Zuzu Distribution Format.

A ZDF-1 distribution is a `tar.gz` archive with one top-level directory.
The directory name is the distribution name and version joined with a
hyphen:

```text
naplog-tools-1.0.0/
  zuzu-distribution.json
  README.md
  LICENSE
  modules/
    naplog/
      parse.zzm
  scripts/
    naplog-summary.zzs
  tests/
    parse.zzs
```

The important directories are:

- `modules/` for installable `.zzm` modules,
- `scripts/` for installable `.zzs` scripts,
- `tests/` for tests Zuzuzoo should run before installation,
- `inc/` for private helper modules used while building or testing,
- and the distribution root for metadata and documentation.

Files in `inc/` are not installed. They are for the distribution's own
build and test code.

The root may also contain `Build.zzs`. If present, Zuzuzoo runs it after
unpacking the archive and before planning the install.


## 18.5 Distribution metadata

Every distribution needs `zuzu-distribution.json` at the distribution root.

For `naplog-tools-1.0.0`, use:

```json
{
	"name": "naplog-tools",
	"version": "1.0.0",
	"author": "Your Name",
	"license": "MIT",
	"status": "trial",
	"abstract": "Small helpers for reading naplog files.",
	"repo": "https://example.com/you/naplog-tools",
	"dependencies": {}
}
```

The required fields are:

- `name`,
- `version`,
- `author`,
- and `license`.

The common optional fields are:

- `abstract`, a short plain-text description,
- `repo`, an `http` or `https` URL for the source repository,
- `status`, either `trial` or `stable`,
- and `dependencies`.

Dependencies are module names, not archive names:

```json
{
	"dependencies": {
		"some/module": "1.2.0",
		"another/module": "0"
	}
}
```

The value is the minimum acceptable version. Use `"0"` when any installed
version is acceptable.

Keep the identity consistent:

- metadata name: `naplog-tools`,
- metadata version: `1.0.0`,
- top-level directory: `naplog-tools-1.0.0`,
- archive filename: `naplog-tools-1.0.0.tar.gz`.

That consistency matters when the distribution is uploaded.


## 18.6 Building the archive

From the directory that contains `naplog-tools-1.0.0/`, run:

```sh
tar -czf naplog-tools-1.0.0.tar.gz naplog-tools-1.0.0
```

Before sharing the archive, test it locally:

```sh
zuzuzoo install --dry-run ./naplog-tools-1.0.0.tar.gz
```

`--dry-run` validates and plans the install without writing modules,
scripts, or installed metadata.

Then test a real local install:

```sh
zuzuzoo install ./naplog-tools-1.0.0.tar.gz
```

During installation, Zuzuzoo:

1. unpacks the archive,
2. reads `zuzu-distribution.json`,
3. runs `Build.zzs` if the archive has one,
4. resolves dependencies,
5. runs tests from `tests/`,
6. installs modules and scripts,
7. and writes installed metadata last.

Tests run with the distribution's `modules/` and `inc/` directories in the
module search path.

If a test fails, installation aborts. Use this only when you understand the
risk:

```sh
zuzuzoo install --force ./naplog-tools-1.0.0.tar.gz
```

That records that the installation continued despite test failures.

To skip tests entirely:

```sh
zuzuzoo install --no-test ./naplog-tools-1.0.0.tar.gz
```

`--no-test` is useful for debugging packaging problems. Do not use it as a
routine publishing habit.


## 18.7 Installing locations

By default, Zuzuzoo installs for the current user:

- modules under `$HOME/.zuzu/modules`,
- scripts under `$HOME/.zuzu/bin`,
- metadata under `$HOME/.zuzu/meta`.

For a system-wide install, use:

```sh
zuzuzoo install --global ./naplog-tools-1.0.0.tar.gz
```

On Unix-like systems, that normally means:

- modules under `/var/lib/zuzu/modules`,
- scripts under `/usr/local/bin`,
- metadata under `/var/lib/zuzu/meta`.

You can choose explicit directories:

```sh
zuzuzoo install \
  --lib-dir local/modules \
  --bin-dir local/bin \
  --meta-dir local/meta \
  ./naplog-tools-1.0.0.tar.gz
```

That is useful for tests, local experiments, and build jobs that should not
touch your normal Zuzu installation.


## 18.8 Uploading to Zuzulang.org

Once the archive installs cleanly, upload it to Zuzulang.org.

The usual flow is:

1. sign in at `https://zuzulang.org/`,
2. verify your email address,
3. make sure your account has upload access,
4. open `https://zuzulang.org/account/distributions`,
5. choose the `.tar.gz` archive,
6. and submit the upload form.

The upload validator checks the same things Zuzuzoo cares about, plus the
rules needed for a public repository:

- the upload must be a `.tar.gz` file,
- the archive must contain one top-level directory,
- the filename, directory name, and metadata identity must match,
- `zuzu-distribution.json` must be valid JSON,
- module files must live under `modules/`,
- scripts must live under `scripts/`,
- tests must live under `tests/`,
- and unsupported files are rejected.

For this example, all of these names should agree:

```text
naplog-tools-1.0.0.tar.gz
naplog-tools-1.0.0/
{
	"name": "naplog-tools",
	"version": "1.0.0"
}
```

After a successful upload, Zuzulang.org redirects to the public
distribution page. The package can then be discovered through the modules
it provides, such as `naplog/parse`.


## 18.9 Installing from Zuzulang.org

Installing a local archive is useful while developing:

```sh
zuzuzoo install ./naplog-tools-1.0.0.tar.gz
```

Installing by module name is useful after upload:

```sh
zuzuzoo install naplog/parse
```

Zuzuzoo asks Zuzulang.org for the distribution that provides that module,
downloads the archive, installs its dependencies, runs its tests, and then
installs the files.

Use `latest` to check what Zuzulang.org reports as the latest stable
distribution for a module:

```sh
zuzuzoo latest naplog/parse
```

Use `query` to inspect a local installation:

```sh
zuzuzoo query naplog/parse
```

Use `list` to see installed distributions and modules:

```sh
zuzuzoo list
```

Use `verify` to check installed files against installed metadata:

```sh
zuzuzoo verify naplog/parse
```

Use `remove` when you no longer want the module:

```sh
zuzuzoo remove naplog/parse
```

If you want to see what an install or removal would do without changing the
installation, add `--dry-run`:

```sh
zuzuzoo install --dry-run naplog/parse
zuzuzoo remove --dry-run naplog/parse
```


## 18.10 A publishing checklist

Before uploading a distribution, check:

- the reusable code is in modules, not hidden inside scripts,
- tests import the modules exactly as users will import them,
- tests pass from the distribution root,
- `zuzu-distribution.json` has the right name and version,
- the top-level directory is named `${name}-${version}`,
- the archive filename is `${name}-${version}.tar.gz`,
- the archive contains only expected ZDF-1 paths,
- `zuzuzoo install --dry-run` succeeds,
- and a local `zuzuzoo install` succeeds without `--force`.

That gives you the useful loop:

1. write modules,
2. test them with `test/more`,
3. package them as ZDF-1,
4. upload the archive,
5. install by module name with Zuzuzoo.

At that point your code has moved from "a script on my machine" to a
distribution another ZuzuScript user can install, verify, and remove.


## 18.11 Philosophy

Some things you may want to think about when publishing modules for
public use.

### Versioning

Version numbers supported by zuzuzoo and zuzulang.org are dotted decimals
like "1.2.0", "1.3.0.1", "0.1", or "1.20260527". Zuzu doesn't impose any
meaning to these version numbers, other than bigger numbers representing
newer releases than smaller numbers. However, you should consider using
[SemVer 2.0.0](https://semver.org/) or an approximation of it to help users
understand what your version numbers mean.

### Backwards compatibility

When releasing a new version of your module, you should attempt to maintain
backwards compatibility with old versions. Code that runs on version 2.0 of
your module should ideally still work with version 2.1, and even version 3.0.

Adding new things to your API (new functions, new classes, new methods)
is unlikely to break other people's existing code. Removing things from
your API or renaming them is likely to break people's code, so do that
only as a last resort. If you currently export a function that takes two
arguments and decide that in the next version it should support a third
argument, make that argument optional so that existing code that passes
two arguments still works.

If you find your old API is holding you back and you want to radically
change things, consider just creating a new module or a new distribution
and recommending people switch to that. Document that the old module
is only kept for legacy support.

Code that runs today should still run in ten years.

Once code is published, people will use it with real data. Chapter 19 turns
to one of the most common shapes of that data: tables, whether they live in
CSV files or SQL databases.
