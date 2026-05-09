# Chapter 3: Names Have Power: Variables & Binding

In Chapter 2, we looked at values: numbers, strings, arrays, dicts,
sets, bags, and how they behave.

Now we zoom in on the thing that makes those values useful in real
programs:

- giving values names,
- controlling where those names are visible,
- deciding whether those names can be reassigned,
- understanding how long bindings live.

This chapter is written for the **Perl implementation** of ZuzuScript,
which is the canonical runtime used throughout this guide.


## 3.1 Values are not names (and this distinction matters)

A value and a variable are related, but not the same thing.

- A **value** is data (like `42`, `"Zia"`, `[ 1, 2, 3 ]`).
- A **binding** is a name associated with that value.

If this sounds academic, here is the practical payoff:

- you can have multiple names pointing at related data,
- you can rebind some names but not others,
- a value can outlive the block where a name was first introduced
  (for example via closures),
- mutating a value is different from rebinding a name.

That final bullet is a very common beginner stumble, so we will keep
returning to it.


## 3.2 Declaring variables with `let`

Use `let` to declare a mutable binding.

```zzs
let cups := 2;
let mascot := "Zia";
let tasks := [ "lint", "test" ];
```

You can also declare first, then assign later:

```zzs
let report;
report := "nightly-ok";
```

This mirrors syntax used across the Perl runtime tests and module docs.
In particular, declarations use `:=`, while later reassignment uses
`:=` as well.

### `:=` is required for declaration assignment

If you come from languages that use `=` for declaration, this is worth
cementing early:

- declaration + initial value: `let x := 1;`
- reassignment of an existing mutable binding: `x := 2;`

```zzs
let score := 5;
score := 9;
score += 1;
```

That distinction keeps parser behaviour predictable and makes it harder
to accidentally write ambiguous declaration forms.

### Typed declarations

You can attach a type at declaration time:

```zzs
let Number total;
total := 9;
total /= 2;

let Any anything := [ 1, 2, 3 ];
```

Type annotations here constrain what can be assigned to the binding.
We will go deeper into typed parameters and return values in Chapter 6,
but variable-level typing is already useful for making intent explicit.


## 3.3 Declaring constants with `const`

Use `const` when a binding should not be reassigned.

```zzs
const language := "ZuzuScript";
const mascot := "Zia";
```

Later reassignment is rejected:

```zzs
const naps := 2;
# naps := 3;   # compile error
```

A nice practical rule:

- default to `const` for names that should stay stable,
- use `let` when the variable is truly state that evolves.

This yields clearer code and fewer "who changed this?" bugs.

### Weak storage with `but weak`

`but weak` marks a binding as weak storage. This is intended for
non-owning references such as parent pointers, owner back-references, and
observer registrations.

```zzs
let parent but weak;
parent := owner;           // stored weakly

let current := owner but weak;
current := replacement;    // also stored weakly

const root := owner but weak;
```

When `but weak` is part of a declaration, the binding is inherently
weak. Any later reference-capable value assigned to that binding is
stored weakly. In `let current := owner but weak`, the declaration-level
meaning wins; `current` remains weak storage after the initializer.

`const root := owner but weak` is valid. `const` still means the binding
cannot be rebound, but the weak referent may later disappear. When a
weak referent has gone away, reading the binding returns `null`.

Scalar Zuzu values are stored normally even in weak storage. This
includes `null`, booleans, numbers, strings, and binary strings. That
rule is based on ZuzuScript semantics, not on how a backend happens to
represent those values internally.

For a single weak write to ordinary strong storage, put `but weak` on a
simple `:=` assignment:

```zzs
let cached := null;
cached := owner but weak;  // only this write is weak
cached := replacement;     // later plain writes are strong
```

Weak writes are only valid for simple `:=` assignments. Compound
assignments such as `+=`, `_=` and `?:=` compute a new value and cannot
use `but weak`. `x but weak` is not a general expression and cannot be
passed as an argument marker.

### Parameters are effectively const bindings

Function parameters in the current implementation behave like const
bindings inside the function body. Attempting to assign or `++` them is
a compile-time error.

```zzs
function bump ( Number x ) {
  # x += 1;    # compile error: cannot assign to const
  return x + 1;
}
```

That design nudges function bodies toward explicit local state:

```zzs
function bump ( Number x ) {
  let y := x;
  y += 1;
  return y;
}
```

It reads a bit more verbosely, but the intent is crystal clear.


## 3.4 Mutation vs rebinding: same data, different action

Let’s split two operations that are often conflated.

### Rebinding a name

```zzs
let mood := "sleepy";
mood := "focused";
```

The binding `mood` now refers to a different value.

### Mutating a value through a binding

```zzs
let snacks := [ "cookie" ];
snacks.push("tea");
```

The binding name `snacks` did not change. The array value changed.

These are different concepts, and knowing that difference will save you
from many tricky bugs in larger programs.

### Important: `const` protects the binding, not necessarily value internals

`const` means "this name cannot be rebound". It does not automatically
turn every referenced object into deeply immutable data.

```zzs
const queue := [ "lint" ];
queue.push("test");   # mutates array value
# queue := [ "ship" ]; # forbidden rebinding
```

If you need deep immutability, that is a separate design choice at the
value/API level rather than a property of `const` alone.


## 3.5 Assignment operators you will actually use

Beyond `:=`, ZuzuScript supports compound assignment forms used heavily
in tests and real scripts:

- `+=`, `-=`, `*=`, `/=`, `**=`
- Unicode equivalents like `×=` and `÷=`
- string concat assignment `_=`
- null-coalescing assignment `?:=`

```zzs
let x := 1;
x += 4;      # 5

x := 3;
x ×= 4;      # 12

let s := "ab";
s _= "cd";   # "abcd"

let maybe;
maybe ?:= 4; # assigns because undefined/null-like
maybe ?:= 9; # leaves current value as-is
```

`?:=` is especially handy for defaulting configuration values without
overwriting explicit input.


## 3.6 Scope: where a name exists

In day-to-day terms, scope answers:

> "From where in the program can I use this name?"

ZuzuScript in the Perl implementation uses lexical scoping, with nested
blocks introducing nested scopes.

### Top-level scope

Names declared at script top-level are visible later in that file:

```zzs
let project := "acorn-index";

function banner () {
  return `Project: ${project}`;
}

say banner();
```

### Block scope

A new block (`{ ... }`) introduces a child lexical scope.

```zzs
let mood := "sleepy";

if ( true ) {
  let snack := "cookie";
  say snack;
}

# say snack;   # out of scope here
```

### Function scope

Function bodies are their own lexical scopes. Parameters and locals live
there.

```zzs
function greet ( name ) {
  let prefix := "Hello";
  return `${prefix}, ${name}`;
}

# prefix is not visible here
```

### Loop scope

Loop variables declared in `for ( let item in ... )` or
`for ( const item in ... )` are block-scoped to the loop.

```zzs
let item := 100;
let total := 0;

for ( const item in [ 2, 3 ] ) {
  total += item;
}

say item + total;   # outer item remains 100
```

This behaviour is covered in integration tests and is extremely useful:
you can reuse short names like `item` locally without leaking them.


## 3.7 Shadowing: same name, closer binding wins

**Shadowing** means declaring a new binding with the same name in an
inner scope.

```zzs
let phase := "root";

if ( true ) {
  let phase := "child";
  say phase;      # child
}

say phase;        # root
```

The inner `phase` temporarily hides the outer one inside that block.
When the block exits, the outer binding is visible again.

Shadowing is not "evil"; it is a tool. The trick is to use it where it
improves clarity.

### Good shadowing

- small, local blocks,
- temporary transformed versions of a value,
- loop-local names like `item`, `row`, `entry`.

### Risky shadowing

- long functions with many nested blocks,
- shadowing names with significantly different meaning,
- shadowing that hides mutable state updates unexpectedly.

If you feel confused reading your own code, rename. Readability wins.


## 3.8 Imports also create bindings

An import is not magical; it creates names in the current lexical scope.

```zzs
from extras/math import add_x, x;
```

Those names follow the same scope and conflict rules as other bindings.
For example:

- imported names do not leak outside the scope where imported,
- import conflicts are compile-time errors,
- wildcard imports have extra restrictions in certain conditional forms.

This unified model is nice: "a name is a name," whether introduced by
`let`, `const`, function declaration, class declaration, or import.


## 3.9 Closures and lifetime: values can outlive local blocks

Here is where bindings become more than syntax trivia.

Consider:

```zzs
function make_adder (x) {
  return fn y -> x + y;
}

let plus_five := make_adder(5);
say plus_five(10);   # 15
```

The returned lambda still "remembers" `x` from the enclosing function.
That is lexical capture (a closure).

So even though `make_adder` has returned, the captured value remains
available for that function object.

This is why scope and lifetime are related but not identical:

- scope is "where can a name be referenced in source code?"
- lifetime is "how long does the underlying value stay alive at
  runtime?"

If closures, objects, or other reachable structures still reference a
value, it stays alive.


## 3.10 High-level garbage collection mental model (Perl runtime)

At beginner/intermediate level, you do not need collector internals to
write good ZuzuScript. A reliable mental model is enough:

1. Values remain alive while reachable from live bindings/objects.
2. Values that become unreachable are eventually reclaimed.
3. You should design for clarity first; micro-managing lifetime is
   rarely needed in normal scripts.

In the Perl implementation, runtime values are hosted by Perl objects,
so memory behaviour follows the implementation runtime's reachability and
cleanup semantics.

For everyday scripting, practical guidance is:

- avoid storing huge data structures in long-lived globals unless needed,
- prefer small function scopes for temporary data,
- stream or chunk large inputs where possible.

The language makes the common case easy; you typically do not need to
"free" values manually.


## 3.11 Destructuring status in current implementation

Some languages let you write patterns like:

```text
let [ a, b ] := pair;
let { name, age } := person;
```

As of the current Perl implementation used in this guide, declaration
syntax centers on single identifiers (optionally typed), not full
pattern destructuring in `let`/`const` declarations.

So if you need "destructuring-like" behaviour today, use explicit access
instead:

```zzs
let pair := [ "Zia", 2 ];
let name := pair[0];
let cups := pair[1];

let user := { name: "Zia", team: "tooling" };
let team := user{team};
```

That is straightforward, readable, and aligned with parser/tested
syntax.


## 3.12 Practical patterns for healthy bindings

### Pattern 1: default to `const`, upgrade to `let` only when needed

If a name is configuration-ish or descriptive, `const` is often right.
If it represents evolving state, use `let`.

### Pattern 2: keep mutation narrow

Prefer:

```zzs
function compute_score (items) {
  let score := 0;
  for ( const item in items ) {
    score += item;
  }
  return score;
}
```

over sprawling shared mutable state at file scope.

### Pattern 3: make shadowing explicit and local

Tiny inner blocks with short-lived shadowing are readable. Deep nesting
with repeated shadow names is not.

### Pattern 4: use `?:=` for defaults at boundaries

```zzs
let retries;
retries ?:= 3;
```

Great for configuration load paths where user input may be missing.

### Pattern 5: keep import scope tight

Import symbols in the smallest sensible scope. This reduces collision
risk and keeps file-level namespace easier to scan.


## 3.13 Mini lab: Zia’s tiny shift planner

Let’s combine declaration, const-ness, scope, shadowing, and mutation.

```zzs
const mascot := "Zia";
let shift := {
  coffees: 1,
  tasks: [ "lint", "review" ],
};

function add_task ( task_name ) {
  shift{tasks}.push(task_name);
}

function coffee_up () {
  shift{coffees} += 1;
}

function status_line () {
  let label := `${mascot}: ${shift{coffees}} coffees`;

  if ( shift{tasks}.length() > 0 ) {
    let label := `${label}, ${shift{tasks}.length()} tasks queued`;
    return label;
  }

  return `${label}, inbox peaceful`;
}

add_task("docs");
coffee_up();
say status_line();
```

What this demonstrates:

- `mascot` is constant metadata (`const`),
- `shift` is mutable state (`let`),
- inner `label` shadows outer `label` only in the `if` block,
- mutations are performed on composite values via methods and field
  updates,
- function scopes keep helper names local.

If this feels natural to read, you have absorbed the core of this
chapter.


## 3.14 Recap

You now have a practical binding model for ZuzuScript (Perl runtime):

- `let` declares mutable bindings,
- `const` declares non-reassignable bindings,
- typed declarations constrain assignment intent,
- mutation of values and rebinding of names are different actions,
- lexical scope governs visibility for blocks, loops, and functions,
- shadowing is local-name override in inner scope,
- closures capture outer lexical values,
- values live as long as they remain reachable,
- declaration destructuring is not currently a primary `let`/`const`
  feature in this implementation.

In Chapter 4, we build on this with operators and expression mechanics,
so your value and binding intuition can turn directly into precise,
predictable computations.
