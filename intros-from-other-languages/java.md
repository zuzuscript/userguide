# Zuzu for Java Programmers

Java programmers usually come to scripting languages looking for less
ceremony: fewer files, less scaffolding, and faster feedback for small
jobs. ZuzuScript is designed for that space. It is useful for
automation, data transforms, text processing, command-line helpers, HTTP
glue, and small programs where creating a full Java project would feel
too heavy.

At the same time, Zuzu has familiar object-oriented pieces. It supports
classes, methods, constructors, modules, explicit function boundaries,
and optional runtime type annotations. Blocks use braces, method calls
use dots, and `new Class(...)` constructs objects.

Here is a small filtering task in both languages. It prints active user
names.

ZuzuScript:

```zzs
let users := [
	{ name: "Ada", active: true },
	{ name: "Grace", active: false },
	{ name: "Lin", active: true },
];

for ( let user in users ) {
	if ( user{active} ) {
		say user{name};
	}
}
```

Java:

```java
record User(String name, boolean active) {}

var users = List.of(
    new User("Ada", true),
    new User("Grace", false),
    new User("Lin", true)
);

for (var user : users) {
    if (user.active()) {
        System.out.println(user.name());
    }
}
```

The logic is the same: make records, loop, branch, and print. Zuzu keeps
the record shape inline as dict values and uses `say` for output. That
makes the one-file script version shorter, while still allowing you to
introduce classes when an object deserves a name.

A Zuzu class is lightweight:

```zzs
class Counter {
	let count := 0;

	method increment () {
		return ++count;
	}
}

let counter := new Counter();
say counter.increment();
```

That should feel structurally familiar, but it is not Java's static type
system. Zuzu annotations are runtime contracts, not compile-time
generics, interfaces, or overload resolution.

Java habits that need adjustment:

- There is no required class wrapper for every script.
- Binding and assignment use `:=`.
- `=` is numeric equality inside expressions.
- Type-aware equality uses `≡` or the ASCII alias `==`; it is not Java
  reference comparison.
- Strings concatenate with `_`, not `+`.
- Dict lookup uses `value{field}`; method calls still use dots.
- Imports look like `from std/data/json import JSON;`, not package
  imports.

A useful first translation is that Java records, small DTOs, and local
helper classes often become Zuzu dicts and functions in the first draft.
When a concept stabilizes, you can still give it a class. This lets you
start with the shape of the problem rather than with project structure,
while retaining enough object-oriented vocabulary to organize larger
scripts.

Zuzu also has scripting features that Java normally handles through
libraries or more verbose APIs: regex literals, concise collection
literals, sets and bags, process and file helpers, and direct path
queries for nested data.

That last feature is where Zuzu can be especially elegant. In Java,
working with arbitrary JSON often means choosing a JSON library, mapping
classes, walking `JsonNode`, handling missing fields, and writing loops
or streams. In Zuzu, a query can be a normal expression:

```zzs
let response := {
	orders: [
		{ id: 100, total: 42, paid: true },
		{ id: 101, total: 19, paid: false },
	],
};

let unpaid_ids := response @@ "/orders/*[!paid]/id";
say unpaid_ids;
```

`@@` returns all matching values. `@` returns the first match, and `@?`
checks whether a match exists. That lets short scripts express data
selection without building a small object model first.

For Java programmers, Zuzu is best treated as a compact scripting
language with enough object structure to stay organized, but much less
ceremony for everyday automation. Continue with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md)
for installation, running scripts, the REPL, values, functions, and the
rest of the language tour.
