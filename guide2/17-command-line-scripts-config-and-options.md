# Chapter 17: Command-Line Scripts with Options and Config

Chapter 16 covered processes and environment variables.

This chapter is about putting several pieces together to write useful
command-line scripts:

- `__main__(argv)`,
- positional arguments,
- options and flags,
- generated usage text,
- config files,
- environment overrides,
- command-line overrides,
- validation,
- exit codes,
- and a medium-sized example script.

The example script is called `naplog`. It reads small local log files,
filters entries, and prints either text or JSON output. It is not a toy
"hello world" script, but it is still small enough to read in one sitting.
The complete script is available as [naplog.zzs](naplog.zzs).


## 17.1 What a script entrypoint receives

When a file is run as a script, the command-line runner calls
`__main__(argv)` if it exists.

```zzs
function __main__ ( argv ) {
	say argv.length();
	return 0;
}
```

`argv` is an array of command-line arguments, not including the ZuzuScript
runner itself or the script filename.

A command like this:

```sh
zuzu naplog.zzs --name Zia logs/today.log
```

passes this array to `__main__`:

```zzs
[ "--name", "Zia", "logs/today.log" ]
```

For command-line tools, return conventional exit codes:

- `0` for success,
- `1` for an operational failure,
- `2` for bad command-line usage.

For example:

```zzs
function __main__ ( argv ) {
	if ( argv.length() = 0 ) {
		say "usage: tool FILE...";
		return 2;
	}

	return 0;
}
```


## 17.2 Compact option parsing

Import `Getopt` from `std/getopt`:

```zzs
from std/getopt import Getopt;
```

The compact API is `Getopt.parse(argv, specs)`.

```zzs
let { ok, options, argv, error } := Getopt.parse(
	raw_argv,
	[
		"help|h",
		"verbose|v",
		"limit|l=i",
		"name|n=s",
	],
);
```

Each spec describes a long option, an optional short option, and an
optional value type:

- `help|h` is a Boolean flag,
- `verbose|v` is a Boolean flag,
- `limit|l=i` takes an integer,
- `name|n=s` takes a string.

The result is a dictionary:

```zzs
if ( not ok ) {
	say `invalid command line: ${error}`;
	return 2;
}
```

`options` contains the options. The destructured `argv` contains the
remaining positional arguments. In a real `__main__(argv)`, use a
different parameter name, such as `raw_argv`, if you want to bind the
remaining arguments as `argv`.

```zzs
let { options, argv } := Getopt.parse(
	[ "--verbose", "--limit", "5", "notes.txt" ],
	[ "verbose|v", "limit|l=i" ],
);

say options{verbose};             // 1
say options{limit};               // 5
say argv[0];                      // notes.txt
```


## 17.3 Schema-based option parsing

For real tools, prefer `Getopt.schema(argv, schema)`.

The schema form gives each option metadata, defaults, required flags, and
usage text.

```zzs
let parsed := Getopt.schema(
	argv,
	[
		{
			name: "help",
			short: "h",
			type: Boolean,
			desc: "Show help.",
		},
		{
			name: "limit",
			short: "l",
			type: Number,
			default: 20,
			desc: "Maximum number of entries.",
		},
		{
			name: "name",
			short: "n",
			type: String,
			multiple: true,
			desc: "Only include this name.",
		},
	],
);
```

Schema entries can use:

- `name`,
- `short`,
- `type`,
- `required`,
- `default`,
- `multiple`,
- `desc`.

The supported type values are usually `Boolean`, `Number`, and `String`.

`multiple: true` means the option may be repeated:

```sh
naplog --name Zia --name Zenia
```

The parsed option value is an array:

```zzs
let names := parsed{options}{name};
```

The schema result contains:

- `ok`,
- `options`,
- `argv`,
- `error`,
- `errors`,
- `usage`.

Use `usage` when printing help:

```zzs
if ( parsed{options}{help} ) {
	say "usage: naplog [options] FILE...";
	say "";
	say parsed{usage};
	return 0;
}
```

Use `error` when parsing failed:

```zzs
if ( not parsed{ok} ) {
	say parsed{error};
	say "";
	say parsed{usage};
	return 2;
}
```


## 17.4 Config files

Import `Config` from `std/config`:

```zzs
from std/config import Config;
```

`Config` wraps ordinary ZuzuScript data and adds loading, merging,
querying, and saving.

Start with defaults:

```zzs
let cfg := Config.from_data(
	{
		output: {
			format: "text",
			limit: 20,
		},
		filters: {
			names: [],
			tags: [],
			since: null,
		},
	},
);
```

Load a config file on top:

```zzs
from std/io import Path;

cfg.load_file( new Path("naplog.toml"), { optional: true } );
```

`std/config` detects common formats from the filename. Formats supported
through the standard library include JSON, YAML, TOML, INI, and TOON.

A TOML config for `naplog` might look like this:

```toml
[output]
format = "text"
limit = 10

[filters]
names = [ "Zia", "Zenia" ]
tags = [ "nap", "walk" ]
since = "2026-05-01"

[input]
files = [ "logs/may.log" ]
```

Read values with `get`:

```zzs
let format := cfg.get("/output/format", "text");
let limit := cfg.get("/output/limit", 20);
```

Or use path operators directly:

```zzs
let format := cfg @ "/output/format";
```

Use `require` for settings that must exist:

```zzs
let database := cfg.require("/database/path");
```

`require` throws if the setting is missing.


## 17.5 Layering defaults, config, env, and CLI

Most useful tools have more than one source of configuration.

A good precedence order is:

1. built-in defaults,
2. config files,
3. environment-style overrides,
4. command-line options,
5. positional arguments.

`std/config` is built for this style.

Use `merge_flat` for environment-style keys:

```zzs
cfg.merge_flat(
	{
		"NAPLOG__OUTPUT__FORMAT": "json",
		"NAPLOG__OUTPUT__LIMIT": "5",
	},
	{
		prefix: "NAPLOG__",
		separator: "__",
		lowercase: true,
		coerce: true,
	},
);
```

That writes:

```text
/output/format = "json"
/output/limit = 5
```

The `coerce` option turns strings such as `"true"`, `"5"`, `"12.5"`,
`"null"`, and simple JSON arrays or objects into real values.

Use `set` for command-line overrides:

```zzs
if ( opts{format} != null ) {
	cfg.set( "/output/format", opts{format} );
}

if ( opts{limit} != null ) {
	cfg.set( "/output/limit", opts{limit} );
}
```

Use `set_default` when a value should be filled in only if nothing else
has supplied it:

```zzs
cfg.set_default("/output/format", "text");
```


## 17.6 The `naplog` script

The rest of this chapter builds one script.

`naplog` reads log files containing lines like this:

```text
2026-05-01 Zia #nap #den slept through breakfast
2026-05-02 Fenn #walk checked the river path
2026-05-03 Zenia #nap borrowed Zia's blanket
```

Each nonblank line has:

1. an ISO date,
2. a name,
3. zero or more `#tags`,
4. a message.

Blank lines and comment lines starting with `#` are ignored.

The tool supports:

- `--config FILE`,
- `--name NAME`,
- `--tag TAG`,
- `--since DATE`,
- `--limit N`,
- `--format text|json`,
- `--output FILE`,
- positional input files.


## 17.7 Imports and defaults

Start with the modules the script needs:

```zzs
from std/config import Config;
from std/data/json import JSON;
from std/getopt import Getopt;
from std/io import Path, STDERR, STDOUT;
from std/proc import Env;
from std/string import join, split, starts_with, substr, trim;
```

Define defaults as ordinary data wrapped in a `Config`:

```zzs
function default_config () {
	return Config.from_data(
		{
			input: {
				files: [],
			},
			filters: {
				names: [],
				tags: [],
				since: null,
			},
			output: {
				format: "text",
				limit: 20,
				file: null,
			},
		},
		{
			source: "defaults",
		},
	);
}
```


## 17.8 Option schema and usage

Use `Getopt.schema` so help and errors can be generated from the same
metadata:

```zzs
function option_schema () {
	return [
		{
			name: "help",
			short: "h",
			type: Boolean,
			desc: "Show this help.",
		},
		{
			name: "config",
			short: "c",
			type: String,
			desc: "Read configuration from FILE.",
		},
		{
			name: "name",
			short: "n",
			type: String,
			multiple: true,
			desc: "Only include this name.",
		},
		{
			name: "tag",
			short: "t",
			type: String,
			multiple: true,
			desc: "Only include entries with this tag.",
		},
		{
			name: "since",
			short: "s",
			type: String,
			desc: "Only include entries on or after DATE.",
		},
		{
			name: "limit",
			short: "l",
			type: Number,
			desc: "Maximum number of entries.",
		},
		{
			name: "format",
			short: "f",
			type: String,
			desc: "Output format: text or json.",
		},
		{
			name: "output",
			short: "o",
			type: String,
			desc: "Write output to FILE instead of stdout.",
		},
	];
}

function usage ( option_usage ) {
	return "usage: naplog [options] FILE...\n\n"
		_ "Summarize sleepy field notes.\n\n"
		_ "Options:\n"
		_ option_usage
		_ "\n";
}
```


## 17.9 Environment and CLI overrides

`Env` cannot list every environment variable portably, so collect the
ones this script knows about:

```zzs
function env_overlay () {
	let overlay := {};
	for ( let name in [
		"NAPLOG__OUTPUT__FORMAT",
		"NAPLOG__OUTPUT__LIMIT",
		"NAPLOG__OUTPUT__FILE",
		"NAPLOG__FILTERS__SINCE",
	] ) {
		let value := Env.get(name, null);
		if ( value != null ) {
			overlay{( name )} := value;
		}
	}
	return overlay;
}
```

Apply command-line options after config and environment:

```zzs
function apply_cli_options ( cfg, opts, rest ) {
	if ( opts{name} != null ) {
		cfg.set( "/filters/names", opts{name} );
	}
	if ( opts{tag} != null ) {
		cfg.set( "/filters/tags", opts{tag} );
	}
	if ( opts{since} != null ) {
		cfg.set( "/filters/since", opts{since} );
	}
	if ( opts{limit} != null ) {
		cfg.set( "/output/limit", opts{limit} );
	}
	if ( opts{format} != null ) {
		cfg.set( "/output/format", opts{format} );
	}
	if ( opts{output} != null ) {
		cfg.set( "/output/file", opts{output} );
	}
	if ( rest.length() > 0 ) {
		cfg.set( "/input/files", rest );
	}
	return cfg;
}
```

This keeps precedence easy to see: later calls override earlier data.


## 17.10 Parsing log lines

A few small helper functions keep the main program readable:

```zzs
function as_array ( value ) {
	if ( value instanceof Array ) {
		return value;
	}
	if ( value == null ) {
		return [];
	}
	return [ value ];
}

function parse_log_line ( raw, source, line_number ) {
	let text := trim(raw);

	if ( text eq "" or starts_with( text, "#" ) ) {
		return null;
	}

	let parts := split( text, " " );
	if ( parts.length() < 2 ) {
		STDERR.say(
			`${source}:${line_number}: ignored malformed line`
		);
		return null;
	}

	let tags := [];
	let words := [];
	let i := 2;

	while ( i < parts.length() ) {
		let part := parts[i];
		if ( starts_with( part, "#" ) ) {
			tags.push( substr( part, 1 ) );
		}
		else {
			words.push(part);
		}
		i += 1;
	}

	return {
		date: parts[0],
		name: parts[1],
		tags: tags,
		message: join( " ", words ),
		source: "" _ source,
		line: line_number,
	};
}
```

This parser is intentionally simple. It is enough for a small local tool.
If the log format grows more complex, make the format more explicit
rather than adding too much cleverness to this function.


## 17.11 Filtering entries

Filtering is also easier as small named functions:

```zzs
function overlaps ( wanted, actual ) {
	if ( wanted.length() = 0 ) {
		return true;
	}

	for ( let item in wanted ) {
		if ( actual.contains(item) ) {
			return true;
		}
	}

	return false;
}

function entry_matches ( entry, cfg ) {
	let since := cfg.get( "/filters/since", null );
	if ( since != null and since ne "" and entry{date} lt since ) {
		return false;
	}

	let names := as_array( cfg.get( "/filters/names", [] ) );
	if ( names.length() > 0 and not( names.contains( entry{name} ) ) ) {
		return false;
	}

	let tags := as_array( cfg.get( "/filters/tags", [] ) );
	if ( not overlaps( tags, entry{tags} ) ) {
		return false;
	}

	return true;
}
```

The date comparison works because ISO dates in `YYYY-MM-DD` order sort
lexically in the same order as calendar dates.


## 17.12 Reading files and rendering output

The file-reading function returns ordinary dictionaries:

```zzs
function read_entries ( files ) {
	let entries := [];

	for ( let file_name in files ) {
		let path := new Path(file_name);
		let line_number := 0;

		for ( let raw in path.lines_utf8() ) {
			line_number += 1;
			let entry := parse_log_line(
				raw,
				path.to_String(),
				line_number,
			);

			if ( entry != null ) {
				entries.push(entry);
			}
		}
	}

	return entries;
}
```

Text output is compact and readable:

```zzs
function format_tags ( tags ) {
	let out := [];
	for ( let tag in tags ) {
		out.push( "#" _ tag );
	}
	return join( " ", out );
}

function render_text ( entries ) {
	let lines := [];

	for ( let entry in entries ) {
		let tags := format_tags( entry{tags} );
		lines.push(
			`${entry{date}} ${entry{name}} ${tags} ${entry{message}}`
		);
	}

	lines.push( "" );
	lines.push( `${entries.length()} entries` );
	return join( "\n", lines ) _ "\n";
}

function render_json ( entries ) {
	return ( new JSON( pretty: true, canonical: true ) ).encode(entries)
		_ "\n";
}
```

Write to a file when configured, otherwise write to stdout:

```zzs
function write_output ( text, cfg ) {
	let destination := cfg.get( "/output/file", null );

	if ( destination != null and destination ne "" ) {
		( new Path(destination) ).spew_utf8(text);
	}
	else {
		STDOUT.print(text);
	}
}
```


## 17.13 Main program

The entrypoint ties everything together:

```zzs
function __main__ ( argv ) {
	let schema := option_schema();
	let parsed := Getopt.schema( argv, schema );

	if ( parsed{options}{help} ) {
		STDOUT.print( usage( parsed{usage} ) );
		return 0;
	}

	if ( not parsed{ok} ) {
		STDERR.say(parsed{error});
		STDERR.say("");
		STDERR.print( usage( parsed{usage} ) );
		return 2;
	}

	let opts := parsed{options};
	let cfg := default_config();

	if ( opts{config} != null ) {
		cfg.load_file( new Path( opts{config} ) );
	}

	cfg.merge_flat(
		env_overlay(),
		{
			prefix: "NAPLOG__",
			separator: "__",
			lowercase: true,
			coerce: true,
		},
	);

	apply_cli_options( cfg, opts, parsed{argv} );

	let files := as_array( cfg.get( "/input/files", [] ) );
	if ( files.length() = 0 ) {
		STDERR.say("naplog: no input files");
		return 2;
	}

	let entries := [];
	for ( let entry in read_entries(files) ) {
		if ( entry_matches( entry, cfg ) ) {
			entries.push(entry);
		}
	}

	let limit := cfg.get( "/output/limit", 20 );
	if ( limit > 0 and entries.length() > limit ) {
		entries := entries.slice( 0, limit );
	}

	let format := cfg.get( "/output/format", "text" );
	if ( format ne "text" and format ne "json" ) {
		STDERR.say(`naplog: unknown output format '${format}'`);
		return 2;
	}

	let output := format eq "json"
		? render_json(entries)
		: render_text(entries);

	write_output( output, cfg );
	return 0;
}
```

The interesting part is not any one line. It is the shape:

1. parse options,
2. handle help and parse errors,
3. build config from defaults,
4. load a config file,
5. merge environment overrides,
6. merge CLI overrides,
7. validate,
8. do the work,
9. choose an output format,
10. return an exit code.

That shape scales well as a script grows.


## 17.14 Running it

With a log file:

```text
2026-05-01 Zia #nap #den slept through breakfast
2026-05-02 Fenn #walk checked the river path
2026-05-03 Zenia #nap borrowed Zia's blanket
2026-05-04 Zachary #snack found the oat biscuits
```

Run:

```sh
zuzu naplog.zzs --tag nap --since 2026-05-02 logs/may.log
```

Expected text output:

```text
2026-05-03 Zenia #nap borrowed Zia's blanket

1 entries
```

For JSON:

```sh
zuzu naplog.zzs --tag nap --format json logs/may.log
```

Or with a config file:

```toml
[filters]
tags = [ "nap" ]
since = "2026-05-02"

[output]
format = "json"
limit = 5
```

Run:

```sh
zuzu naplog.zzs --config naplog.toml logs/may.log
```

And override one setting from the environment:

```sh
NAPLOG__OUTPUT__FORMAT=text zuzu naplog.zzs --config naplog.toml logs/may.log
```

The precedence is:

```text
defaults < config file < environment < command line < positional files
```


## 17.15 What to copy from this pattern

For your own command-line scripts:

- parse `argv`, do not read a hidden global argument list,
- use schema-based options once a tool has more than a few flags,
- print help from the same schema that parses the options,
- keep defaults as data,
- layer config in one obvious order,
- use positional arguments for the main inputs,
- validate before doing irreversible work,
- return conventional exit codes,
- and keep parsing, config, business logic, and output in separate
  functions.

The exact `naplog` format is not important. The durable idea is that a
script should have a clear outside boundary: command-line arguments,
configuration, environment, files, and exit status are all made explicit.
