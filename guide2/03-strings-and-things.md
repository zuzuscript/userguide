# Chapter 3: Strings and Things

<img src="https://zuzulang.org/img/zia-strings.png" alt="All tangled up!" class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />

Strings can be thought of as pieces of text or collections of bytes. In fact,
these two ways of thinking of strings are the reason ZuzuScript has not one,
but two string types:

- `String` for character strings, chunks of text, etc.
- `BinaryString` for byte strings, binary data, file contents, etc.


## 3.1 The `String` data type

The `String` type is text (internally using UTF-8). To specify a `String`
use double quotes. This is the main string type that you will normally use for
most string-related things.

```zzs
let greeting := "Hello world!";
let alphabet := "αβγδεζηθικλμνξοπρστυφχψω";
let thespoon := "没有勺子。";

say typeof greeting;  // says "String"
```

### Escaping rules

Within a double-quoted string, certain special characters can be "escaped".

```zzs
let str := "String with a quote mark \" inside the string";
```

- Carriage returns and new lines can be escaped as `\r` and `\n`.
- Tabs can be escaped as `\t`.
- Slash and backslash can be escaped as `\/` and `\\`.
- The dollar sign can be escaped as `\$`.
- Quote marks and backticks can be escaped with a leading backslash too.
- `\xHH` where H is a hexadecimal digit is a binary octet.
- `\uHHHH` where H is a hexadecimal digit is a Unicode character.

The only characters which usually *need* to be escaped are backslash and
the closing quote character. But it may help clarity if you escape others
too: especially carriage return, new line, tab, and any surprising-looking
unicode characters.


### Multiline strings

If you have a long, multi-line string, you can use tripled double-quotes.

```zzs
let webpage := """
<html lang="en">
  <head>
    <title>Hello world</title>
  </head>
  <body>
    <p>Zia says hello!</p>
  </body>
</html>
""";

say webpage;
```

Within tripled quote marks, no escapes are supported.

Unlike Perl and PHP, variables are not interpolated in strings by default.
(But we will soon learn how to interpolate variables in strings.)


## 3.2 The `BinaryString` data type

The `BinnaryString` data type is for sequences of bytes. To specify one,
use single quotes.

```zzs
let greeting := 'Hello world';
say typeof greeting;  // says "BinaryString"
```

Like `String`, `BinaryString` allows escape characters and has a tripled
quote mark form. The `\uHHHH` escape is not supported in `BinaryString`.

The `BinaryString` data type is mostly useful when you need an exact
sequence of bytes, for example, when writing to a file or socket, or
when dealing with encrypting, decrypting, and signing data.


## 3.3 Template literals

ZuzuScript borrows the idea of template literals from JavaScript. They
use backticks as their quote marks, and within them you can use `${ ... }`
to insert a ZuzuScript expression.

```zzs
let product_id  := 144;
let product_url := `https://example.com/products/${product_id}`;

say product_url; // says https://example.com/products/144
```

The `${ ... }` syntax is not limited to variables but can include
arbitrary expressions.

```zzs
say `Three plus four is: ${ 3 + 4 }`;
```

Template literals have a multiline form too.

```zzs
let data := {
        lang:    "en",
        title:   "Hello World",
        content: "Zia says hello.",
};

let webpage := ```
<html lang="${ data{lang} }">
  <head>
    <title>${ data{title} }</title>
  </head>
  <body>
    <p>${ data{content} }</p>
  </body>
</html>```;

say webpage;
```


## 3.4 String operators

The following binary infix operators work on `String` and `BinaryString`
values:

```zzs
a _ b;         // Concatenate

a & b;         // Bitwise AND, BinaryString only
a | b;         // Bitwise OR,  BinaryString only
a ^ b;         // Bitwise XOR, BinaryString only
```

For most people, the `_` operator will be the most common of these. It
joins two strings together.

```zzs
let greeting := "Hello";
let greeted  := "world";

say greeting _ " " _ greeted;
```

Most programming languages use `+` or `.` to concatenate strings like this.
SQL uses `||`. ZuzuScript is unusual in using `_`.

You can concatenate two `String` values or two `BinaryString` values. If
you concatenate a `String` and `BinaryString` value together, the
`BinaryString` will first be "upgraded" to a `String`, assuming it was
in UTF-8 encoding.

Concatenation is often better implemented using templates though.

```zzs
let greeting := "Hello";
let greeted  := "world";

say `${greeting} ${greeted}`;
```

There are also a few prefix operators supported on strings:

```zzs
length a;      // Length of string
uc a;          // Uppercase, String only
lc a;          // Lowercase, String only
~a;            // Bitwise NOT, BinaryString only
```

The `length` operator will return the number of characters of a `String`
or the number of bytes of a `BinaryString`.


## 3.5 String comparison operators

Just as `Number` has a family of operators for comparing numbers,
`String` has a family of operators for comparing strings. Like with
concatenation, these can be used on a pair of `String` values or a
pair of `BinaryString` values, and if you try them on one of each,
the `BinaryString` will be "upgraded".

The most commonly used comparison is `eq` whichh tests if two strings
are equal.

```zzs
let is_admin := false;

if ( username eq "zia" ) {
	is_admin := true;
}
```

Other string comparison operators are:

```zzs
a eq b;    // a equals b
a ne b;    // a is not equal to b
a gt b;    // a is greater than b
a lt b;    // a is less than b
a ge b;    // a is greater than or equal to b
a le b;    // a is less than or equal to b
a cmp b;   // Three-way comparison: -1, 0, or 1.
```


### Case-insensitive comparison operators

It is quite common to compare values case-insensitively. The naive way
to do this is by converting both to lowercase (or both to uppercase) and
then comparing them:

```zzs
let owner := "Zia";
if ( lc(username) eq lc(owner) ) {
	...;
}
```

But ZuzuScript makes it easier by including a family of case-sensitive
comparison operators:

```zzs
let owner := "Zia";
if ( username eqi owner ) {
	...;
}
```

The full case-insensitive comparison operator family is:

```zzs
a eqi b;   // a equals b
a nei b;   // a is not equal to b
a gti b;   // a is greater than b
a lti b;   // a is less than b
a gei b;   // a is greater than or equal to b
a lei b;   // a is less than or equal to b
a cmpi b;  // Three-way comparison: -1, 0, or 1.
```

## 3.6 String coercion

Like how `+` coerces its values to numbers, Zuzu's string operators will
coerce values to strings.

These are the rules ZuzuScript uses to coerce values to strings:

- anything which is already a `String` or `BinaryString` stays that way
- a number will become the string representation of that number (`6` becomes `"6"`)
- the special value `null` is treated as `""`
- the special value `false` is treated as `"false"`
- the special value `true` is treated as `"true"`
- if an `Object` (we will learn about objects later) has a `to_String` method, that method will be called
- any other value cannot be coerced and will result in an error

```zzs
say 1 + 2 + 3; // says "6"
say 1 _ 2 _ 3; // says "123"
```


## 3.7 String indices and slices

It is possible to use `[...]` to index into a string and access a particular
character.

```zzs
let alphabet := "abcdefghijklmnopqrstuvwxyz";
say alphabet[0];        // says "a"
say alphabet[1];        // says "b"
say alphabet[2];        // says "c"
say alphabet[ 1 + 2 ];  // says "d"
say alphabet[-2];       // says "y" (counting back from the end)
```

String indices can also be assigned to:

```zzs
let alphabet := "abcdefghijklmnopqrstuvwxyz";
alphabet[1]  := "XXX";
say alphabet;   // says "aXXXcdefghijklmnopqrstuvwxyz"
```

It's possible to refer to a "slice" of a string using `[start:length]` too.

```zzs
let alphabet := "abcdefghijklmnopqrstuvwxyz";
say alphabet[5:3];      // says "fgh"
```

It's similarly possible to assign to slices:

```zzs
let alphabet := "abcdefghijklmnopqrstuvwxyz";
alphabet[20:1] := "";   // remove "u"
alphabet[8:0] := "u";   // insert "u" after "h"
say alphabet; // if I could rearrange the alphabet, I'd put U and I together
```


## 3.8 The `std/string` module

Just as numbers have `std/math` to provide additional functionality, strings
have `std/string`.

The `std/string` module includes useful functions for searching strings,
converting between characters and numbers, splitting strings, joining
strings, and common string formatting changes.

```zzs
from std/string import starts_with, ends_with;

let username := "zirconia";

if ( starts_with( username, "z" ) ) {

	say "It might be Zia!";

	if ( ends_with( username, "ia" ) ) {

		say "It must be Zia!";

		if ( username ne "zia" ) {
			say "I can't believe it wasn't Zia!";
		}
	}
}
```
