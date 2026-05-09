# Chapter 4: Do the Math (and Then Some): Operators & Expressions

In Chapter 3, we focused on names: `let`, `const`, scope, and how
bindings differ from the values they refer to.

Now we finally get to the part where sleepy raccoons become productive:
**expressions**.

Expressions are how ZuzuScript computes values.

- `1 + 2` is an expression.
- `name _ "!"` is an expression.
- `score > 10 and not tired` is an expression.
- `left union right` is an expression.

If Chapter 3 gave us labeled boxes, Chapter 4 teaches us how to do
useful work with what is inside the boxes.

As in the previous chapters, this material is written against the
**Perl implementation**, and examples follow syntax validated by the
language ztests and POD-backed runtime docs.


## 4.1 What counts as an expression?

An expression is anything that evaluates to a value.

```zzs
let naps := 2 + 1;
let mascot := "Zia" _ " the sleepy";
let ready := naps < 3 and true;
```

Each right-hand side computes a value:

- `2 + 1` -> number,
- `"Zia" _ " the sleepy"` -> string,
- `naps < 3 and true` -> boolean-like truth result.

You can nest expressions as deeply as needed:

```zzs
let label := `status=${ ( 8 ÷ 2 ) > 3 ? "ok": "hmm" }`;
```

The parser evaluates this using precedence rules we will cover later.


## 4.2 Arithmetic operators (with Unicode aliases)

Let’s start with the daily coffee math.

### Core numeric operators

```zzs
let a := 7 + 5;      # 12
let b := 7 - 5;      # 2
let c := 3 * 4;      # 12
let d := 3 × 4;      # 12 (unicode alias)
let e := 9 / 4;      # 2.25
let f := 9 ÷ 4;      # 2.25 (unicode alias)
let g := 2 ** 5;     # 32
let h := 17 mod 5;   # 2
```

A practical note: this guide prefers Unicode forms (`×`, `÷`, `≤`,
`≥`, etc.) when available, but ASCII forms are fully supported and are
used in many real scripts.

### Numeric coercion in the Perl runtime

In the Perl implementation, numeric operators coerce operands to numbers
using runtime rules. That means values like booleans and numeric-looking
strings can still participate in arithmetic.

```zzs
say "12" + 5;     # 17
say null + 3;     # 3
```

This is powerful, but you should still keep your intent explicit.
When a value should be numeric, keep it numeric on purpose.


## 4.3 Comparison operators: numeric, string, and type-aware

One of ZuzuScript’s friendliest design choices is splitting comparison
families by intent.

## Numeric comparisons

Use numeric operators for numeric meaning:

```zzs
say 2 = 2;
say 2 != 3;
say 2 ≠ 3;
say 2 < 3;
say 2 <= 2;
say 2 ≤ 3;
say 3 >= 3;
say 3 ≥ 2;
say 1 <=> 9;      # -1
say 9 ≶ 1;        # 1
say 4 ≷ 4;        # 0
```

Yes, in ZuzuScript `=` is numeric equality inside expressions.
Declaration and assignment still use `:=`, so there is no ambiguity.

## String comparisons

Use lexical string operators when comparing text:

```zzs
say "abc" eq "abc";
say "abc" ne "abd";
say "b" gt "a";
say "a" lt "b";
say "a" cmp "b";     # -1
```

Case-insensitive variants are also available:

```zzs
say "A" eqi "a";
say "B" gti "a";
say "a" cmpi "B";    # -1
```

## Type-aware equality

`==` checks type-aware equality, not loose coercive equality.

```zzs
say 1 == 1;       # true
say 1 ≡ 1;        # true
say "1" == 1;    # false (different types)
say 1 ≢ "1";     # true
```

This catches whole classes of bugs early, especially in scripts that
mix user input, config text, and numeric data.


## 4.4 Boolean logic and truthiness

Boolean expressions are where conditions get assembled.

### Logical operators

```zzs
say true and true;
say false or true;
say true xor false;
say not false;
```

Unicode aliases also exist:

```zzs
say true ⋀ true;
say false ⋁ true;
say true ⊻ false;
say ¬false;
```

`nand` and `⊼` are available too:

```zzs
say true nand true;   # false
say true ⊼ false;     # true
```

### Truthiness (important in conditionals)

In the Perl runtime, condition evaluation uses a truthiness model:

- falsy scalars include `null`, `""`, `"0"`, `"0.0"`, and numeric `0`,
- non-empty collections are truthy,
- empty collections are falsy,
- objects are truthy unless they provide custom boolean behaviour.

That means this is valid and common:

```zzs
let queue := [ ];
if ( not queue ) {
  say "No tasks yet.";
}
```

And this is also common:

```zzs
let queue := [ "lint" ];
if ( queue ) {
  say "Work exists (unfortunately).";
}
```


## 4.5 String expressions: concatenation and text helpers

ZuzuScript uses `_` for string concatenation.

```zzs
let name := "Zia";
let msg := "Hello, " _ name _ "!";
say msg;
```

Assignment form `_=` is useful for builders:

```zzs
let line := "TODO:";
line _= " drink coffee";
line _= " and write tests";
```

### Template interpolation

Template strings let you embed expressions directly.

```zzs
let sleepy := true;
say `Zia mood=${ sleepy ? "sleepy": "awake" }`;
```

Interpolation evaluates full expressions, not just variable names.

### `to_String` hooks for objects

In the Perl implementation, non-string objects can participate in string
contexts by implementing `to_String`.

```zzs
class Label {
  let value;

  method to_String () {
    return "L:" _ value;
  }
}

let label := new Label( value: 7 );
say "x=" _ label;      # "x=L:7"
```

This makes object output feel natural in logs, templates, and messages.


## 4.6 Set and bag operators (the fun part)

Chapter 2 introduced sets and bags. Now we operate on them.

### Membership

Use `in`, `∈`, and `∉` to test membership:

```zzs
say 2 in[ 1, 2, 3 ];
say 2 ∈ << 1, 2, 3 >>;
say 4 ∉ << 1, 2, 3 >>;
```

### Union, intersection, and difference (sets)

```zzs
let left := << 1, 2 >>;
let right := << 2, 3 >>;

let u1 := left union right;
let u2 := left ⋃ right;

let i1 := left intersection right;
let i2 := left ⋂ right;

let d1 := << 1, 2, 3 >> \ << 2 >>;
let d2 := << 1, 2, 3 >> ∖ << 2 >>;
```

### Subset, superset, and set-equivalence

```zzs
say << 1, 2 >> subsetof << 1, 2, 3 >>;
say << 1, 2 >> ⊂ << 1, 2, 3 >>;

say << 1, 2, 3 >> supersetof << 1, 2 >>;
say << 1, 2, 3 >> ⊃ << 1, 2 >>;

say << 1, 2 >> equivalentof << 2, 1 >>;
say << 1, 2 >> ⊂⊃ << 2, 1 >>;
```

### Bags and multiplicity

Bags keep duplicate counts, so operations should be read with
multiplicity in mind.

```zzs
let snacks := <<< "cookie", "cookie", "tea" >>>;
say snacks.count("cookie");
```

As we go deeper in Chapter 8, we will map out exactly how each bag
operation treats counts.


## 4.7 Ranges are expressions too

Range syntax is `start ... end` and can be used inside arrays, sets,
and bags.

```zzs
say [ 1 ... 5 ];
say [ 5 ... 1 ];

say << 1 ... 4, 3 ... 1 >>;
say <<< 1 ... 2, 2 ... 1 >>>;
```

Both ascending and descending forms are supported.

For beginners, ranges are a great way to avoid noisy index-building
loops when you just need a short sequence.


## 4.8 Assignment operators as expressions-in-action

You already saw `:=` in Chapter 3. Here are the operator forms that
bundle a computation with assignment.

```zzs
let x := 1;
x += 4;      # 5
x -= 1;      # 4
x *= 3;      # 12
x ×= 2;      # 24
x /= 6;      # 4
x ÷= 2;      # 2
x **= 3;     # 8
```

String assignment:

```zzs
let s := "ab";
s _= "cd";
```

Null-coalescing assignment:

```zzs
let preferred;
preferred ?:= "coffee";   # assigned
preferred ?:= "tea";      # ignored (already defined)
```

`?:=` is excellent for defaults coming from environment, CLI options,
or config layers.

In-place regexp replacement assignment with `~=`:

```zzs
let label := "10x20 30x40";
label ~= /([0-9]+)x([0-9]+)/g -> `${m[1]}y${m[2]}`;
label ~= /foo/i → ( m[0] _ "!" );
```

Inside the replacement expression, `m` is the regexp match array for
that replacement pass only.

These assignment forms also work on path lvalues:

```zzs
report @ "/meta/count" += 1;
report @@ "/users/*/role" _= "-active";
report @? "/meta/title" ?:= "Untitled";
```

## 4.9 Prefix/postfix operators and reference expressions

### Increment/decrement

```zzs
let x := 1;
x++;

let y := 1;
say ++y;
```

Both prefix and postfix forms are supported for `++` and `--`.

Path lvalues can also be updated this way, but they should be wrapped in
parentheses so the target is unambiguous:

```zzs
( report @ "/meta/count" )++;
++ ( report @@ "/users/*/age" );
--( report @? "/meta/maybe_count" );
```

### Lvalue references with unary `\`

The Perl implementation supports creating references to assignable
locations (lvalues), including indexes, dict entries, and slices.

```zzs
let row := [ 10, 20 ];
let slot := \ row[1];

say slot();      # getter
slot(42);        # setter
say row[1];
```

This is advanced, but very useful for APIs that need a reusable
read/write handle to part of a larger structure.

Path lvalues can also produce references:

```zzs
let title_ref := \( report @ "/meta/title" );
let age_refs := \( report @@ "/users/*/age" );
let maybe_ref := \( report @? "/meta/title" );
```

As with `++` and `--`, keep the path expression in parentheses when
using unary `\` so it is parsed as the intended lvalue target.


## 4.10 Operator precedence: who binds first?

When you write a mixed expression, precedence rules determine grouping
unless you add parentheses.

A practical high-to-low sketch for common day-to-day use:

1. `**`
2. `*`, `×`, `/`, `÷`, `mod`
3. `+`, `-`
4. `_`
5. set operators (`union`, `intersection`, `\`, `∖`)
6. comparisons (`=`, `<`, `eq`, `in`, `~`, path operators)
7. type-aware equality (`==`, `!=`, `≡`, `≢`)
8. `and`, `xor`, `or` (low precedence family)
9. ternary `? :` and short ternary `?:`
10. assignment forms (`:=`, `~=`, `+=`, `_=`, `?:=`)

`**` is right-associative. Most other binary operators are left-
associative.

### Parentheses are a kindness

Even if you know precedence, add parentheses when it aids readability:

```zzs
let ok := ( retries < 3 ) and ( status eq "ready" );
let msg := "n=" _ ( count + 1 );
```

Readable code wins over clever code every time.


## 4.11 Expressions with custom object coercion

Objects can hook into expression behaviour by providing methods such as
`to_Number`, `to_String`, and `to_Boolean`.

```zzs
class Numeric {
  method to_Number () {
    return 7;
  }
}

class Flag {
  method to_Boolean () {
    return 0;
  }
}

let n := new Numeric();
let f := new Flag();

say n + 5;        # 12
say f ? 1: 2;     # 2
```

This is a powerful extension point, but use it with restraint: people
reading your code should still be able to predict behaviour quickly.


## 4.12 Common expression pitfalls (and easy fixes)

### Pitfall 1: using numeric operators for strings

```zzs
# Not ideal when you mean lexical compare:
# if ( user_input > "m" ) { ... }
```

Use string operators for text intent:

```zzs
if ( user_input gt "m" ) {
  say "late alphabet";
}
```

### Pitfall 2: forgetting type-aware `==`

If you expect coercion, `==` may surprise you.

```zzs
say "1" == 1;    # false
say "1" = 1;     # true after numeric coercion
```

Choose explicitly: strict type-aware comparison, or numeric compare.

### Pitfall 3: relying on implicit precedence in dense expressions

If you need to pause and mentally parse it, future-you will too.
Use grouping parentheses.

### Pitfall 4: treating set and bag operations as interchangeable

Sets answer membership. Bags answer multiplicity. Pick based on what the
program is trying to preserve.


## 4.13 Mini walkthrough: Zia’s coffee task score

Let’s combine several operator families in one small expression pipeline.

```zzs
let tasks := [ "lint", "test", "docs" ];
let done := << "lint", "docs" >>;
let coffees := 2;

let remaining := << "lint", "test", "docs" >> \ done;
let score := ( done.length() * 10 ) + ( coffees * 3 );
let mood := score >= 20 and not remaining.contains("docs");

say `remaining=${remaining.length()} score=${score}`;
say mood ? "Zia cartwheel-ready": "Zia nap-ready";
```

This reads close to plain language:

- compute remaining tasks with set difference,
- compute a numeric score,
- compute boolean mood from threshold logic.

That is the essence of expression-driven code.


## 4.14 What’s next

You now have the operator toolbox needed for meaningful computation:

- arithmetic and comparison,
- boolean logic,
- string composition,
- collection algebra,
- precedence and grouping habits.

In Chapter 5, we move from expressions to **execution direction**:
`if`, loops, switch/match patterns, and the control-flow tools that turn
single expressions into full program behaviour.
