# Chapter 6: Go with the Flow

<img src="https://zuzulang.org/img/zia-flow.jpeg" alt="Wake up, Zia!" class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />

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

## 6.6 The `const` keywords

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


