# Chapter 7: Functions: Small Pieces, Big Ideas

In Chapter 6, we learned how to control *when* code runs.

Now we ask the next question:

> “How do we package useful logic so we can call it again?”

That is the job of functions.

Functions are where scripts stop being a pile of steps and start being
small systems of reusable ideas.

In this chapter we will cover:

- function definition and calling,
- positional, optional, default, typed, and named-style parameters,
- typed return values,
- closures and lexical capture,
- anonymous functions and lambdas,
- recursion.


## 7.1 Function definition basics

A named function definition looks like this:

```zzs
// Function definition
function add ( x, y ) {
	return x + y;
}

// Function call
say add( 2, 5 );  # 7
```

A few quick reminders:

- Parameters are listed in parentheses.
- Function bodies are blocks in `{ ... }`.
- `return` exits the function and gives back a value.
- If you do not return explicitly, the function still returns something
  (often `null`).

A named function declaration creates a binding in the current scope, so
you can call it like any other value.

You can also predeclare a function name before giving it a body. This is
useful when two functions need to call each other:

```zzs
function handle_array;
function handle_dict;

function handle_array ( value ) {
	if ( typeof value eq "Dict" ) {
		return handle_dict( value );
	}
	return "array";
}

function handle_dict ( value ) {
	if ( typeof value eq "Array" ) {
		return handle_array( value );
	}
	return "dict";
}
```

After `function handle_array;`, the name is in scope, but calling it before
the full definition is reached throws an exception. Once the definition is
evaluated, the placeholder receives its body and behaves like an ordinary
function.


### Function scoping

Like variables, functions are only defined in a particular scope.

```zzs
{
	function do_it () {
		say "Just do it!";
	}
	
	do_it();  // works
}

do_it();  // error: no such function
```


## 7.2 Positional parameters: the default shape

Most functions start with positional parameters.

```zzs
function brew_label ( cups, mode ) {
	return mode _ ":" _ cups;
}

say brew_label( 2, "cozy" );
```

Arguments are matched by order:

1. first argument -> first parameter,
2. second argument -> second parameter,
3. and so on.

If a function is called with too many or too few arguments, an error
occurs.


## 7.3 Typed parameters: runtime-checked contracts

SusScript supports type annotations on parameters:

```zzs
function label_score ( Number n ) {
	return `score=${n}`;
}

say label_score( 9 );
```

If the function is called with a mismatched value it causes a
`TypeException` error.

You can type multiple parameters:

```zzs
function announce ( String name, Number cups ) {
	say `${name} has ${cups} of coffee.`;
}
```

Type annotations are especially useful at boundaries:

- user input parsing,
- module APIs,
- public helper functions used by many scripts.

They make failures earlier and clearer. If a function is just a little
helper used in one place, it may be less important to check parameter
types.


## 7.4 Optional and default parameters

You can make trailing parameters optional with `?`:

```zzs
function mood_line ( mood, label? ) {
	if ( label ≡ null ) {
		return "(no label) " _ mood;
	}

	return label _ ": " _ mood;
}

say mood_line( "sleepy" );
say mood_line( "sleepy", "Zia" );
```

You can also provide defaults with `:=`:

```zzs
function tea_or_coffee (
	String name,
	String drink := "coffee"
) {
	return name _ " picks " _ drink;
}

say tea_or_coffee( "Zia" );
say tea_or_coffee( "Zenia", "tea" );
```

### Ordering rule (important)

Once a parameter is optional (`?`) or has a default (`:=`), following
parameters cannot be required.

Good:

```zzs
function ok ( a, b?, c := 3 ) {
	return a;
}
```

Not allowed:

```zzs
function not_ok ( a?, b ) {
	return a;
}
```


## 7.5 Variadic parameters, named-style arguments, and argument spread

Sometimes you want to accept additional arguments.

### Positional rest collector

In a parameter list, use `...` with a collector name:

```zzs
function add_all ( ... rest ) {
	let total := 0;

	for ( let n in rest ) {
		total += n;
	}

	return total;
}

say add_all( 1, 2, 3, 4 );  # 10
```

The `rest` variable will be an Array that collects any additional
positional arguments.

### Named argument collection

ZuzuScript call syntax supports key–value-style arguments. To receive
these, define a collector but give it type `PairList`:

```zzs
function describe_job ( name, ... PairList opts ) {
	return name _ " options=" _ opts.length();
}

say describe_job(
	"build",
	mode: "release",
	drink: "coffee"
);
```

You can think of this as a practical named-parameter pattern:

- caller writes labeled arguments,
- function receives collected key/value pairs.

It is a flexible pattern for options and evolving APIs.

It is possible to define both kinds of collectors for the dame
function:

```zzs
function do_stuff ( ... Array args, PairList opts ) {
	...;
}

do_stuff( opt1: true, "arg1", opt2: false, "arg2", "arg3" );
```

### Argument spread at call sites

In a call argument list, `...expr` is argument spread. It evaluates
`expr` and expands the resulting collection at that point in the call:

```zzs
function positional ( ... args ) {
	return args;
}

say positional( 1, ...[ 2, 3 ], 4 );  // [ 1, 2, 3, 4 ]
```

Array spread appends positional arguments. Dict and PairList spread
append named arguments:

```zzs
function capture ( ... PairList opts ) {
	return opts;
}

let from_dict := capture( before: 0, ...{ alpha: 1 }, after: 2 );
say from_dict{"alpha"};  // 1

let from_pairs := capture( ...{{ tag: "one", tag: "two" }} );
say from_pairs.get_all("tag");  // [ "one", "two" ]
```

Argument operands evaluate left-to-right, with each spread expanded in
place. PairList spread preserves pair order and duplicate keys. Dict
spread supplies named arguments for its entries, but code should not rely
on Dict iteration order. Spreading any value other than Array, Dict, or
PairList throws an exception.

Keep the three `...` meanings separate:

- `function f ( ... args )` collects extra positional arguments,
- `f( ...values )` spreads a collection into a call,
- `[ 1 ... 5 ]` is an inclusive range inside collection literals.


## 7.6 Return values and return types

So far we have returned values informally. You can also declare a
return type using `→` (or the more keyboard-friendly `->`):

```zzs
function double ( Number n ) → Number {
	return n × 2;
}
```

If a returned value does not match the declared type, runtime raises a
type error.

```zzs
function bad_label () → Number {
	return "oops";
}
```

That contract applies whether returning from:

- the end of the function,
- an early `return` inside a branch.

### Why return types help

Return types answer: “What comes back from this function?”

That improves:

- call-site confidence,
- maintenance safety,
- tooling opportunities.

For small local helpers they may be optional, but for shared APIs they
are often worth adding.


## 7.7 Closures: functions that remember

A closure is a function value that captures variables from its lexical
surroundings.

```zzs
function make_counter ( start := 0 ) {
	let current := start;

	return function () {
		current := current + 1;
		return current;
	};
}

let next_id := make_counter( 40 );
say next_id();  # 41
say next_id();  # 42
```

Even after `make_counter` returns, the inner function keeps access to
`current`. That is lexical capture in action.

### Practical closure uses

Closures are great for:

- stateful callbacks,
- tiny configurable helpers,
- memoization-like caches,
- dependency injection without classes.


## 7.8 Anonymous functions and `fn` lambdas

You have two common expression forms for unnamed callables.

### Anonymous `function` expression

```zzs
let square := function ( n ) {
	return n × n;
};

say square( 6 );
```

Use this when you want a full block body.

### `fn` lambda shorthand

The `fn` keyword is an abbreviated way of writing anonymous functions.
The example above could be written as:

```zzs
let square := fn n → n × n;
say square( 6 );
```

`fn` is handy for short expression-like callbacks:

```zzs
let nums := [ 1, 2, 3, 4 ];
let doubled := nums.map( fn n → n * 2 );

say doubled;
```

For the shortest single-argument callbacks, start with an arrow. This
creates a lambda whose optional argument is available as `^^`:

```zzs
let square := → ^^ × ^^;

say square( 6 );
```

The `^^` parameter is implicit in the leading-arrow form. It is not valid
in an explicit `fn` parameter list.

You can also use block-like side effects when collection APIs expect a
callback:

```zzs
let total := 0;
[ 1, 2, 3 ].each( → total := total + ^^ );
say total;  # 6
```

A practical rule:

- use `fn` for short, local callback intent,
- use `function` when logic grows beyond one simple expression/step.

The ultra-abbreviated form using the leading `→` and `^^` is best when
it already obvious from context that an anonymous function is expected,
such as in the `.each` example above.


## 7.9 Recursion: functions calling themselves

Recursion is useful when a problem is naturally "same shape, smaller
input."

Classic example:

```zzs
function factorial ( Number n ) -> Number {
	if ( n ≤ 1 ) {
		return 1;
	}
	return n × factorial( n - 1 );
}

say factorial( 5 );  # 120
```

Another beginner-friendly pattern: walking nested data.

```zzs
function count_nodes ( item ) → Number {
	return 0 if item ≡ null;

	if ( typeof item eq "Array" ) {
		let total := 1;

		for ( const child in item ) {
			total := total + count_nodes( child );
		}

		return total;
	}

	return 1;
}
```

### Recursion safety checklist

When writing recursive functions, verify three things:

1. **Base case exists** (stops recursion),
2. **Recursive step progresses** toward base case,
3. **Return type stays consistent** across all paths.

If any of these are missing, recursion becomes an infinite loop.


## 7.10 Function design patterns you will use constantly

Functions are not just syntax; they are design choices.

Here are practical patterns that work well in ZuzuScript:

### Pattern A: Validate early, return early

```zzs
function parse_cups ( raw ) -> Number {
	return 0 if raw ≡ null;
	return 0 if raw eq "";

	return raw + 0;
}
```

Keeps the “normal path” at the bottom and shallow.

### Pattern B: Keep one job per function

```zzs
function fetch_feed ( url ) { ... }
function parse_feed ( text ) { ... }
function render_feed ( items ) { ... }
```

Small, focused functions are easier to test and reuse.

### Pattern C: Name by outcome, not mechanism

Prefer `build_report` over `loop_and_concat_stuff`.

Future-you (and teammates) will thank you.

### Pattern D: Stabilize boundaries with types

For public or reused helpers:

- type key parameters,
- add return types for non-trivial outputs,
- use defaults for optional behaviour.

This creates contracts that scale as code grows.


## 7.11 Mini walkthrough: callback pipeline for sleepy tasks

Let’s combine typed params, defaults, closures, and lambdas.

```zzs
function make_priority_filter ( Number min_priority := 5 ) {
	return fn task → task{priority} ≥ min_priority;
}

function pick_titles ( Array tasks, Function keep ) → Array {
	let out := [];

	for ( const t in tasks ) {
		next unless keep(t);
		out.push( t{name} );
	}
	
	return out;
}

let tasks := [
	{ name: "lint", priority: 3 },
	{ name: "docs", priority: 7 },
	{ name: "ship", priority: 9 },
];

let keep_important := make_priority_filter(7);
let picks := pick_titles( tasks, keep_important );

say picks;  # [ "docs", "ship" ]
```

What this demonstrates:

1. `make_priority_filter` returns a closure capturing `min_priority`,
2. `pick_titles` accepts a function as a parameter,
3. callback-driven logic stays reusable and composable.

This style appears all over real scripts (build tooling, filters,
reporting jobs, CLI data transforms).

<img src="https://zuzulang.org/img/zia-toolbox.jpeg" alt="Let her sleep." class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />


## 7.12 What comes next

You now know how to package behaviour and pass it around as values.

In Chapter 8, we move from function-level structure to object-level
structure:

- classes and methods,
- traits/roles,
- encapsulation,
- composition patterns.

If functions are excellent tools, objects are the toolbox.

