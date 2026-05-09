# Zuzu for Shell Scripters

Shell is excellent for launching programs, wiring commands together, and
doing small jobs directly from the terminal. It becomes harder to keep
tidy when the script grows real data structures, nested conditionals,
non-trivial string handling, JSON processing, or reusable functions.
ZuzuScript is designed for that next step: still script-shaped, still
good for automation, but with arrays, dicts, classes, modules, regexes,
exceptions, and path queries in the language.

The mental model is familiar. A Zuzu script can read arguments, inspect
environment variables, call processes through the standard library, loop
over values, and print output. The syntax is more like a general-purpose
language than POSIX shell: braces for blocks, `let` for variables, and
semicolons as statement terminators.

Here is a small filtering task in both languages. It prints enabled
service entries from `name:state` lines.

ZuzuScript:

```zzs
let lines := [
	"api:enabled",
	"worker:disabled",
	"cron:enabled",
];

for ( let line in lines ) {
	if ( line ~ /:enabled$/ ) {
		say line;
	}
}
```

Shell:

```sh
printf '%s\n' \
  'api:enabled' \
  'worker:disabled' \
  'cron:enabled' |
while IFS=: read -r name state; do
  if [ "$state" = enabled ]; then
    printf '%s:%s\n' "$name" "$state"
  fi
done
```

Both scripts are doing the same job: loop over lines, check a suffix, and
print selected entries. Zuzu gives you an actual array, a regular
expression operator, lexical variables, and normal structured control
flow. You do not have to worry about word splitting, quoting every
expansion, subshell behaviour in pipelines, or whether a value contains
spaces.

Shell habits that need adjustment:

- Variables are introduced with `let name := value`; there is no `$name`
  expansion for ordinary variable reads.
- Strings are ordinary values, not command-line words waiting to be
  split.
- Assignment uses `:=`, and numeric equality in expressions is `=`.
- `if`, `for`, and functions use braces, not `then`, `fi`, `do`, and
  `done`.
- External commands are not the centre of the language. Use standard
  modules for files, JSON, HTTP, paths, and processes where appropriate.
- Comments are `//` and `/* ... */`, not `#`.

A useful first translation is to stop treating every value as text until
proved otherwise. In Zuzu, a boolean can stay a boolean, a list can stay
a list, and a dict can carry named fields without being flattened into
lines. You can still call external programs when that is the right tool,
but you do not need a pipeline just to keep state or branch on structured
data.

Zuzu also gives you data structures that shell does not naturally have.
A small configuration value can be a dict:

```zzs
let config := {
	name: "backup",
	paths: [ "/etc/app", "/var/lib/app" ],
	enabled: true,
};

if ( config{enabled} ) {
	say "running " _ config{name};
}
```

That is much easier to extend than parallel shell arrays or ad-hoc text
formats.

The especially concise Zuzu feature for shell users is querying nested
data. In shell, JSON normally means calling `jq` or writing fragile text
processing. Zuzu brings that style of query into ordinary code:

```zzs
let report := {
	services: [
		{ name: "api", state: "ok" },
		{ name: "worker", state: "fail" },
	],
};

let failed := report @@ "/services/*[state == 'fail']/name";
say failed;
```

`@@` returns all matches. `@` returns the first match, and `@?` checks
whether a match exists. This gives automation scripts a direct way to
inspect structured data without dropping out to a separate query tool
for every operation.

For shell scripters, Zuzu is the language to reach for when the script is
still automation, but the data and control flow deserve stronger tools.
Continue with
[Chapter 1 of the main guide](../01-hello-world-and-everything-after.md)
to install Zuzu, run scripts, try the REPL, and learn the core syntax.
