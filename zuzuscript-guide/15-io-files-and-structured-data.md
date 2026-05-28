# Chapter 15: Files, Paths, and Structured Data

Chapter 14 introduced tasks and asynchronous work.

This chapter introduces filesystem IO:

- `std/io`,
- `Path` objects,
- reading and writing text and binary files,
- synchronous and asynchronous file access,
- file metadata,
- temporary files and directories,
- standard streams,
- and structured file formats such as JSON, CBOR, YAML, TOML, KDL, and
  XML.

The examples in this chapter use files on the local machine. That means
they require a runtime with filesystem support. `std/io` is available in
the command-line implementations and in the JavaScript implementations
that have host filesystem access, such as Node and Electron. It is not
available in a normal browser page.


## 15.1 Importing `std/io`

Import `Path` from `std/io` when you want to work with filesystem paths:

```zzs
from std/io import Path;

let notes := new Path("notes.txt");
```

`Path` objects represent filesystem paths. They are not just strings with
methods attached. Converting user input to a `Path` early makes the rest of
the code clearer:

```zzs
function notes_file ( String name ) {
	let dir := new Path("notes");
	return dir.child(`${name}.txt`);
}
```

If you need the string form, call `to_String()`:

```zzs
let path := new Path("notes/today.txt");
say path.to_String;
```


## 15.2 Building paths

Avoid building paths by concatenating strings with `/`. Different
platforms use different path conventions, and even on one platform it is
easy to create doubled or missing separators.

Use `child`, `parent`, and `sibling` instead:

```zzs
from std/io import Path;

let root := new Path("data");
let settings := root.child("settings.json");
let backup := settings.sibling("settings.backup.json");

say settings.parent.to_String;      // data
say backup.basename;                // settings.backup.json
```

Useful `Path` constructors and helpers include:

- `new Path(string)` to wrap a path string,
- `Path.cwd()` for the current working directory,
- `Path.rootdir()` for the filesystem root,
- `Path.join(parts)` to join an array of path parts,
- `Path.split(path)` to split a path into parts,
- `Path.normalize(path)` to clean up redundant separators and `.` parts.

For example:

```zzs
let joined := Path.join( [ "data", "profiles", "zia.json" ] );
let clean := Path.normalize("data/./profiles/zia.json");
```

`absolute`, `canonpath`, and `realpath` are useful when you need a more
explicit form:

```zzs
let path := new Path("data/settings.json");

say path.absolute.to_String;
say path.canonpath.to_String;

let real := path.realpath;
say real.to_String unless real ≡ null;
```

`realpath()` resolves the real filesystem path and returns `null` if the
path cannot be resolved.


## 15.3 Checking what exists

Use the query methods before taking action when the absence or type of a
path changes what your program should do:

```zzs
let path := new Path("data/settings.json");

if ( path.exists and path.is_file ) {
	say "settings are present";
}
else {
	say "settings are missing";
}
```

Common query methods include:

- `exists()`,
- `is_file()`,
- `is_dir()`,
- `is_absolute()`,
- `is_relative()`,
- `is_rootdir()`,
- `subsumes(other)`.

`subsumes` is useful when checking whether one path is inside another:

```zzs
let data := new Path("data").absolute;
let config := data.child("settings.json");

if ( data.subsumes(config) ) {
	say "inside the data directory";
}
```


## 15.4 Reading and writing text

For ordinary text files, use the UTF-8 methods:

```zzs
from std/io import Path;

let file := Path.tempfile;

file.spew_utf8("first line\nsecond line\n");
say file.slurp_utf8();

file.append_utf8("third line\n");
```

The names are deliberately plain:

- `spew_utf8(text)` writes the whole file,
- `append_utf8(text)` appends to the file,
- `slurp_utf8()` reads the whole file,
- `lines_utf8()` reads the file as an array of lines.

Use `lines_utf8()` when the file is naturally line-based:

```zzs
let lines := file.lines_utf8;

for ( const line in lines ) {
	say trim(line);
}
```

Use `each_line` when you want to process lines one at a time:

```zzs
file.each_line(
	function ( line ) {
		say trim(line);
	}
);
```

`next_line()` reads one line. This is useful for simple streaming-style
programs:

```zzs
let first := file.next_line;
say first unless first ≡ null;
```


## 15.5 Binary files

The methods without `_utf8` work with `BinaryString` values:

```zzs
from std/io import Path;

let file := Path.tempfile;
let bytes := ~to_binary("ABC");

file.spew(bytes);
let again := file.slurp;

say typeof again;             // BinaryString
```

The binary methods are:

- `spew(bytes)`,
- `append(bytes)`,
- `slurp()`,
- `lines()`.

Use the UTF-8 methods for text and the binary methods for bytes. Do not
rely on accidental conversion between the two. It is clearer and more
portable to be explicit.


## 15.6 Creating, moving, and removing

`Path` also provides common filesystem actions:

```zzs
from std/io import Path;

let dir := Path.tempdir;
let source := dir.child("source.txt");
let copy := dir.child("copy.txt");
let moved := dir.child("moved.txt");

source.spew_utf8("hello\n");

source.copy(copy);
copy.move(moved);

moved.remove;
dir.remove_tree;
```

Useful action methods include:

- `mkdir()` to create a directory,
- `mkdir_exclusive()` to create one directory only if it does not exist,
- `touch()` to create or update a file,
- `touchpath()` to create parent directories and then touch a file,
- `copy(to)`,
- `move(to)`,
- `remove()`,
- `remove_tree()` for directory trees,
- `chmod(mode)`.

`mkdir_exclusive()` returns true when it creates the directory and false
when the path already exists:

```zzs
let cache := Path.tempdir.child("cache");

if ( cache.mkdir_exclusive ) {
	say "created cache directory";
}
else {
	say "cache directory already exists";
}
```

Use `remove_tree()` carefully. It removes a directory and its contents.


## 15.7 Listing directories

Use `children()` to list the direct children of a directory:

```zzs
let dir := Path.tempdir;

dir.child("one.txt").spew_utf8("one\n");
dir.child("two.txt").spew_utf8("two\n");

for ( const child in dir.children ) {
	say child.basename;
}

dir.remove_tree;
```

`iterator()` returns a `PathIterator`, which is useful when the directory
may be large and you do not want to build one array first.

Use `Path.glob(pattern)` when you want shell-style filename matching:

```zzs
let text_files := Path.glob("data/*.txt");

for ( const file in text_files ) {
	say file.basename;
}
```

For recursive traversal, use `visit(callback, ...)`. The callback receives
each visited path.


## 15.8 File metadata

Use `size()` for the file size in bytes and `size_human()` for a
human-readable size:

```zzs
let file := Path.tempfile;
file.spew_utf8("sleep report\n");

say file.size;                // 13
say file.size_human;          // runtime-specific display string
```

For richer metadata, use `stat()`:

```zzs
let meta := file.stat;

say meta{size};
say meta{mtime};
```

`stat()` returns a dictionary of metadata from the host filesystem.
Common keys include:

- `size`,
- `mode`,
- `uid`,
- `gid`,
- `atime`,
- `mtime`,
- `ctime`.

The time fields are numeric timestamps:

- `atime` is the last access time,
- `mtime` is the last modification time,
- `ctime` is the last status-change time.

Do not assume that `ctime` means "creation time". On Unix-like systems it
usually does not.

There are portability limits here. Fields such as `uid`, `gid`, `mode`,
`blksize`, and `blocks` reflect the host operating system. On some
platforms they may be missing, zero, or less meaningful. When writing
portable code, use defaults or test for the key you need:

```zzs
let meta := file.stat;
let size := meta.get("size", 0);
let mode := meta.get("mode", null);
```

`lstat()` is like `stat()`, but asks about the path itself rather than the
target of a symbolic link.


## 15.9 Temporary files and directories

`Path.tempfile` creates a temporary file and returns it as a `Path`:

```zzs
let file := Path.tempfile;
file.spew_utf8("temporary notes\n");
say file.slurp_utf8;
file.remove;
```

`Path.tempdir()` creates a temporary directory:

```zzs
let dir := Path.tempdir;
let file := dir.child("settings.json");

file.spew_utf8("{}\n");
dir.remove_tree;
```

Temporary paths are useful for tests, scratch work, and intermediate
files. They are still real filesystem paths. Unless you arrange otherwise,
your program should clean them up when it is finished.


## 15.10 Asynchronous file access

The async file methods return `Task` objects. Await them just like the
tasks from Chapter 14.

```zzs
from std/io import Path;

async function __main__ ( argv ) {
	let file := Path.tempfile();

	await {
		file.spew_utf8_async("one\ntwo\n");
	};

	let text := await {
		file.slurp_utf8_async;
	};

	say text;
	file.remove;
}
```

The text async methods are:

- `spew_utf8_async(text)`,
- `append_utf8_async(text)`,
- `slurp_utf8_async()`,
- `lines_utf8_async()`.

The binary async methods are:

- `spew_async(bytes)`,
- `append_async(bytes)`,
- `slurp_async()`,
- `lines_async()`.

Async methods are most useful when file access is part of a larger
concurrent workflow:

```zzs
from std/io import Path;
from std/task import all;

async function __main__ ( argv ) {
	let a := Path.tempfile();
	let b := Path.tempfile();

	await {
		all( [
			a.spew_utf8_async("alpha\n"),
			b.spew_utf8_async("beta\n"),
		] );
	};

	let texts := await {
		all( [
			a.slurp_utf8_async(),
			b.slurp_utf8_async(),
		] );
	};

	say texts[0] _ texts[1];

	a.remove;
	b.remove;
}
```


## 15.11 Standard input and output

`std/io` also exports `STDIN`, `STDOUT`, and `STDERR`:

```zzs
from std/io import STDOUT, STDERR;

STDOUT.say("normal output");
STDERR.say("diagnostic output");
```

`STDOUT` and `STDERR` provide `print` and `say`.

`STDIN` provides `next_line` and `each_line`:

```zzs
from std/io import STDIN;

STDIN.each_line(
	function ( line ) {
		say trim(line);
	}
);
```


## 15.12 Structured data files

Plain text is enough for logs and simple notes. For settings, cached
results, indexes, and data exchange, use a structured format.

ZuzuScript's standard library includes several codec modules:

- `std/data/json`,
- `std/data/cbor`,
- `std/data/yaml`,
- `std/data/toml`,
- `std/data/kdl`,
- `std/data/xml`.

Most of these have the same rough shape:

- create a codec object,
- `encode(value)` turns data into text or bytes,
- `decode(encoded)` turns text or bytes back into data,
- `encode_binarystring(value)` uses a `BinaryString`,
- `decode_binarystring(bytes)` reads a `BinaryString`,
- `load(path)` reads from a `Path`,
- `dump(path, value)` writes to a `Path`.

The details vary by format. JSON, YAML, and TOML are text formats. CBOR is
binary. KDL and XML expose richer document models because those formats
have structure that is not always a plain dictionary or array.

`load` and `dump` expect `Path` objects, not path strings:

```zzs
let path := new Path("settings.json");
json.dump(path, settings);
```


## 15.13 JSON files

JSON is a good default for simple structured data.

```zzs
from std/io import Path;
from std/data/json import JSON;

let json := new JSON( pretty: true, canonical: true );
let file := Path.tempfile;

json.dump(
	file,
	{
		name: "Zia",
		naps: 3,
		tags: [ "quiet", "careful" ],
	},
);

let loaded := json.load(file);

say loaded{name};             // Zia
say loaded{naps};             // 3

file.remove();
```

`pretty` asks for readable output. `canonical` asks for stable ordering
where the implementation supports it, which is useful for generated files
that will be reviewed in version control. The JSON codec also supports a
`pairlists` option which returns JSON objects as Zuzu PairLists instead
of Dicts; this preserves key order and even allows duplicate keys.

For async file access, keep the codec in memory and use async `Path`
methods:

```zzs
from std/data/json import JSON;
from std/io import Path;

async function load_json_async ( Path path ) {
	let text := await {
		path.slurp_utf8_async();
	};

	return (new JSON()).decode(text);
}

async function save_json_async ( Path path, value ) {
	let text := (new JSON( pretty: true, canonical: true )).encode(value);

	await {
		path.spew_utf8_async(text);
	};
}
```

The codec's `load` and `dump` methods are synchronous conveniences. When
you want asynchronous IO, combine `encode`/`decode` with the async file
methods.


## 15.14 CBOR files

CBOR is a binary structured format. It is useful when you want compact
files or when preserving binary data matters.

```zzs
from std/data/cbor import CBOR;
from std/io import Path;

let cbor := new CBOR();
let file := Path.tempfile();

cbor.dump(
	file,
	{
		name: "Zenia",
		score: 42,
		active: true,
	},
);

let loaded := cbor.load(file);
say loaded{score};            // 42

file.remove();
```

CBOR `encode` returns bytes, not text:

```zzs
let bytes := cbor.encode( [ 1, 2, 3 ] );
say typeof bytes;             // BinaryString
```

The async form uses the binary path methods:

```zzs
async function save_cbor_async ( Path path, value ) {
	let bytes := (new CBOR()).encode(value);

	await {
		path.spew_async(bytes);
	};
}

async function load_cbor_async ( Path path ) {
	let bytes := await {
		path.slurp_async();
	};

	return (new CBOR()).decode(bytes);
}
```


## 15.15 YAML and TOML

YAML and TOML are often used for configuration files.

YAML is flexible and human-friendly:

```zzs
from std/data/yaml import YAML;
from std/io import Path;

let yaml := new YAML( pretty: true, canonical: true );
let file := Path.tempfile();

yaml.dump(
	file,
	{
		title: "Nap schedule",
		enabled: true,
		times: [ "10:00", "14:00" ],
	},
);

let loaded := yaml.load(file);
say loaded{title};

file.remove();
```

TOML is stricter and is especially common for settings:

```zzs
from std/data/toml import TOML;
from std/io import Path;

let toml := new TOML( pretty: true, canonical: true );
let file := Path.tempfile;

toml.dump(
	file,
	{
		title: "Application settings",
		owner: {
			name: "Zachary",
		},
	},
);

let loaded := toml.load(file);
say loaded{owner}{name};

file.remove();
```

TOML documents have a table-shaped top level, so the value you encode or
dump should be a dictionary or compatible mapping.


## 15.16 KDL

KDL is a document format made of named nodes with arguments, properties,
and children. Because that structure is not the same as a normal
dictionary, `std/data/kdl` exposes KDL-specific classes.

```zzs
from std/data/kdl import KDL, KDLDocument, KDLNode, KDLValue;
from std/io import Path;

let kdl := new KDL();
let file := Path.tempfile();

let document := new KDLDocument(
	nodes: [
		new KDLNode(
			name: "person",
			args: [
				new KDLValue( type: "string", value: "Zia" ),
			],
			props: new PairList(
				[
					"role",
					new KDLValue( type: "string", value: "planner" ),
				],
			),
		),
	],
);

kdl.dump(file, document);

let loaded := kdl.load(file);
say loaded.nodes()[0].name();        // person

file.remove();
```

Use KDL when you want a node-oriented configuration or document format.
Use JSON, YAML, or TOML when ordinary dictionaries and arrays are a better
fit.


## 15.17 XML

XML is also structured as a document rather than as a plain dictionary.
`std/data/xml` therefore works with XML document and node objects.

```zzs
from std/data/xml import XML;
from std/io import Path;

let doc := XML.parse(
	"<friends><friend name='Zia'/><friend name='Zenia'/></friends>"
);

let file := Path.tempfile;
XML.dump(file, doc, true);

let loaded := XML.load(file);
let names := loaded.findnodes("/friends/friend/@name");

for ( const name in names ) {
	say name.to_String;
}

file.remove();
```

`XML.parse(text)` parses XML from a string. `XML.load(path)` reads and
parses a file. `XML.dump(path, document, pretty)` writes a document or
node to a file.

XML has its own querying and document APIs. It is the right choice when
you are interoperating with XML files or protocols, not merely because you
need a settings file.


## 15.18 Choosing a format

There is no single correct structured format:

- JSON is widely supported and simple.
- CBOR is compact and binary.
- YAML is pleasant for humans but has more syntax.
- TOML is good for configuration with clear tables.
- KDL is good for node-oriented documents.
- XML is best when the surrounding ecosystem already uses XML.

For generated files, prefer stable output where possible:

```zzs
let json := new JSON( pretty: true, canonical: true );
json.dump(new Path("generated-index.json"), index);
```

For binary formats, remember to use binary file methods in async code:

```zzs
let bytes := (new CBOR()).encode(index);

await {
	new Path("index.cbor").spew_async(bytes);
};
```


## 15.19 A complete small example

This script keeps a small JSON settings file in a directory, creating the
directory when needed.

```zzs
from std/data/json import JSON;
from std/io import Path;

function load_settings ( Path file ) {
	let json := new JSON( pretty: true, canonical: true );

	if ( file.exists ) {
		return json.load(file);
	}

	return {
		theme: "plain",
		recent: [],
	};
}

function save_settings ( Path file, settings ) {
	let json := new JSON( pretty: true, canonical: true );

	file.parent.mkdir;
	json.dump(file, settings);
}

let dir := new Path("data");
let file := dir.child("settings.json");

let settings := load_settings(file);
settings{recent}.push("chapter-15");
save_settings(file, settings);
```

The same shape works for other codecs:

- choose the codec,
- convert the incoming path to a `Path`,
- read with `load` or async `slurp`,
- update ordinary ZuzuScript data,
- write with `dump` or async `spew`.

Keep the boundary clear: `Path` handles the filesystem, while the codec
handles the file format.

Files are one major boundary between a script and the outside world.
Chapter 16 turns to another: environment variables, child processes,
signals, and the runtime state exposed through `__system__`.
