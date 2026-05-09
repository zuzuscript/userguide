# Zuzu for Ruby Programmers

Ruby programmers tend to value expressive code, friendly collection
operations, lightweight objects, and scripts that read close to the
problem being solved. ZuzuScript has a similar practical spirit. It is
aimed at automation, text processing, command-line tools, data
transforms, HTTP integration, and programs that start small but deserve
clean structure as they grow.

The syntax looks less like Ruby at first because Zuzu uses braces,
semicolons, and explicit binding with `let name := value`. But the
everyday style is familiar: arrays and dicts are direct values, methods
live on objects, blocks of behaviour can be passed around, and small
scripts can still be organized with functions and classes.

Here is a small collection transformation in both languages. It selects
enabled accounts and prints labels.

ZuzuScript:

```zzs
let accounts := [
	{ name: "Ada", enabled: true },
	{ name: "Grace", enabled: false },
	{ name: "Lin", enabled: true },
];

let labels := [];

for ( let account in accounts ) {
	if ( account{enabled} ) {
		labels.push( account{name} _ " is enabled" );
	}
}

for ( let label in labels ) {
	say label;
}
```

Ruby:

```ruby
accounts = [
  { name: "Ada", enabled: true },
  { name: "Grace", enabled: false },
  { name: "Lin", enabled: true },
]

labels = accounts
  .select { |account| account[:enabled] }
  .map { |account| "#{account[:name]} is enabled" }

labels.each { |label| puts label }
```

The resemblance is in the collection workflow: select the interesting
items, build labels, then print them. Zuzu uses explicit loops here,
while Ruby uses blocks; both keep the transformation close to the data.
Zuzu uses `say` where Ruby uses `puts`. String concatenation uses `_`,
though template strings are also available in Zuzu for many interpolation
cases.

Zuzu classes are also lightweight enough to feel natural in scripts:

```zzs
class Job {
	let name;

	method label () {
		return "job:" _ name;
	}
}

let job := new Job( name: "deploy" );
say job.label();
```

The named argument constructor style keeps object creation readable
without a lot of setup code.

Ruby habits that need adjustment:

- Blocks are not Ruby blocks. Use `fn value -> expression` or full
  `function` declarations.
- Binding and assignment use `:=`.
- Numeric equality is `=`, while type-aware equality is `==` or the
  Unicode alias `≡`.
- String comparison uses `eq`, `ne`, `lt`, and `gt` for lexical meaning.
- String concatenation uses `_`, not `+`.
- Dict lookup commonly uses `value{key}`, not `value[:key]`.
- `if`, `for`, `while`, functions, and classes use braces.

A useful first translation is that Zuzu collection code is usually more
explicit than Ruby block chains, but it should still stay close to the
data. When a transformation becomes hard to read, the usual Zuzu answer
is the same as the Ruby answer: name the operation. Pull repeated logic
into a function or method, keep the collection code obvious, and let the
script tell the story in small steps.

Zuzu and Ruby both care about regular expressions and text handling.
Zuzu makes regex matching a direct operator:

```zzs
if ( line ~ /^deploy:(ok|fail)$/ ) {
	say "deployment line";
}
```

Where Zuzu becomes especially compact is nested data querying. Ruby can
handle this with `dig`, `select`, `map`, pattern matching, or helper
gems. Zuzu makes the query an operator:

```zzs
let payload := {
	projects: [
		{ name: "core", status: "green" },
		{ name: "docs", status: "red" },
	],
};

let failing := payload @@ "/projects/*[status == 'red']/name";
say failing;
```

`@@` returns all matches. `@` returns the first match, and `@?` checks
whether a match exists. For scripts that consume JSON, YAML-like data,
test reports, or service responses, this is often clearer than stacking
several collection calls just to reach the interesting fields.

For Ruby programmers, Zuzu is a brace-based scripting language with a
similar taste for expressive collections, small objects, regexes, and
readable automation. Continue with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md)
to install Zuzu, run your first script, and learn the core syntax in a
guided order.
