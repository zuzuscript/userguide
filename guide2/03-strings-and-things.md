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
```

## 3.2 The `BinaryString` data type

The `BinnaryString` data type is for sequences of bytes. To specify one,
use single quotes.

```zzs
let greeting := 'Hello world';
```