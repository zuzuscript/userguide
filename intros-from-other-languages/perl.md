# Zuzu for Perl Programmers

If you are comfortable in Perl, ZuzuScript should feel familiar before it
feels new. It is a practical scripting language with a strong bias
towards text processing, data wrangling, command-line tools, modules,
regular expressions, and small programs that can grow into larger ones.
The primary implementation is written in Perl, and it inherited a lot of
Perl's semantics "by default". Zuzu makes simple jobs easy, keeps
useful operations close to the language, and let scripts stay readable.
Zuzu even allows you to embed pod in your scripts and modules.

But there are differences too. You will quickly notice that Zuzu avoids
Perl's sigil system. Variables are introduced with `let` or `const`, and
assignment uses `:=`. Expression equality is not assignment. String
concatenation uses `_`, regular expression matching uses `~`, and method
calls use a dot.

Here is a small text-processing task in both languages. It prints enabled
entries from simple `name:status` lines.

ZuzuScript:

```zzs
const lines := [
	"ada:enabled",
	"grace:disabled",
	"lin:enabled",
];

for ( const line in lines ) {
	say line if line ~ /:enabled$/;
}
```

Perl:

```perl
use feature 'say';

my @lines = (
	"ada:enabled",
	"grace:disabled",
	"lin:enabled",
);

for my $line ( @lines ) {
	say $line if $line =~ /:enabled$/;
}
```

The shape is very close: a list, a loop, a regular expression, and output.
Zuzu keeps the same quick scripting rhythm, but removes `$line`,
`@lines`, and other sigil-driven context cues. Instead, arrays are just
values, and the `for` loop says explicitly which value is being bound.
For regular expressions, `text ~ /pattern/` is the direct match operator.

Some gotchas matter early:

- `:=` is binding or assignment. Numeric equality is `=`, and type-aware
  equality is `≡` or the ASCII alias `==`.
- Zuzu uses `_` for string concatenation, not `.`.
- Comments are `//` and `/* ... */`, not `#`.
- Neither double quotes nor single quotes interpolate. Instead, ZuzuScript
  supports JavaScript-like backtick templates.
- Double quotes give you Unicode strings; single quotes for binary.

But there are also useful similarities in the standard scripting vocabulary.
Zuzu has `say` and `die`, regex literals, lexical imports, objects,
classes, methods, runtime type annotations, and rich collection support.
It also has dedicated string comparison operators such as `eq`, `ne`,
`lt`, and `gt`, so Perl programmers do not have to unlearn the habit of
choosing numeric and string comparison deliberately.

Where Zuzu gets especially concise is nested data access. Perl can do
this well, but deep traversals usually involve chained dereferences,
helper modules, or loops. Especially if you wish to avoid autovivification.
In Zuzu, path queries are a built-in operator:

```zzs
let payload := {
	users: [
		{ name: "Ada", role: "admin" },
		{ name: "Lin", role: "user" },
	],
};

say payload @ "/users/*[role ≡ 'admin']/name";
```

The `@` operator returns the first match. `@@` returns all matches, and
`@?` checks whether a match exists. That gives Zuzu a compact way to work
with JSON-like structures without turning the script into navigation
boilerplate.

If Perl is your home language, think of Zuzu as a cleaner, more regular
scripting surface with familiar instincts: regexes, small tools, useful
operators, and a standard library aimed at real jobs. From here, continue
with [Chapter 1 of the main guide](../01-hello-world-and-everything-after.md)
for installation, the REPL, and the first complete programs.
