# Chapter 6: Functions: Small Pieces, Big Ideas

In Chapter 5, we learned how to control *when* code runs.

Now we ask the next cozy question:

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

As with earlier chapters, examples here follow the **Perl
implementation** and align with parser/runtime behaviour used in tests.


## 6.1 Function definition basics

A named function definition looks like this:

```zzs
function add ( x, y ) {
  return x + y;
}

say add( 2, 5 );  # 7
```

A few quick reminders:

- Parameters are listed in parentheses.
- Function bodies are blocks in `{ ... }`.
- `return` exits the function and gives back a value.
- If you do not return explicitly, the function still returns (often
  `null` unless your body computes and returns via control semantics).

A named function declaration creates a binding in the current scope, so
you can call it like any other value.

You can also predeclare a function name before giving it a body. This is
useful when two functions need to call each other:

```zzs
function handle_array;
function handle_dict;

function handle_array ( value ) {
	if ( value instanceof Dict ) {
		return handle_dict( value );
	}
	return "array";
}

function handle_dict ( value ) {
	if ( value instanceof Array ) {
		return handle_array( value );
	}
	return "dict";
}
```

After `function handle_array;`, the name is in scope, but calling it before
the full definition is reached throws an exception. Once the definition is
evaluated, the placeholder receives its body and behaves like an ordinary
const-like function binding.

The same predeclaration form is supported for `method name;` and
`static method name;` inside classes and traits, though it is rarely needed
because method lookup normally happens after the class or trait has been
built.

## 6.2 Positional parameters: the default shape

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

If the arity does not match the function contract, runtime errors are
raised (for example, too many or too few arguments when no variadic
collector is present).

### Readability tip

Try to keep positional argument order “obvious”:

- data first,
- options later,
- avoid signatures like `( a, b, c, flag, mode, maybe, x )`.

If a signature grows awkward, split into helper functions or use
named-style collection (covered below).


## 6.3 Typed parameters: runtime-checked contracts

The Perl implementation supports type annotations on parameters:

```zzs
function label_score ( Number n ) {
  return `score=${n}`;
}

say label_score( 9 );
```

If a call provides a mismatched value, runtime type checking raises a
`TypeException`.

You can type multiple parameters:

```zzs
function announce ( String name, Number cups ) {
  return name _ " has " _ cups _ " coffee";
}
```

Type annotations are especially useful at boundaries:

- user input parsing,
- module APIs,
- public helper functions used by many scripts.

They make failures earlier and clearer.


## 6.4 Optional and default parameters

You can make trailing parameters optional with `?`:

```zzs
function mood_line ( mood, label? ) {
  if ( label = null ) {
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
say tea_or_coffee( "Zia", "tea" );
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

That ordering rule is parser-enforced in the Perl implementation.


## 6.5 Variadic parameters and “named-style” arguments

Sometimes you want to accept additional arguments.

### Positional rest collector

Use `...` with a collector name:

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

In the Perl runtime, `rest` is an array-like value.

### Named argument collection

ZuzuScript call syntax supports label-style arguments such as
`key: value`. To receive these, define a `PairList` collector:

```zzs
function describe_raccoon ( name, ... PairList opts ) {
  return name _ " options=" _ opts.size();
}

say describe_raccoon(
  "Zia",
  mood: "sleepy",
  drink: "coffee"
);
```

You can think of this as a practical named-parameter pattern:

- caller writes labeled arguments,
- function receives collected key/value pairs.

It is flexible for option bags and evolving APIs.


## 6.6 Return values and return types

So far we have returned values informally. You can also declare a
return type using `->` (or Unicode `→`):

```zzs
function double ( Number n ) -> Number {
  return n * 2;
}
```

If a returned value does not match the declared type, runtime raises a
type error.

```zzs
function bad_label () -> Number {
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


## 6.7 Closures: functions that remember

A closure is a function value that captures variables from its lexical
surroundings.

```zzs
function make_counter ( start := 0 ) {
  let current := start;

  return function () {
    current += 1;
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

If Chapter 3’s scope/lifetime section felt abstract, closures are where
it becomes concrete.


## 6.8 Anonymous functions and `fn` lambdas

You have two common expression forms for unnamed callables.

### Anonymous `function` expression

```zzs
let square := function ( x ) {
  return x * x;
};

say square( 6 );
```

Use this when you want a full block body.

### `fn` lambda shorthand

`fn` is handy for short expression-like callbacks:

```zzs
let nums := [ 1, 2, 3, 4 ];
let doubled := nums.map( fn x -> x * 2 );

say doubled;
```

You can also use block-like side effects when collection APIs expect a
callback:

```zzs
let total := 0;

[ 1, 2, 3 ].each(
  fn x -> total := total + x
);

say total;  # 6
```

A practical rule:

- use `fn` for short, local callback intent,
- use `function` when logic grows beyond one simple expression/step.

Both are first-class values.


## 6.9 Recursion: functions calling themselves

Recursion is useful when a problem is naturally “same shape, smaller
input.”

Classic example:

```zzs
function factorial ( Number n ) -> Number {
  if ( n <= 1 ) {
    return 1;
  }

  return n * factorial( n - 1 );
}

say factorial( 5 );  # 120
```

Another beginner-friendly pattern: walking nested data.

```zzs
function count_nodes ( item ) -> Number {
  if ( item = null ) {
    return 0;
  }

  if ( typeof item = "Array" ) {
    let total := 1;

    for ( let child in item ) {
      total += count_nodes( child );
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

If any of these are missing, recursion becomes an infinite nap loop.
(Adorable, but not productive.)


## 6.10 Function design patterns you will use constantly

Functions are not just syntax; they are design choices.

Here are practical patterns that work well in ZuzuScript:

### Pattern A: Validate early, return early

```zzs
function parse_cups ( raw ) -> Number {
  if ( raw = null ) {
    return 0;
  }

  if ( raw = "" ) {
    return 0;
  }

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


## 6.11 Mini walkthrough: callback pipeline for sleepy tasks

Let’s combine typed params, defaults, closures, and lambdas.

```zzs
function make_filter ( Number min_priority := 5 ) {
  return fn task -> task{priority} >= min_priority;
}

function pick_titles ( tasks, keep ) -> Array {
  let out := [ ];

  for ( let t in tasks ) {
    if ( not keep( t ) ) {
      next;
    }

    out.push( t{name} );
  }

  return out;
}

let tasks := [
  { name: "lint", priority: 3 },
  { name: "docs", priority: 7 },
  { name: "ship", priority: 9 },
];

let keep_important := make_filter( 7 );
let picks := pick_titles( tasks, keep_important );

say picks;  # [ "docs", "ship" ]
```

What this demonstrates:

1. `make_filter` returns a closure capturing `min_priority`,
2. `pick_titles` accepts a function as a parameter,
3. callback-driven logic stays reusable and composable.

This style appears all over real scripts (build tooling, filters,
reporting jobs, CLI data transforms).


## 6.12 What comes next

You now know how to package behaviour and pass it around as values.

In Chapter 7, we move from function-level structure to object-level
structure:

- classes and methods,
- traits/roles,
- encapsulation,
- composition patterns.

If functions are excellent tools, objects are the toolbox.
Zia is currently sleeping *inside* that toolbox.
