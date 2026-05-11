# Zuzu for JavaScript Programmers

JavaScript programmers will recognize much of ZuzuScript's surface
shape. It uses braces for blocks, `let` and `const` for bindings, arrays
with `[ ... ]`, object-like dict literals with `{ ... }`, method calls
with dots, callbacks, classes, constructors, and `async`/`await`. If you
are used to writing Node scripts for automation, data cleanup, HTTP
calls, or small command-line tools, Zuzu is aimed at a nearby workflow.

Zuzu is not JavaScript with another runtime. It is a scripting language
with its own operator choices, module system, collection types, and
runtime semantics. But the first read should not feel alien.

Here is a tiny transformation in both languages. It builds labels for
active users.

ZuzuScript:

```zzs
const users := [
	{ name: "Ada", active: true },
	{ name: "Grace", active: false },
	{ name: "Lin", active: true },
];

for ( const user in users ) {
	if ( user{active} ) {
		say user{name} _ " is active";
	}
}
```

JavaScript:

```js
const users = [
  { name: "Ada", active: true },
  { name: "Grace", active: false },
  { name: "Lin", active: true },
];

for (const user of users) {
  if (user.active) {
    console.log(user.name + " is active");
  }
}
```

The resemblance is clear: array of records, loop, condition, field
access, and output. Zuzu uses `say` instead of `console.log`, `_` instead
of `+` for string concatenation, and `user{name}` for dict lookup.
Bindings use `:=`, so `let users := ...` is the usual form.

Several JavaScript habits transfer well. Zuzu code is expression-heavy.
Objects can have methods. Functions can be passed around. Arrays have
collection methods. `async function` and `await` exist, though the host
runtime determines which asynchronous standard-library features are
available.

Some gotchas are important:

- `=` is numeric equality in expressions. It is not assignment.
- Binding and assignment use `:=`.
- Type-aware equality is written with `≡` or the ASCII alias `==`; do
  not assume JavaScript's loose `==`.
- String comparison uses words such as `eq`, `ne`, `lt`, and `gt`.
- String concatenation is `_`, not `+`.
- Dict fields are read with `{}` syntax, as in `user{name}`.

A useful first translation is that Zuzu separates jobs JavaScript often
loads onto the same operator. Indead of `+` acting as both addition and
string concatenation, and needing to guess from context what it means,
in ZuzuScript it is always numeric addition. There are different operators
for equality depending on if you want a numeric, string, or type-aware
comparison. Dict lookup is visibly different from method calls. This
makes Zuzu a little more explicit, especially in scripts that mix
command-line strings, JSON, numbers, booleans, and nulls. The result is
still compact, but fewer conversions are hidden in familiar punctuation.

Zuzu also has collection types that JavaScript does not make as central.
Sets and bags have literal syntax, and operators such as union and
intersection are part of the language. Regular expressions are direct
expression syntax too:

```zzs
if ( line ~ /^status=(ok|warn|fail)$/ ) {
	say "status found";
}
```

Where Zuzu becomes especially concise is JSONPath-like data querying.
JavaScript often reaches for optional chaining, `map`, `filter`, and
small helper functions to traverse nested structures. Zuzu has path
operators:

```zzs
let payload := {
	releases: [
		{ version: "1.0", stable: true },
		{ version: "1.1", stable: false },
	],
};

let stable_versions := payload @@ "/releases/*[stable]/version";
say stable_versions;
```

`@@` returns every match. `@` returns the first match, and `@?` checks
for existence. This is useful when a script spends its time reading API
responses, package metadata, or configuration blobs.

Chaining is still possible:

```zzs
say payload{releases}.grep(fn r → r{stable}).map(fn r → r{release});
```

But paths hide a lot of the "plumbing".

For a JavaScript programmer, the best mental model is: familiar brace syntax
and object-oriented scripting, but with Perl-style text processing power,
explicit comparison families, and built-in nested-data queries. Continue with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md) to
install Zuzu, run files, try the REPL, and learn the core syntax in order.
