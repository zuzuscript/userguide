# Zuzu for PHP Programmers

PHP programmers will find ZuzuScript familiar in several practical ways.
Both languages are comfortable with scripts, arrays or dict-like data,
strings, regular expressions, web-adjacent work, and programs that start
small before growing into modules and classes. Zuzu is not a web-first
language in the same way PHP historically has been, but it is very much a
language for useful glue: command-line tools, data transforms,
automation, HTTP calls, text processing, and small services.

The syntax is cleaner and less sigil-heavy. Zuzu variables do not start
with `$`; they are introduced with `let` or `const`. Assignment uses
`:=`, arrays use `[ ... ]`, dicts use `{ ... }`, and output commonly uses
`say`.

Here is a small data filtering task in both languages. It prints the
names of paid orders.

ZuzuScript:

```zzs
const orders := [
	{ id: 100, customer: "Ada", paid: true },
	{ id: 101, customer: "Grace", paid: false },
	{ id: 102, customer: "Lin", paid: true },
];

for ( const order in orders ) {
	if ( order{paid} ) {
		say order{customer};
	}
}
```

PHP:

```php
<?php

$orders = [
    [ "id" => 100, "customer" => "Ada", "paid" => true ],
    [ "id" => 101, "customer" => "Grace", "paid" => false ],
    [ "id" => 102, "customer" => "Lin", "paid" => true ],
];

foreach ( $orders as $order ) {
    if ( $order["paid"] ) {
        echo $order["customer"], "\n";
    }
}
```

The structure is similar: array of records, loop, condition, field
lookup, output. Zuzu makes the records dict values with `key: value`
syntax, uses `for ( myvar in mylist )`, and reads fields with
`order{customer}`.

PHP programmers will also recognise classes and methods:

```zzs
class Formatter {
	let prefix := "#";
	method order_label ( order ) {
		return prefix _ order{id} _ " " _ order{customer};
	}
}

let formatter := new Formatter( prefix: ">>" );
say formatter.order_label( orders[0] );
```

Zuzu's class syntax is lighter, and named constructor arguments are
used when creating objects with initial fields.

Important gotchas:

- No `$` sigil is used for ordinary variables.
- Binding and assignment use `:=`.
- Numeric equality in expressions is `=`, while type-aware equality uses
  `≡` or the ASCII alias `==`.
- String concatenation uses `_`, not `.`.
- String comparison uses words such as `eq`, `ne`, `lt`, and `gt`.
- Dict lookup uses `value{key}`, not `$value["key"]`.
- Comments use `//` and `/* ... */`, which should feel familiar, but
  not `#`.

A useful first translation is that many small PHP arrays become Zuzu
dicts with lighter field access. You do not need to choose between a
loose associative array and a full class immediately. Start with the
data shape the script receives, add functions around repeated behaviour,
and introduce classes only when the concept has become stable enough to
deserve one.

Zuzu has first-class regular expression syntax:

```zzs
if ( line ~ /^order:(\d+)$/ ) {
	say "order line " _ line;
}
```

It also has path operators, which are often more concise than PHP code
for nested arrays. PHP can traverse nested decoded JSON with array
indexes, loops, `array_filter`, `array_map`, or helper libraries. Zuzu
can query the data directly:

```zzs
const response := {
	orders: [
		{ id: 100, paid: true },
		{ id: 101, paid: false },
	],
};

const unpaid := response @@ "/orders/*[!paid]/id";
say unpaid;
```

`@@` returns all matching values. `@` returns the first match, and `@?`
checks whether a path exists. For scripts that consume API responses or
configuration data, this keeps the interesting operation visible.

For PHP programmers, Zuzu offers a familiar scripting and object model
without sigils, with more deliberate operators and concise nested-data
queries. Continue with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md)
for installation, running scripts, the REPL, and the first full tour of
the language.
