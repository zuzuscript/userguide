# Zuzu for TypeScript Programmers

TypeScript programmers will find ZuzuScript visually approachable:
braces, `let`, `const`, classes, constructors, methods, arrays, dict-like
object literals, callbacks, `async function`, and `await` all have
familiar shapes. Zuzu is useful for the same practical jobs where people
often reach for TypeScript running on Node: command-line helpers, data
transforms, HTTP integration, build scripts, and automation that has
outgrown shell.

The biggest conceptual difference is the type system. TypeScript types
are static and erased before runtime. Zuzu is dynamically oriented, with
optional runtime type annotations. A Zuzu annotation is closer to a
contract checked while the program runs than to TypeScript's compile-time
analysis.

Here is a small example in both languages. It defines a function that
formats active user names.

ZuzuScript:

```zzs
function active_labels ( Array users ) → Array {
	const labels := [];

	for ( const user in users ) {
		if ( user{active} ) {
			labels.push( user{name} _ " is active" );
		}
	}

	return labels;
}
```

TypeScript:

```ts
type User = {
  name: string;
  active: boolean;
};

function activeLabels(users: User[]): string[] {
  const labels: string[] = [];

  for (const user of users) {
    if (user.active) {
      labels.push(user.name + " is active");
    }
  }

  return labels;
}
```

The outline is similar: function, array, loop, condition, push, return.
Zuzu uses `function name ( Type arg ) → Type`, string concatenation with
`_`, and dict lookup with `user{name}`. It keeps type information close
to the function boundary without requiring interface declarations for
small scripts.

Classes also look familiar at first glance:

```zzs
class Client {
	let base_url;

	method status_url ( path ) {
		return base_url _ path;
	}
}

let client := new Client( base_url: "https://example.test" );
```

The named constructor argument style is common in Zuzu. It makes small
objects readable without a separate options interface.

TypeScript habits that need adjustment:

- Binding and assignment use `:=`; `=` is numeric equality inside
  expressions.
- Type-aware equality is `≡` or the ASCII alias `==`.
- Zuzu annotations are runtime checks, not static types.
- String comparison uses `eq`, `ne`, `lt`, and `gt` for lexical meaning.
- String concatenation uses `_`, not `+`.
- Dict lookup uses `value{key}`; method calls use dots.
- Zuzu modules are imported like `from std/data/json import JSON;`.

A useful first translation is that TypeScript interfaces often become
plain Zuzu dicts until the script proves it needs a named class or a
typed function boundary. You can still document intent with annotations,
but you do not need to model every intermediate payload before touching
it. That is deliberate: Zuzu is optimized for scripts that inspect,
reshape, and emit data quickly while remaining readable.

The place where Zuzu can be much terser than TypeScript is nested data
navigation. TypeScript can model the shape of data well, but extracting
values from real API payloads still often involves optional chaining,
array methods, narrowing, and defensive checks. In Zuzu, path operators
make that query part of the language:

```zzs
let payload := {
	issues: [
		{ id: 1, state: "open", assignee: { login: "ada" } },
		{ id: 2, state: "closed", assignee: null },
	],
};

let logins := payload @@ "/issues/*[state == 'open']/assignee/login";
say logins;
```

`@@` returns all matches. `@` returns the first match, and `@?` answers
whether a path exists. This gives scripts a direct way to express “pull
these fields out of that response” without building a little traversal
framework around the data.

For TypeScript programmers, Zuzu is best understood as a lighter
scripting language with a familiar brace-and-class surface, runtime
contracts, rich operators, and first-class path queries. Continue with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md)
for setup, the REPL, and a guided tour of the core syntax.
