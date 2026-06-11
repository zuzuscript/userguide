# Chapter 2: Math is Math

<img src="https://zuzulang.org/img/zia-mim.jpeg" alt="Why would they change math?" class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />

Almost every non-trivial program you write will need to deal with numbers, so
numbers are a great place to start learning about ZuzuScript.

Chapter 1 got a script onto the screen. This chapter starts turning that
script into a language you can reason about: values, types, expressions,
operators, modules, and method calls.

We'll sneak in that general knowledge while we are talking about arithmetic,
because numbers give us a simple place to see how ZuzuScript thinks.


## 2.1 The `Number` data type

ZuzuScript, like many programming languages, has types. All values internally
know whether they're, for example, a number or a boolean (truth value) or a
string (chunk of text).

Some programming languages provide multiple different numeric types, but
ZuzuScript has just one: `Number`.

The `Number` data type can represent both integers and floating point numbers.
The interpreter keeps the details of exactly how the number in represented
internally hidden away: you don't need to think about it.

```zzs
let value := 4;           // sets value to 4
value := 5;               // now it is 5
value := "Hello";         // now it is not a number at all
```

Note that `//` indicates the beginning of a comment. It tells the ZuzuScript
interpreter to ignore the rest of the line. This allows you to put short
bits of text in your scripts for humans to read.

It is possible to declare the type of a variable to ensure it will always
hold a value of a particular type.

```zzs
let Number value := 4;    // declares value to always be a Number, currently 4
value := 5;               // now it is 5
value := "Hello";         // this causes an error!!
```


### Alternative ways to write numbers

Plain decimal numbers like `4` and `2.5` cover most needs, but number
literals come in a few other forms.

Very large and very small numbers can be written in scientific notation
("E notation") using an upper-case `E`:

```zzs
say 1E3;      // 1000
say 2.5E-7;   // 0.00000025
```

Integers can also be written in hexadecimal (base 16), binary (base 2),
or octal (base 8) using the prefixes `0x`, `0b`, and `0o`:

```zzs
say 0x1F;        // 31
say 0b11111111;  // 255
say 0o100;       // 64
```

Hexadecimal digits may be written in either case (`0xDEADBEEF` works),
but the `E` of scientific notation must be upper-case and the letter in
the `0x`, `0b`, and `0o` prefixes must be lower-case: `1e3` and `0X1F`
are syntax errors. However it is written, the result is an ordinary
`Number`; `0x1F` and `31` are the same value.

### Introducing the `typeof` operator

The `typeof` operator tells you the data type associated with a value.

```zzs
let Number x := 4;
say x;               // says 4
say typeof x;        // says Number

let y := 5;
say y;               // says 5
say typeof y;        // says Number
```

Note that it's not telling you about the type you declared a variable as,
but about the type of the value currently in that variable.

Strings (even ones that look like numbers) are a distinct data type and
should not be confused with numbers.

```zzs
let x := 4;
let y := "4";

say typeof x;   // Number
say typeof y;   // String
```


## 2.2 Mathematical operators

ZuzuScript has a lot of mathematical operators built into the language,
allowing you to perform simple maths.

```zzs
let x := 8;
let y := 3;
let z := x + y;

say z;   // says 11
```

The `+` operator is a "binary infix operator". That is just fancy speak
for a symbol that sits between two values and does something to them.
The `+` operator sits between two values and adds them together.

Other mathematical binary infix operators include:

```zzs
a + b;       // Addition
a - b;       // Subtraction
a × b;       // Multiplication
a * b;       // Multiplication, but easier to type
a ÷ b;       // Division
a / b;       // Division, but easier to type
a ** b;      // Exponention ("to the power of")
a mod b;     // Modulo arithmetic (remainder after division)
```

### Numeric assignment operators

If you want to update a variable using its current value, you can combine
a mathematical operator with assignment.

```zzs
let score := 10;

score += 5;     // same as score := score + 5
score -= 2;     // same as score := score - 2
score *= 3;     // same as score := score * 3
score ×= 2;     // same as score := score × 2
score /= 4;     // same as score := score / 4
score ÷= 2;     // same as score := score ÷ 2
score **= 2;    // same as score := score ** 2
```

These are useful for counters, totals, scores, and other values that change
a little at a time.

Some operators only operate on a single value instead of sitting between
values. They are called unary prefix or unary suffix operators depending
on whether you put the operator before or after the value.

```zzs
// Unary prefix operators
+a;          // Identity operator (does nothing much)
-a;          // Make positive number negative, and vice versa
++a;         // Increment (add 1)
--a;         // Decrement (subtract 1)
√a;          // Square root
sqrt a;      // Square root, but easier to type
abs a;       // Absolute value (always makes positive number)
floor a;     // Round towards zero
ceil a;      // Round away from zero
round a;     // Round to nearest whole number
int a;       // Effectively the same as floor

// Unary suffix operators
a++;         // Increment (add 1)
a--;         // Decrement (subtract 1)
```

The increment and decrement operators act slightly differently depending on
if they occur before or after a number.

```zzs
let a := 3;
let b := 10 + a++;

say a;  // 4
say b;  // 13: the addition happened before a was incremented
```

Versus:

```zzs
let a := 3;
let b := 10 + ++a;

say a;  // 4
say b;  // 14: the addition happened after a was incremented
```

The `floor` and `ceil` operators also have "circumfix" versions. This means
an operator which "wraps around" a value:

```zzs
let x:= ⌊ 1.8 + 0.5 ⌋; // circumfix floor
say x;  // says 2

let y:= ⌈ 1.8 + 0.5 ⌉; // circumfix ceil
say y;  // says 3
```

It is interesting to compare them to the unary prefix versions:

```zzs
let x:= floor 1.8 + 0.5;
say x;  // says 1.5

let y:= ceil 1.8 + 0.5;
say y;  // says 2.5
```

Why the difference? Prefix and suffix operators "bind tightly" to the value
they're operating on. Which means in `floor 1.8 + 0.5` the `floor` only
applies to the `1.8` (resulting in `1`) and the `0.5` is added on afterwards.
With circumfix, the beginning and end markers show exactly what the operator
applies to.

It is still possible to achieve this behaviour with `floor` and `ceil`
though, by wrapping what you want the operator to see with parentheses.

```zzs
let x:= floor( 1.8 + 0.5 ); // fake circumfix floor
say x;  // says 2

let y:= ceil( 1.8 + 0.5 ); // fake circumfix ceil
say y;  // says 3
```

### Expressions

The examples above illustrate something important. Although we've been
talking about operators being between two numbers, or being a prefix or
suffix for a number, in reality they operate on *expressions*.

In `ceil( 1.8 + 0.5 )`, the `ceil` doesn't just apply to the number `1.8`
but to the result of the whole expression `1.8 + 0.5`.

An expression is anything that evaluates to a value. Expressions can
be combined almost infinitely.

```zzs
let a := ( 1 + 2 ) × 10;  // 30
let b := 1 + ( 2 × 10 );  // 21

// also 21, because times is higher precedence than plus.
let c := 1 + 2 × 10;

let d := 2 × ( 1 + 2 × 10 ) + 10;  // 52
```


## 2.3 Numeric comparison operators

The operators we have looked at so far take one or two numbers and give you
a result which is also a number. The result of `3 + 5` is `8`. Comparison
operators instead mostly return a "truth value" or "boolean".

```zzs
if ( month_day = 1 ) {
	say "A new month begins...";
}
```

The `=` sign is a comparison operator. In particular it's the numeric equality
comparison operator: it tests that the two values on either side of it are
numerically equal. You will often use comparison operators in `if` conditions
like the last example.

Many programming languages use `=` as their assignment operator, used to set
a variable to a new value. We have already seen that ZuzuScript uses `:=`
for the assignment operator, freeing up the humble equals sign for its
normal mathematical use.

Other numeric comparison operators are:

```zzs
a = b;     // a equals b
a ≠ b;     // a is not equal to b
a <=> b;   // a is not equal to b, but easier to type
a < b;     // a is less than b
a > b;     // a is greater than b
a ≤ b;     // a is less than or equal to b
a <= b;    // a is less than or equal to b, but easier to type
a ≥ b;     // a is greater than or equal to b
a >= b;    // a is greater than or equal to b, but easier to type
```

The `<=>` operator is actually an interesting. While most of these comparison
operators return a simple true or false (including `≠`!), `<=>` returns:

* the number -1 if a is less than b
* the number 0 if a is equals to b
* the number 1 if a is greater than b

Because the number 0 is continued false and all other numbers are considered
true, this has the same affect as `≠`. The additional information about
which number is bigger comes in handy later if you are needing to sort a
list of numbers in order.

The `<=>` operator can also be written as `≶` or `≷`.


## 2.4 Numeric coercion

One important thing to know about Zuzu's operators is that they usually know
what data type they are supposed to operate on. For example, `+` knows that
its purpose is to add numbers, so it assumes that the values to its left
and right are both data type `Number`.

An interesting thing happens if you try to add things which are not numbers.
ZuzuScript will try to convert (or "coerce") them to numbers. An error will
occur if either of the values cannot be coerced to numbers.

These are the rules ZuzuScript uses to coerce values to numbers:

- anything which is already a `Number` stays that way
- a `String` that looks like a number, such as `"5"` is interpreted as a number
  (this includes scientific notation and the `0x`/`0b`/`0o` prefixes, and in
  Strings the markers may be either case, so `"1e3"` and `"0X1F"` coerce fine)
- the special value `null` is treated as 0
- the special value `false` is treated as 0
- the special value `true` is treated as 1
- if an `Object` (we will learn about objects later) has a `to_Number` method, that method will be called
- any other value cannot be coerced and will result in an error

This is different from some programming languages like JavaScript, where `+` on
two strings will concatenate them together.

```javascript
// JavaScript example
console.log( "1" + "2" );  // 12
```

```zzs
// ZuzuScript example
say "1" + "2";  // 3
```

Do you remember the unary prefix identity operator `+` which we said
"does nothing much"?

```zzs
let n := +x;
```

Because it's a numeric operator, this means `n` will now definitely be a
`Number`, even if `x` might not have been.


## 2.5 The `std/math` module

Although the built-in operators are enough for most programs that deal with
basic maths, sometimes you will need access to more powerful mathematical
functionality. For this, there is the `std/math` module.


### Using pi

```zzs
from std/math import π;

let radius        := 10;
let circumference := 2 × π × radius;

say circumference;
```

Here we're using a module. When we use a module, we need to tell it what
things to import from the module. Here we've imported `π`. A module's
documentation should tell you what things are available to import.

You can also just import everything:

```zzs
from std/math import *;

let radius        := 10;
let circumference := 2 × π × radius;
```

This will import everything the module makes available for export, except
things whose names start with an underscore, which are generally used for
the module's internals and other stuff you don't want to think about. It
is still possible to import things that start with an underscore, but you
need to list them explicitly: they are not imported by `import *`.

You can rename your imports too:

```zzs
from std/math import π as blueberry_pi;

let radius        := 10;
let circumference := 2 × blueberry_pi × radius;
```

### Trigonometry

```zzs
from std/math import Math;

let β := Math.deg2rad(30);    // 30 degree angle in radians
say Math.sin(β);
```

Notice we can use Greek letters in variable names. ZuzuScript accepts any
Unicode "word characters" as variable names: that's mostly letters (in any
alphabet, not just the Latin alphabet), numeric digits, and underscore.
However, you cannot name a variable just a single underscore `_` (for reasons
covered in the next chapter) and the variable name must start with a non-digit.

Another thing you might notice is the dot (`.`) operator. Dots signify a
method call. The thing on the left of the dot is an object or class, and
the thing on the right is a "method" or something you want to do with the
object/class.

So in `Math.sin(β)`, `Math` is the object or class (in this case class, but
it doesn't really matter here) that we wish to do something with, and `sin(β)`
is what we want to do.

### Random numbers

It's often useful to generate random numbers.

```zzs
from std/math import Math;

// Roll a 6-sided die
let result := ceil Math.rand(6);
```

`Math.rand(max)` will give you a random number between 0 and the maximum.
This is not necessarily a whole number, so might be, for example 5.51.
The `ceil` operator here will round up to a whole number, so we end up with
the whole numbers 1 to 6 as possible results.

If no `max` is given, the maximum is assumed to be 1. This is often useful.
For example, if you want a 5% chance of something happening:

```zzs
from std/math import Math;

if ( Math.rand() < 0.05 ) {
	say "Zia, something is happening!";
}
```

We can actually make that a bit tidier. In ZuzuScript if a method call has
empty parentheses like `Math.rand()` we can omit them like `Math.rand`.

```zzs
from std/math import Math;

if ( Math.rand < 0.05 ) {
	say "Help me Zia, it's happening again!";
}
```

## 2.6 Recap

Now we've learnt about:

- the `Number` datatype and what data types are
- that variables can be typed (`let Number x := 1`)
- mathematical operators (like `+`, `×`, and `floor`)
- what expressions are
- numeric comparison operators (like `=`, `>`, and `<=>`)
- the difference between unary prefix, unary suffix, binary infix, and circumfix operators
- how ZuzuScript operators coerce things
- the `std/math` module

We've also learnt a little bit about how method calls work and how to use
modules. Those details will return later when scripts become larger, but
for now the important thing is that expressions can already do real work.

In Chapter 3, we will introduce another important basic data type: strings.
