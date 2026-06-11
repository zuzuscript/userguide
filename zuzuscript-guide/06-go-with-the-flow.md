# Chapter 6: Go with the Flow

<img src="https://zuzulang.org/img/zia-flow.jpeg" alt="Wake up, Zia!" class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />

Chapter 5 gave us truth values. Now we can use them to choose what happens
next.

With what we've learnt so far, we can already write some basic programs.

```zzs
let tax_percent := 0.20;
let initial_price := 75.00;

let total_pennies := ⌈ 100 × initial_price × ( 1 + tax_percent ) ⌉;
say `The price including tax is ${ total_pennies ÷ 100 }.`;
```

However, it's rare for programs to be as simple and linear as that.
Real programs often:

- make a decision,
- repeat work while a condition holds,
- iterate over collections,
- skip some steps,
- stop a loop early,
- return from a function as soon as you have the answer.


## 6.1 Conditions with `if`, `else if`, and `else`

The most common control structure is probably the trusty `if`. We've
already seen it a few times in examples, but here's a longer example.

```zzs
let cups := 2;

if ( cups ≥ 3 ) {
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
expressions can appear there. Image `get_data()` is a function that
sometimes returns a `Dict` of data, but other times (when there's no new
data available to process?) returns `null`

```zzs
if ( let data := get_data() ) {
	say `There are ${ data{widget_count} } new widgets available.`;
}
```


## 6.2 Postfix conditionals: quick and tidy

ZuzuScript also supports postfix `if` and `unless` conditionals for
short one-liners.

Instead of this:

```zzs
let points := 0;

if ( not it_is_tuesday ) {
	points++;
}

if ( username eq "zia" ) {
	points++;
}
```

Try this:

```zzs
let points := 0;

points++ unless it_is_tuesday;
points++ if username eq "zia";
```

Postfix forms are great for small “do this only when…” statements.
For larger logic, prefer full `if` blocks.


## 6.3 Braces and scopes

Most control flow structures use braces `{ ... }`. Braces serve an important
function in ZuzuScript: they act as scopes.

```zzs
{
	let mynum := 4;
	...;
	say mynum; // says 4
}

say mynum; // ERROR
```

In this example, the variable `mynum` is available as soon as it's been
declared by `let` and stays visible until the end of its scope, marked with
`}`. Outside that scope, it is no longer visible and trying to access the
variable by name is an error.

Names defined in "higher level" scopes are still accessible.

```zzs
{
	let mynum := 4;
	
	{
		let othernum := 1;

		// In this scope, both othernum and mynum are visible
		say othernum + mynum;
	}

	// Here othernum has gone out of scope, but mynum is
	// still visible.
}

// And here neither is visible.
```

The whole file is the "highest level" scope.

ZuzuScript's idea of variable scoping roughly matches Perl, Raku,
modern JavaScript, Rust, Swift, Go, and Lua. This is different from the
scoping rules used by older JavaScript (`var`) and Python, where variables
are function-scoped rather than block-scoped.


## 6.4 Repeating work with `while`

Use `while` when you want to repeat as long as a condition stays truthy.

```zzs
let brewed := 0;

while ( brewed < 3 ) {
	say "Brewing coffee…";
	brewed++;
}

say brewed;  # 3
```

## 6.5 Iteration with `for`

Use `for` when you already have a collection (or iterable expression)
and want to visit each item.

```zzs
let sum := 0;

for ( let n in [ 1, 2, 3 ] ) {
	sum := sum + n;
}

say sum;  # 6
```

The examples in this chapter use `for` to loop over arrays. It also
works with sets and bags, though you cannot rely on why order it loops
over elements.

You can loop over a dict or pairlist too:

```zzs
let my_dict := { foo: 1, bar: 2 };

for ( let item in my_dict ) {
	say item;
}
```

In this example, `item` is set to each key, which is fairly useful for
dicts, but less useful for pairlists because keys may reoccur. The
`enumerate` method is useful here as it allows you to access the key–value
pair as a single object.

```zzs
let my_dict := { foo: 1, bar: 2 };

for ( let pair in my_dict.enumerate ) {
	say `${pair.key}: ${pair.value}`;
}
```

### Looping over Strings and BinaryStrings

A `for` loop over a String visits each character in turn, as a
one-character String:

```zzs
for ( let char in "héllo" ) {
	say char;  # h, é, l, l, o
}
```

Characters here means what Unicode calls code points, so `"é"` is a
single character even though it takes more than one byte to store.

Looping over a BinaryString visits each byte instead, as a one-byte
BinaryString:

```zzs
let count := 0;

for ( let byte in to_binary("héllo") ) {
	count++;
}

say count;  # 6 — "é" takes two bytes in UTF-8
```

This is the same text either way; the difference is whether you are
asking for its characters or for the raw bytes of its UTF-8 encoding.
The postfix form of `for` (covered below) works with Strings and
BinaryStrings too.

It is also possible to loop over an iterator, which is a special type
of function that will be covered in a future chapter.


## 6.6 The `const` keyword

An alternative to `let` is the `const` keyword. You can use it to indicate
that you don't expect a value to change for the rest of the scope.

We've actually already seen one constant:

```zzs
const π := 3.14159265359;

π := 3; // Error!
```

Simple scalar types (null, booleans, numbers, strings, binary strings) which
are declared with `const` will be strictly constant.

Collections (and objects when we come to those) have a more nuanced
interpretation of `const`.

```zzs
const my_list := [];

my_list.add( "Coffee" );  // This is okay
my_list.add( "Sleep" );   // So is this
my_list.clear();          // And this

my_list := []; // Error!
```

If a collection or object is declared with `const` you will still be able
to change its contents, but you can't assign a different list or object
to that name.

`const` can be used with `for` loops and it's actually often a good idea.

```zzs
let sum := 0;

// Within the loop body, we are not going to alter `n`.
for ( const n in [ 1, 2, 3 ] ) {
	sum := sum + n;
}

say sum;  # 6
```


## 6.7 More about `for`

There are a couple of shortcuts to using `for`. Firstly, it's possible
to skip specifying a name for the loop variable and it will default to
the magic `^^` (looks like Zia's ears).

```zzs
let sum := 0;

for ( [ 1, 2, 3 ] ) {
	sum := sum + ^^;
}

say sum;  # 6
```

`^^` acts like a const (you cannot assign a new value to it). This is
not the only feature of ZuzuScript which makes use of this magic const.

Another shortcut is that for loops of just a single statement, you can
use a postfix form of `for` (like `if` and `unless`).

```zzs
let sum := 0;
sum := sum + ^^ for [ 1, 2, 3 ];
say sum;  # 6
```

For larger loops that do a lot of work inside the loop body, you'll
probably find that using the full long form of `for` makes your code
look easiest to read. But if you just need to do something small in a
loop, the postfix `for` combined with `^^` is quite readable.

The `for` keyword supports an `else` block like `if` does:

```zzs
let my_numbers := [];
let sum := 0;

for ( const n in my_numbers ) {
	sum := sum + n;
}
else {
	// This code runs if my_numbers is empty.
	sub := -1;
}

say sum;

```

## 6.8 `next` and `last` in loops

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

  total := total + x;
}

say total;  // 1 + 3 + 5 = 9
```

A reliable mental model:

- use `next` when one iteration should be skipped,
- use `last` when the whole loop should stop.


## 6.9 `switch`, `case`, `default`, and `continue`

When you have one subject value and multiple branches, `switch` keeps
logic flatter than a long `else if` chain.

```zzs
let mood := "sleepy";
let plan := "";

switch ( mood : eq ) {
	case "focused":
		plan := "write module";
	case "sleepy", "cozy":
		plan := "drink coffee";
	default:
		plan := "take a walk";
}

say plan;
```

### Comparator form

`switch` can specify a comparator after `:`.

- `switch ( value )` → default matching behaviour,
- `switch ( value : eq )` → string comparison,
- `switch ( value : eqi )` → case-insensitive string comparison,
- `switch ( value : = )` → numeric comparison,

Any comparison operator is allowed.

### Fallthrough is explicit with `continue`

Unlike C-style switch semantics, cases do **not** fall through by
default. If you want to continue into the next case/default, use the
`continue` keyword explicitly.

```zzs
let picked := "";

switch ( "bar" : eq ) {
	case "foo":
		picked := "foo";
	case "bar", "baz":
		picked := "matched";
		continue;
	default:
		picked := picked _ "-default";
}

say picked;  # "matched-default"
```

That explicitness is great for readability: fallthrough is always a
conscious choice.


## 6.10 Choosing the right flow tool

When writing real scripts, choose control-flow constructs by shape:

- **`if` / `else if` / `else`** for one-off branching decisions,
- **`switch`** for one subject with many named cases,
- **`while`** for condition-driven repetition,
- **`for`** for collection-driven repetition,
- **`next` / `last`** for fine-grained loop control,

Readable control flow is one of the biggest quality multipliers in
scripting code.

If the code starts feeling twisty, split the work into helper functions.
Which is our next topic.
