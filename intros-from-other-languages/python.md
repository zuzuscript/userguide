# Zuzu for Python Programmers

Python programmers usually look for three things in a new scripting
language: readable code, good collection handling, and a short path from
idea to running program. ZuzuScript is aimed at the same kind of work:
automation, text cleanup, small command-line tools, data transforms,
HTTP glue, and scripts that can become modules when they grow.

The visual style is different. Python uses indentation as syntax. Zuzu
uses braces, semicolons, and explicit binding with `let name := value`.
But the day-to-day model is familiar: create a few values, loop over
collections, call methods, build dicts and arrays, and print the result.

Here is the same small report in both languages. It keeps active users
and prints their names.

ZuzuScript:

```zzs
const rows := [
	{ name: "Ada", active: true },
	{ name: "Grace", active: false },
	{ name: "Lin", active: true },
];

for ( const row in rows ) {
	if ( row{active} ) {
		say row{name};
	}
}
```

Python:

```python
rows = [
    {"name": "Ada", "active": True},
    {"name": "Grace", "active": False},
    {"name": "Lin", "active": True},
]

for row in rows:
    if row["active"]:
        print(row["name"])
```

The main ideas transfer directly. Zuzu arrays look like Python lists,
Zuzu dicts look like Python dictionaries, booleans are plain values, and
loops are direct. The main syntactic changes are braces around blocks,
`true` and `false` in lowercase, and `row{name}` for dict-style field
lookup.

Python list comprehensions are also easy to translate:

```python
for name in [ x["name"] for x in rows if x["active"] ]:
    print(name)
```

Becomes:

``zzs
for ( let name in rows.grep(fn x → x{active}).map(fn x → x{name}) ) {
	say name;
}
```

Zuzu also has a familiar module story. You import names from modules,
write functions for reusable logic, and use classes when modelling an
object is clearer than passing dicts around. Methods are called with
`object.method()`, and constructors use `new Class(...)`.

The differences are worth learning early:

- Assignment and binding use `:=`, not `=`.
- Numeric equality uses `=`, while type-aware equality uses `≡` or its
  ASCII alias `==`.
- String concatenation uses `_`, not `+`, so you can rely on `+` always
  really being addition.
- Blocks always use braces, so indentation is style rather than syntax.
- Zuzu has separate numeric and string comparison operators. Use `lt`,
  `gt`, `eq`, and `ne` for lexical string comparisons.
- Regular expressions are language syntax, so `text ~ /pattern/` is a
  normal expression.

A useful first translation is that a short Python script often maps to a
short Zuzu script with slightly more punctuation and slightly more
operator choice. Instead of relying on Python's broad `+`, `==`, and
truth model for many jobs, Zuzu asks you to choose string comparison,
numeric comparison, type-aware equality, or concatenation explicitly.
That feels unusual at first, but it can make scripts clearer when data
comes from files, environment variables, and APIs.

Python programmers should also notice that Zuzu has sets and bags as
first-class collection literals. A set stores unique values; a bag stores
counts of repeated values. That makes some counting and membership tasks
more direct than repeatedly reaching for `set`, `Counter`, or helper
imports.

The feature that often feels most uniquely Zuzu-like is path querying
through nested data. In Python, extracting a nested JSON field usually means
indexing, `.get()` calls, comprehensions, or a third-party query library.
In Zuzu, the query can sit directly in the expression:

```zzs
let payload := {
	tickets: [
		{ id: 1, state: "open", labels: [ "bug" ] },
		{ id: 2, state: "closed", labels: [ "docs" ] },
	],
};

let open_ids := payload @@ "/tickets/*[state == 'open']/id";
say open_ids;
```

`@@` returns all matching values. Use `@` when you want the first match,
and `@?` when you only need to know whether a path exists. For scripts
that process API responses, config files, or test results, this can
replace a surprising amount of navigation code.

Think of Zuzu as occupying a similar practical space to Python, but with
Perl-family operators, braces, richer literal collection syntax, and
built-in path queries. To keep going, start with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md),
which covers running scripts, the REPL, values, and the first complete
programs.
