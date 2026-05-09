# Chapter 5: Choose Your Own Adventure: Control Flow

In Chapter 4, we learned how expressions produce values.

Now we answer the next question:

> “Great. But which code runs, when, and how many times?”

That is control flow.

Control flow is where a script starts feeling like an actual program:

- make a decision,
- repeat work while a condition holds,
- iterate over collections,
- skip some steps,
- stop a loop early,
- return from a function as soon as you have the answer.

As in previous chapters, examples are written for the **Perl
implementation** and follow syntax used in language tests and runtime
documentation.


## 5.1 Conditions with `if`, `else if`, and `else`

The most common branch is still the trusty `if`.

```zzs
let cups := 2;

if ( cups >= 3 ) {
  say "Zia is fully operational.";
}
else if ( cups = 2 ) {
  say "Zia can code, but only in cozy mode.";
}
else {
  say "Zia is loading... very slowly.";
}
```

A few practical notes:

- Conditions are wrapped in parentheses: `if ( condition ) { ... }`.
- Blocks use braces.
- `else if` chains are evaluated top to bottom.
- The first matching branch wins.

### Expressions inside conditions

A condition is just an expression context, so assignment and declaration
expressions can appear there.

```zzs
let badge := "";

if ( let naps := 7 ) {
  badge := `naps=${naps}`;
}

if ( badge := badge _ " ✅" ) {
  say badge;
}
```

That style is supported, but use it intentionally. If it hurts
readability, split it into separate lines.

### Postfix conditionals: quick and tidy

ZuzuScript also supports postfix forms for short one-liners:

```zzs
let points := 0;

points += 2 if true;
points += 100 unless true;
```

Postfix forms are great for small “do this only when…” statements.
For larger logic, prefer full `if` blocks.


## 5.2 Truthiness rules (what counts as true?)

When ZuzuScript evaluates a condition, it applies truthiness coercion.
In the Perl runtime, the practical rules are:

- `null` is falsy,
- numeric `0` is falsy,
- `""`, `"0"`, and `"0.0"` are falsy,
- empty collections are falsy,
- non-empty collections are truthy,
- objects/functions/classes are truthy by default unless they define
  custom boolean behaviour.

```zzs
let inbox := [ ];
if ( inbox ) {
  say "Messages!";
}
else {
  say "Inbox is quiet.";
}

let queue := [ "lint" ];
if ( queue ) {
  say "Work exists (regrettably).";
}
```

Tip: even though truthiness is convenient, explicit comparisons are
sometimes clearer in production code (for example, `items.length() > 0`
when intent matters more than brevity).


## 5.3 Repeating work with `while`

Use `while` when you want to repeat as long as a condition stays truthy.

```zzs
let i := 0;
let brewed := 0;

while ( i < 3 ) {
  brewed += 1;
  i++;
}

say brewed;  # 3
```

### `next` and `last` in loops

Inside `while` and `for`, you can control loop flow with:

- `next` -> skip to the next iteration,
- `last` -> exit the loop immediately.

```zzs
let x := 0;
let total := 0;

while ( x < 8 ) {
  x++;

  if ( x mod 2 = 0 ) {
    next;
  }

  if ( x > 5 ) {
    last;
  }

  total += x;
}

say total;  # 1 + 3 + 5 = 9
```

A reliable mental model:

- use `next` when one iteration should be skipped,
- use `last` when the whole loop should stop.


## 5.4 Iteration with `for`

Use `for` when you already have a collection (or iterable expression)
and want to visit each item.

```zzs
let sum := 0;

for ( let n in [ 1, 2, 3 ] ) {
  sum := sum + n;
}

say sum;  # 6
```

You can declare the loop variable as `let` or `const`:

```zzs
let total := 0;

for ( const item in [ 4, 5 ] ) {
  total += item;
}
```

`const` is nice when you want to guarantee no reassignment of the loop
variable inside the body.

### `for ... else` for empty-iteration handling

A handy feature in the Perl implementation: `for` supports an `else`
block that runs when the loop body does not run.

```zzs
let fallback := "";

for ( let task in [ ] ) {
  fallback := "had-work";
}
else {
  fallback := "no-work";
}

say fallback;  # "no-work"
```

This can remove an extra pre-check and keep logic local.


## 5.5 Iterating arrays, sets, bags, dicts, and pairlists

Control flow becomes much more useful when paired with collections.

### Arrays

Arrays preserve order and duplicates, so `for` sees each element in
sequence:

```zzs
for ( let snack in [ "coffee", "cookie", "coffee" ] ) {
  say snack;
}
```

### Sets

Sets represent membership. You can still iterate them directly:

```zzs
let seen := 0;

for ( let x in << 1, 2 >> ) {
  seen += x;
}
```

Remember: sets deduplicate values, so there are no repeated members.

### Bags (multisets)

Bags preserve multiplicity, so repeated values may be visited multiple
times:

```zzs
let count := 0;

for ( let token in <<< "a", "a", "b" >>> ) {
  count += 1;
}

say count;  # 3
```

### Dicts

Dicts are typically iterated using `.enumerate()`:

```zzs
let entries := 0;

for ( let item in { a: 1, b: 2 }.enumerate() ) {
  entries += 1;
}

say entries;  # 2
```

### PairLists

PairLists preserve pair order and allow duplicate keys:

```zzs
let tags := 0;

for ( let p in {{ tag: "perl", tag: "zuzu" }}.enumerate() ) {
  tags += 1;
}

say tags;  # 2
```

In Chapter 8 we will do a deeper collection strategy tour. For now,
focus on this: `for` works cleanly across the major collection types.

### Function iterators

`for` can also iterate a function value directly (no `()` call). The
function is called once per item and should throw
`ExhaustedException` when no items remain.

```zzs
let i := 4;

function countdown () {
  throw new ExhaustedException() if i == 0;
  return i--;
}

for ( let n in countdown ) {
  say n;
}
```

If you include `()`, the function is called immediately and its return
value is iterated instead:

```zzs
function make_list () {
  return [ 1 ... 4 ].reverse();
}

for ( let n in make_list() ) {
  say n;
}
```

### Custom objects as loop sources

Objects can participate in `for` if they expose either:

- `to_Iterator()` returning a function iterator, or
- `to_Array()` returning an array.

When both exist, `to_Iterator()` is preferred.

```zzs
class MyThing {
  let i := 4;

  method to_Iterator () {
    let x := i;
    return function () {
      throw new ExhaustedException() if x == 0;
      return x--;
    };
  }
}

for ( let n in new MyThing() ) {
  say n;
}
```


## 5.6 `switch`, `case`, `default`, and `continue`

When you have one subject value and multiple branches, `switch` keeps
logic flatter than a long `else if` chain.

```zzs
let mood := "sleepy";
let plan := "";

switch ( mood: eq ) {
  case "focused": plan := "write module";
  case "sleepy", "cozy": plan := "drink coffee";
  default: plan := "take a walk";
}

say plan;
```

### Comparator form

`switch` can specify a comparator after `:`.

- `switch ( value )` -> default matching behaviour,
- `switch ( value: eq )` -> lexical/string comparison,
- `switch ( value: ~ )` -> regex/operator style matching.

```zzs
let found := "";

switch ( "xylophone": ~ ) {
  case /x/: found := found _ "x";
  continue;
  case /y/: found := found _ "y";
}

say found;  # "xy"
```

### Fallthrough is explicit with `continue`

Unlike C-style switch semantics, cases do **not** fall through by
default. If you want to continue into the next case/default, use the
`continue` keyword explicitly.

```zzs
let picked := "";

switch ( "bar": eq ) {
  case "foo": picked := "foo";
  case "bar", "baz": picked := "matched";
  continue;
  default: picked := picked _ "-default";
}

say picked;  # "matched-default"
```

That explicitness is great for readability: fallthrough is always a
conscious choice.


## 5.7 Early exit with `return`

Loops control repetition. `return` controls function exit.

Use early returns to keep logic shallow and readable.

```zzs
function brew_status ( cups ) {
  if ( cups <= 0 ) {
    return "no-coffee";
  }

  if ( cups = 1 ) {
    return "barely-awake";
  }

  return "ready";
}
```

This pattern avoids deeply nested conditionals and makes each path
obvious.

A practical guideline:

- return early for edge cases,
- let the “main happy path” read straight through.


## 5.8 Choosing the right flow tool

When writing real scripts, choose control-flow constructs by shape:

- **`if` / `else if` / `else`** for one-off branching decisions,
- **`switch`** for one subject with many named cases,
- **`while`** for condition-driven repetition,
- **`for`** for collection-driven repetition,
- **`next` / `last`** for fine-grained loop control,
- **`return`** for function-level early exits.

If the code starts feeling twisty, split work into helper functions.
Readable control flow is one of the biggest quality multipliers in
scripting code.


## 5.9 Mini walkthrough: sleepy raccoon triage loop

Let’s finish with a compact example that combines most chapter topics.

```zzs
function choose_task ( tasks ) {
  for ( let task in tasks ) {
    if ( not task{ready} ) {
      next;
    }

    if ( task{priority} >= 9 ) {
      return task{name};
    }
  }

  return "coffee-break";
}

let tasks := [
  { name: "lint", ready: true,  priority: 4 },
  { name: "ship", ready: false, priority: 10 },
  { name: "docs", ready: true,  priority: 9 },
];

say choose_task( tasks );
```

What happens:

1. iterate each task,
2. skip not-ready entries with `next`,
3. return early on first high-priority ready task,
4. otherwise return a default fallback.

This is the kind of shape you will use constantly in tooling and
automation scripts.


## 5.10 What comes next

You now have the language tools to control *when* work happens.

In Chapter 6, we will zoom in on functions themselves:

- parameters (required/optional/default/named patterns),
- return types,
- closures,
- anonymous functions,
- recursion.

In other words: we are about to make your control flow reusable.
Zia approves (between naps).
