# Chapter 12: Paths, Destructuring, and Nested Structures

Chapter 11 was about finding patterns in text.

This chapter is about finding values in data.

Real data is rarely flat. A config file, API response, parsed document, or
application state object is usually a Dict containing Arrays containing
more Dicts. ZuzuScript gives you two main tools for that shape of work:

- destructuring declarations for pulling named top-level values into local
  bindings,
- path operators and `ZZPath` for reaching through nested structures.

We will also cover references to path-selected locations, because they
make more sense once you have seen path lvalues.

This chapter covers:

- destructuring declarations with `let` and `const`,
- defaults, aliases, typed bindings, and computed keys,
- manual access to nested Arrays and Dicts,
- `@`, `@@`, and `@?`,
- the default `ZZPath` syntax,
- reusable `ZZPath` objects,
- lexical path syntax switching,
- path assignment,
- and the pseudo-reference operator `\`.


## 12.1 A running nested value

We will use this value through most of the chapter:

```zzs
let den := {
	name: "The Quiet Den",
	settings: {
		light: "dim",
		snack: "berries",
	},
	friends: [
		{
			name: "Zia",
			species: "raccoon",
			mood: "sleepy",
			naps: 3,
			tags: [ "night", "blanket" ],
		},
		{
			name: "Zenia",
			species: "raccoon",
			mood: "awake",
			naps: 1,
			tags: [ "tea" ],
		},
		{
			name: "Zachary",
			species: "raccoon",
			mood: "busy",
			naps: 0,
			tags: [ "tools", "map" ],
		},
		{
			name: "Fenn",
			species: "fox",
			mood: "curious",
			naps: 1,
			tags: [ "fern", "compass" ],
		},
	],
};
```

Without any special syntax, you can already reach into it:

```zzs
say den{settings}{light};       // dim
say den{friends}[0]{name};      // Zia
say den{friends}[2]{tags}[1];   // map
```

This is fine for a few direct lookups. It becomes noisy when you need to
ask questions like "which friends are sleepy?" or "update every mood".


## 12.2 Destructuring declarations

Many languages call this feature destructuring. In ZuzuScript, the
supported form is specifically a declaration:

```zzs
let { name, settings, friends } := den;

say name;                  // The Quiet Den
say settings{snack};       // berries
say friends[0]{name};      // Zia
```

The expression on the right is evaluated once. It must be a Dict or
PairList. Each entry in the pattern reads a key and creates a local
binding.

Use `let` for mutable bindings and `const` for bindings that should not
be reassigned:

```zzs
const { name } := den;
name := "Noisy Den";     // invalid: const binding
```

This is not assignment destructuring:

```zzs
let target := { name: "Zia" };
{ name } := target;      // invalid
```

When you need to update an existing value inside a structure, use ordinary
assignment, a path assignment, or a reference.


## 12.3 Shorthand, aliases, and types

The shorthand form uses the local binding name as the key:

```zzs
let { friends } := den;
```

Use `key: local_name` when the local name should differ from the key:

```zzs
let { name: den_name } := den;

say den_name;               // The Quiet Den
```

You can add type checks in both forms:

```zzs
let {
	String name,
	friends: Array den_friends,
} := den;
```

The type goes next to the local binding, not next to the source key.


## 12.4 Defaults in destructuring

An unpacked binding can have a default value:

```zzs
let { location := "unknown", name } := den;

say location;               // unknown
say name;                   // The Quiet Den
```

The default runs only when the key is absent. If a key exists and its
value is `null`, the default is not used.

Defaults in destructuring use `:=`. That is different from the `default`
operator from Chapter 4:

```zzs
let raw_options := {
	light: "bright",
};

let options := raw_options default {
	light: "dim",
	snack: "berries",
	blanket: true,
};

say options{light};          // bright
say options{snack};          // berries
```

Use destructuring defaults when you are binding local names. Use
`default` when you want to merge an option Dict or PairList with fallback
keys. `default` is not a deep merge, so apply it at the level you mean.


## 12.5 String keys and computed keys

Destructuring keys can be identifiers, strings, templates, or computed
expressions:

```zzs
let headers := {
	"content-type": "text/plain",
	"x-den": "quiet",
};

let prefix := "x-";

let {
	"content-type": content_type,
	(prefix _ "den"): den_kind,
} := headers;

say content_type;            // text/plain
say den_kind;                // quiet
```

Computed keys are useful when the key is not a word-like identifier.


## 12.6 Destructuring is shallow

The declaration pattern reads keys from one Dict or PairList. It does not
walk through an entire nested structure for you.

For nested values, destructure in stages:

```zzs
let { settings } := den;
let { light, snack } := settings;

say light;                   // dim
say snack;                   // berries
```

That style is clear when the structure is only one or two levels deep.
For deeper or repeated traversal, path operators are usually better.


## 12.7 The path operators

ZuzuScript has three path operators:

| Operator | Meaning |
|---|---|
| `target @ path` | first match |
| `target @@ path` | all matches |
| `target @? path` | whether there is a match |

By default, string paths use a syntax called `ZZPath`.

```zzs
say den @ "name";                    // The Quiet Den
say den @ "friends/#0/name";         // Zia
say den @@ "friends/*/name";         // [ "Zia", "Zenia", "Zachary", "Fenn" ]
say den @? "friends/#1/mood";        // true
say den @? "friends/#9/mood";        // false
```

Read the operators like this:

- `@`: "give me the first value selected by this path",
- `@@`: "give me every value selected by this path",
- `@?`: "does this path select anything?"

When `@` finds no value, it returns `null`. When `@@` finds no values, it
returns an empty Array. When `@?` finds no values, it returns `false`.


## 12.8 Reading ZZPath strings

A `ZZPath` string is a compact expression for navigating a nested value.
Here are the pieces you will use most often:

| Path piece | Meaning |
|---|---|
| `name` | child named `name` |
| `/name` | child named `name`; also common at the start of a path |
| `*` | all children at this level |
| `#0` | array item at index 0 |
| `#1` | array item at index 1 |
| `[expr]` | filter the current selection using `expr` |

Examples:

```zzs
say den @ "settings/light";           // dim
say den @ "friends/#2/name";          // Zachary
say den @@ "friends/*/tags/*";        // all tags
```

Paths can start with a slash:

```zzs
say den @ "/friends/#0/name";         // Zia
```

In these examples, `"friends/#0/name"` and `"/friends/#0/name"` both start
from the root value supplied on the left of `@`.


## 12.9 Filters

Filters go in square brackets.

```zzs
let sleepy := den @@ "friends/*[mood eq 'sleepy']/name";
say sleepy;                           // [ "Zia" ]
```

A bare field name in a filter checks that the field exists:

```zzs
let tagged := den @@ "friends/*[tags]/name";
say tagged;                           // [ "Zia", "Zenia", "Zachary", "Fenn" ]
```

For value tests, use comparisons and ordinary expression operators:

```zzs
let nappers := den @@ "friends/*[naps > 0]/name";
say nappers;                          // [ "Zia", "Zenia", "Fenn" ]

let busy := den @@ "friends/*[naps = 0 or mood eq 'busy']/name";
say busy;                             // [ "Zachary" ]
```

Keep filters readable. If a path expression becomes hard to understand,
split the problem into a path query followed by ordinary Array code.

Using `'single quotes'` normally gives you a `BinaryString` instead of
a string. For convenience, within a ZZPath expression, single quotes
give you a regular `String`. This avoids needing to do:

```zzs
let sleepy := den @@ "friends/*[mood eq \"sleepy\"]/name";
```


## 12.10 Reusable `ZZPath` objects

The path operators are best for short, local queries. If you will run the
same query many times, compile it into a `ZZPath` object:

```zzs
from std/path/zz import ZZPath;

let friend_names := new ZZPath( path: "friends/*/name" );

say friend_names.query(den);           // [ "Zia", "Zenia", "Zachary", "Fenn" ]
say friend_names.first( den, "none" ); // Zia
say friend_names.exists(den);          // true
```

The main methods are:

- `query(value)` returns all selected values,
- `first(value, fallback?)` returns the first selected value or the
  fallback,
- `exists(value)` returns `true` or `false`.

Compiled paths are also useful when you want to pass a query to another
function:

```zzs
function selected_names ( data, path ) {
	return path.query(data);
}

let sleepy_names := do {
	from std/path/zz import ZZPath;
	new ZZPath( path: "friends/*[mood eq 'sleepy']/name" );
};
say selected_names( den, sleepy_names );
```


## 12.11 Path syntax is lexical

The `@`, `@@`, and `@?` operators do not have one hard-coded string
syntax. They use the active path class for the current lexical scope.

The default is `ZZPath`, but another path class can opt in with its
`use()` method:

```zzs
from std/path/simple import SimplePath;

say den @ "friends/#0/name";          // ZZPath

{
	SimplePath.use();

	say den @ "friends[0].name";      // SimplePath
	say den @@ "friends[*].name";     // SimplePath
}

say den @ "friends/#0/name";          // ZZPath again
```

The switch is lexical. It affects the current block and inner blocks, then
the outer scope's path setting is restored.

You can explicitly choose `ZZPath` in a nested scope:

```zzs
from std/path/zz import ZZPath;

{
	ZZPath.use();
	say den @ "friends/#1/name";
}
```

There is also a `std/path/z` module for ZPath, `std/path/jsonpointer` for
JSON Pointer syntax, and `std/path/kdl` for KDL Query Language.


## 12.12 Path assignment

Path expressions can be assignment targets.

Use `@` when exactly one first match should be updated:

```zzs
den @ "friends/#0/mood" := "asleep";
den @ "friends/#0/naps" ×= 2; // double naps!
( den @ "friends/*/naps" )++; // naps for everybody!
```

Use `@@` when every match should be updated:

```zzs
den @@ "friends/*/mood" := "settled";
den @@ "friends/*/mood" ~= /ed$/ → "ing";
```

The no-match behaviour is intentionally different:

- assigning through `@` throws if there is no match,
- assigning through `@@` updates every match and tolerates no matches,
- `@?` is for existence checks, not `:=` assignment.

This makes `@` good for required structure and `@@` good for optional bulk
updates.


## 12.13 Path references with `\`

The unary `\` operator creates a reference-like function for an assignable
location. Call the function with no arguments to get the current value.
Call it with one argument to set the value.

```zzs
let mood_ref := \( den @ "friends/#0/mood" );

say mood_ref();               // asleep
mood_ref("sleepy");
say den @ "friends/#0/mood";  // sleepy
```

With `@@`, you get an Array of reference-like functions:

```zzs
let nap_refs := \( den @@ "friends/*/naps" );

nap_refs[0]( 4 );
nap_refs[1]( 2 );

say den @@ "friends/*/naps";  // [ 4, 2, 0 ]
```

With `@?`, you get a reference when a match exists, or `null` when it does
not:

```zzs
let maybe_snack_ref := \( den @? "settings/snack" );

if ( maybe_snack_ref ) {
	maybe_snack_ref("biscuits");
}
```

The pseudo-reference operator isn't specific to paths. You can use it
any time you want to give another bit of code access to one of your local
variables.

```zzs
function in_place_capitalize ( ref ) {
	const orig := ref();
	ref( uc orig );
	return true;
}

let name := "zia";
in_place_capitalize( \name );
say name; // "ZIA"
```

Technically what it is doing is creating a getter/setter function as a
closure around the given variable.


## 12.14 Practical shape: unpack, default, path

A common workflow is:

1. apply option defaults at the right Dict level,
2. destructure the top-level names you care about,
3. use paths for repeated or deep access.

```zzs
let report := den default {
	settings: {},
};

let { settings, friends } := report;

let resolved_settings := settings default {
	light: "dim",
	snack: "berries",
	blanket: true,
};

let sleepy_names := report @@ "friends/*[mood eq 'sleepy']/name";
let first_friend := report @ "friends/#0/name";

say resolved_settings{blanket};       // true
say sleepy_names;                     // [ "Zia" ]
say first_friend;                     // Zia
```

Use destructuring for nearby named fields. Use `default` for option-like
Dicts and PairLists. Use `ZZPath` when the shape is nested, repeated, or
conditional.


## 12.15 Chapter recap

You now have the main tools for nested data:

- destructuring declarations bind selected Dict or PairList keys,
- destructuring defaults use `:=`,
- the `default` operator fills missing option keys in Dicts and PairLists,
- `@`, `@@`, and `@?` query nested data through the active path syntax,
- string paths use `ZZPath` by default,
- `ZZPath` supports names, indexes, wildcards, filters, and expressions,
- path syntax can be switched lexically with `PathClass.use()`,
- path assignment updates selected locations,
- and `\` creates reusable getter/setter references to lvalues.

Next up: chains, where expressions start flowing into each other.
