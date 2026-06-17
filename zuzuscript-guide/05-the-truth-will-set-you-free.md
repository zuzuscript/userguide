# Chapter 5: The Truth Will Set You Free

<img src="https://zuzulang.org/img/zia-booleans.jpeg" alt="Zia learning about booleans." class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />

Chapter 4 gave us collections: values with shape.

Now we need the values that help programs make decisions about that shape.

This chapter will introduce you to the final two basic data types in
ZuzuScript: `Null` and `Boolean`.

## 5.1 `Null`

The `Null` data type only has one possible value: `null`.

The `null` value is the value that any variable has before you assign it
a value. It can also be assigned explicitly.

```zzs
let x;
let y := null;

say typeof x;  // says Null
say typeof y;  // says Null
```

It's not an error or bad practice for some of your variables to simply
be `null`.

But there's not a lot of interesting things to say about `null`. So we will
move on to…

## 5.2 `Boolean`

The `Boolean` data type has two possible values: `true` and `false`.

```zzs
let sky_is_blue           := true;
let vaccines_cause_autism := false;

// Unicode aliases:
let raccoons_are_cute     := ⊤;  // true
let zuzuscript_is_hard    := ⊥;  // false
```

We've already encountered booleans: most comparison operators (`=`, `eq`,
`subsetof`, etc) return them.

```zzs
let is_admin = ( username eq "zia" );

if ( is_admin ) {
	say "Good morning, Zia!";
}
```


## 5.3 Boolean operators

There are a few operators for working with booleans

```zzs
// NOT
¬ a;       // The opposite of a
not a;     // Same, but easier to type
!a;        // Same, but easier to type and shorter

// AND
a ⋀ b;     // True only if a and b are both true
a and b;   // Same, but easier to type

// NAND
a ⊼ b;     // False only if a and b are both true
a nand b;  // Same, but easier to type

// XOR
a ⊻ b;     // True if exactly one of a and b is true
a xor b;   // Same, but easier to type

// OR
a ⋁ b;     // True if at least one of a and b is true
a or b;    // Same, but easier to type
```

Combining these operators allow you to build up some quite complex logic.

```zzs
if ( ( is_weekend or is_holiday ) and ( temperature > 30 ) ) {
	go_to( "the beach" );
}
```

## 5.4 Boolean coercion

Much like how numeric operators coerce the values they operate on to numbers
and string operators coerce values to string, boolean operators coerce
values to `Boolean`.

- `null` is coerced to `false`. 
- `0` is coerced to `false`.
- the empty string is coerced to `false`.
- empty collections are coerced to `false`.
- objects with a `to_Boolean` method are coerced by calling the method.
- everything else is coerced to `true`.

One key thing to notice: the number `0` is coerced to false, but the string
`"0"` is coerced to true. 


## 5.5 Value-preserving Boolean operators

The Boolean operators above always return Boolean values: `true` or `false`.
Sometimes you want to make the same kind of logical choice, but keep one of
the original values.

That is what the value-preserving Boolean operators are for. They use the
same names with a `?` suffix.

```zzs
let title := custom_title or? "Untitled";
```

This is like `or`, but instead of returning `true`, it returns the first
truthy value. If `custom_title` is falsey, it returns `"Untitled"`.

The two most useful ones are `and?` and `or?`.

```zzs
user and? user.name;       // null if user is null, otherwise user.name
name or? "Anonymous";      // name if name is truthy, otherwise "Anonymous"
```

`and?` returns its left value when that value is falsey. Otherwise it returns
the right value. Like `and`, it does not evaluate the right side when the
left side is falsey.

`or?` returns its left value when that value is truthy. Otherwise it returns
the right value. Like `or`, it does not evaluate the right side when the left
side is truthy.

There are also value-preserving forms of `xor` and `nand`.

```zzs
a xor? b;   // returns the one truthy value, or false
a nand? b;  // value-preserving form of nand
```

`xor?` is useful when exactly one of two values should be present. If both
are truthy, or both are falsey, it returns `false`. If exactly one is truthy,
it returns that original value.

`nand?` is less common, but follows the same naming pattern. It is the
value-preserving form of `nand`: when `nand` would be false because both
values are truthy, `nand?` returns `false`; in the case where the right side
is the only truthy value, it preserves and returns that right-side value.

The Unicode spellings work too: `⋀?`, `⋁?`, `⊻?`, and `⊼?`. In ordinary
code, the word spellings are usually easier to type and read.


## 5.6 The ternary operator

The ternary operator is a widely used logical operator that many programming
languages have. The operators we've looked at so far have been unary (working
on one value) or binary (working on two values). The ternary operator works
on three values.

This operator uses the form `c ? a : b`. When `c` is true, the result is `a`.
When `c` is false, the result is `b`.

```zzs
use_template( is_admin ? "admin-dashboard.html" : "dashboard.html" );
```

It can be thought of as a concise and convenient "if/then/else" expression.

It's possible to chain ternary operators.

```zzs
let tmpl := is_admin ? "admin.html" : ( logged_in ? "user.html" : "anon.html" );

// Or you could write it as...
let tmpl :=
	is_admin  ? "admin.html" : 
	logged_in ? "user.html" :
	"anon.html";
```


## 5.7 The Elvis Operator

<img src="https://zuzulang.org/img/zia-elvis.jpeg" alt="Zia, for one night only." class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />

One common use of the ternary operator is something like this:

```zzs
let display_name = name ? name : "Anon";
```

This usage is subtly wrong, or at least often not what you want. Often
what we want it to check if `name` is `null`. Something like:

```zzs
let display_name = is_null(name) ? name : "Anon";
```

ZuzuScript provides a shortcut for this. Imagine collapsing the `?` and `:`
of the ternary operator into each other.

```zzs
let display_name = name ?: "Anon";
```

This operator is known affectionately as "the Elvis operator".

It is the only operator that is primarily associated with the `Null`
data type.

### Elvis assignment

The `?:=` operator assigns a value only when the variable currently contains
`null`. If the variable already contains a non-null value, it is left alone.

```zzs
let display_name;

display_name ?:= "Anon";
display_name ?:= "Zia";

say display_name;  // says "Anon"
```

This is useful for defaults. You can set a fallback without overwriting a
value that was already provided by the user, a config file, or another part
of the program.

## 5.8 Type-Aware comparison operators

We've covered all of Zuzu's basic builtin types (`Number`, `String`,
`BinaryString`, `Null`, and `Boolean`) as well as collection types
(`Array`, `Dict`, `Set`, `Bag`, and `PairList`). 

We've seen a few different per-type equality operators:

- the `=` operator for numbers,
- the `eq` and `eqi` operators for strings and binary strings, and
- the `⊂⊃` or `equivalentof` operator for collections

But ZuzuScript also offers a type-aware comparison operator: `a ≡ b`.
This operator:

- if `a` and `b` are different types, then is false.
- if `a` and `b` are both null, then is true.
- if `a` and `b` are both booleans, then checks they have the same value.
- if `a` and `b` are both numbers, then compares them with `=`.
- if `a` and `b` are both strings or binarystrings, then compares them with `eq`.
- if `a` and `b` are both collections, then compares them with `⊂⊃`.
- if `a` and `b` are both objects, then checks they are the same object.

It is your best choice for strictly testing equality.

It has an alternative spelling and an opposite too.

```zzs
a ≡ b;    // Type-aware equality
a == b;   // Same, but easier to type

a ≢ b;    // Type-aware inequality
a != b;   // Same, but easier to type
```


## 5.9 Recap

In this chapter, we covered:

- the `Null` and `Boolean` data types
- boolean (logical) operators
- coercion to boolean
- value-preserving Boolean operators
- the ternary and Elvis operators
- type-aware equality operators

In Chapter 6, we will put all this learning to work and write some simple
practical scripts while learning about control flow.
Booleans are the bridge: they are what conditions, loops, and choices listen
to.
