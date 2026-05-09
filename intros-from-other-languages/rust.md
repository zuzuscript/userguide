# Zuzu for Rust Programmers

Rust and ZuzuScript serve different purposes. Rust is built for systems
programming, predictable performance, ownership, and strong static
guarantees. Zuzu is built for practical scripting: automation, text and
data handling, command-line glue, quick integrations, and scripts that
should remain readable as they grow.

Even so, Rust programmers will find some familiar values. Zuzu likes
explicit names, block structure, expression-oriented code, clear module
boundaries, and functions with optional type annotations. It also has a
Rust implementation track, but the primary language experience is much
more dynamic and script-oriented.

Here is a small data filtering task in both languages. It prints names
for enabled accounts.

ZuzuScript:

```zzs
let accounts := [
	{ name: "Ada", enabled: true },
	{ name: "Grace", enabled: false },
	{ name: "Lin", enabled: true },
];

for ( let account in accounts ) {
	if ( account{enabled} ) {
		say account{name};
	}
}
```

Rust:

```rust
struct Account {
    name: &'static str,
    enabled: bool,
}

let accounts = [
    Account { name: "Ada", enabled: true },
    Account { name: "Grace", enabled: false },
    Account { name: "Lin", enabled: true },
];

for account in accounts {
    if account.enabled {
        println!("{}", account.name);
    }
}
```

The high-level shape is similar: create structured data, loop, branch,
and print. Zuzu removes the struct declaration when the data is just a
small dict, and it does not require lifetimes, ownership annotations, or
borrowing decisions. That is the point: Zuzu optimizes for quick scripts,
not static guarantees.

Zuzu function signatures may still look comfortable:

```zzs
function label ( String name, Int count ) -> String {
	return name _ ": " _ count;
}
```

Those annotations are runtime contracts. They do not make Zuzu a static
language, and they do not replace Rust's type checker. They are useful
when a script has grown enough that function boundaries deserve explicit
checks.

Rust habits that need adjustment:

- Variables are introduced with `let`, but assignment uses `:=`.
- Zuzu values are dynamically managed; there is no ownership or borrow
  checker model.
- `=` is numeric equality in expressions, not assignment.
- Type-aware equality is written as `==` or the Unicode alias `≡`.
- String concatenation uses `_`.
- Dict lookup uses `value{field}` rather than `value.field`; dots are
  for methods and object members.
- Errors are thrown with `throw` or `die`, and can be handled with
  `try` forms rather than `Result<T, E>`.

A useful first translation is that Zuzu lets you postpone structure
decisions that Rust quite properly asks you to make early. A payload can
begin life as a dict, a list can hold the values your script actually
has, and a helper can grow type annotations later. This is not a safety
upgrade over Rust. It is a different tradeoff for short-lived or
fast-changing automation code.

One area where Zuzu can be more concise than Rust is exploratory work
with nested, JSON-like data. Rust can do this robustly with `serde`,
typed structs, `Value`, pattern matching, and helper methods, but that is
more machinery than many one-off scripts need. Zuzu's path operators are
designed for that scripting case:

```zzs
let payload := {
	builds: [
		{ id: 1, status: "ok", duration: 12 },
		{ id: 2, status: "fail", duration: 4 },
	],
};

let failed := payload @@ "/builds/*[status == 'fail']/id";
say failed;
```

`@@` returns every matching value, `@` returns the first, and `@?`
checks whether a match exists. For quick inspection of API payloads,
logs, test reports, and configuration data, that is intentionally less
ceremonial than modelling the whole shape first.

For Rust programmers, Zuzu is not a replacement for Rust. It is a
companion for the messy outer layers: scripts, adapters, data extraction,
automation, and glue code. Continue with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md)
to learn how to run Zuzu code and then move through values, functions,
modules, and path queries.
