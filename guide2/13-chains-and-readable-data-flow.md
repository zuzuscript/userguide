# Chapter 13: Chains and Readable Data Flow

Chapter 12 was about reaching into nested data.

This chapter is about making nested expressions readable.

Many languages encourage a style where the innermost operation happens
first, but appears deepest inside the code. That can make real programs
feel backwards.

For example, in JavaScript you might see something like this:

```javascript
const html = renderProfile(sortByScore(filterActive(parseJson(fetchText(url)), 3)));
```

The data actually flows like this:

1. fetch text from `url`,
2. parse the JSON,
3. keep profiles active within last three months,
4. sort them,
5. render HTML.

But the code makes you start reading in the middle.

ZuzuScript's chain operators let you write that flow in order.

This chapter covers:

- the forward chain operator `▷`,
- the placeholder `^^`,
- how chains replace deeply nested calls,
- when to keep a normal local variable instead,
- the reverse chain operator `◁`,
- and the ASCII aliases `|>` and `<|`.


## 13.1 The forward chain operator

The operator you will usually use is `▷`.

It evaluates the expression on the left, makes that value available as
`^^` on the right, and returns the value of the right-hand expression.

```zzs
let label := "  Sleepy Raccoon  "
	▷ trim(^^)
	▷ kebab(^^)
	▷ "friend-" _ ^^;

say label;                  // friend-sleepy-raccoon
```

Read it from top to bottom:

1. start with the string,
2. trim it,
3. turn it into kebab-case,
4. add a prefix.

The value of the whole chain is the value produced by the last step.

This feature of ZuzuScript is inspired by the Unix shell, where
pipes can be used to connect commands so that the output of one becomes
the input of the next.

```sh
cat access.log | grep 404 | sort | uniq -c
```

If you have worked at the command line on Linux/Unix, this concept should
immediately feel familiar and you may find yourself asking "why don't all
programming languages have this?"


## 13.2 What `^^` means

Inside the right side of a chain step, `^^` means "the value that has
arrived here".

```zzs
let doubled := 21 ▷ ^^ * 2;
say doubled;                // 42
```

You can use `^^` more than once:

```zzs
let square := 7 ▷ ^^ × ^^;
say square;                 // 49
```

`^^` is const-like. You can read it, pass it to functions, index into it,
or use it in expressions, but you do not assign to it.

```zzs
let next := 4 ▷ ^^ + 1;

^^ := 5;                 // invalid
```

Each step gets its own `^^`. A later step receives the result of the
previous step.


## 13.3 A JavaScript-shaped example

Here is a typical nested JavaScript expression:

```javascript
const rows = sortRows(projectRows(normalizeUsers(parseJson(readText(path)), true)));
```

It is doing ordinary work, but the shape is hard to scan. The first thing
that happens, `readText(path)`, is buried at the far right. It's not immediately
clear which function that `true` is being passed to.

In ZuzuScript, write the data flow in the order it happens:

```zzs
let rows := path
	▷ read_text(^^)
	▷ parse_json(^^)
	▷ normalize_users(^^, true)
	▷ project_rows(^^)
	▷ sort_rows(^^);
```

The names here are application functions. The point is the shape:

- each line says one thing,
- the input to each line is `^^`,
- the final value is assigned to `rows`.

This is especially useful when every function has one obvious data input.

It's even possible to include that final assignment in the chain:

```zzs
path 
	▷ read_text()
	▷ parse_json(^^)
	▷ normalize_users(^^, true)
	▷ project_rows(^^)
	▷ sort_rows(^^)
	▷ ( let rows := ^^ );
```


## 13.4 A Python-shaped example

Python code often uses small functions well, but deeply nested calls can
still become inside-out:

```python
summary = format_summary(round(mean(extract_scores(load_json(read_file(path))))))
```

In ZuzuScript, the same pipeline can be written as a chain:

```zzs
let summary := path
	▷ read_file(^^)
	▷ load_json(^^)
	▷ extract_scores(^^)
	▷ mean(^^)
	▷ round(^^)
	▷ format_summary(^^);
```

Chains work best when each step transforms the previous value into the
next value.

If a step needs another argument, keep using normal function-call syntax:

```zzs
let top_scores := path
	▷ read_file(^^)
	▷ load_json(^^)
	▷ extract_scores(^^)
	▷ take_first( 10, ^^ )
	▷ format_scores( ^^, "compact" );
```

The placeholder can appear anywhere in the call. It does not need to be
the first argument.


## 13.5 A PHP-shaped example

In PHP, nested text processing often grows from the inside out:

```php
$preview = htmlentities(substr(trim($comment), 0, 80), ENT_QUOTES);
```

In ZuzuScript, the same idea can be written as a readable sequence:

```zzs
let preview := comment
	▷ trim(^^)
	▷ substr( ^^, 0, 80 )
	▷ escape_html(^^);
```

That version makes the order of operations visible:

- clean up the input,
- take a preview-length slice,
- escape it for HTML.

The chain does not hide any work. It just removes the need to read nested
parentheses from the inside out.


## 13.6 Chains with path queries

Chapter 12 introduced path operators. They are ordinary expressions, so
they fit well inside chains.

```zzs
let sleepy_names := den
	▷ ^^ @@ "friends/*[mood eq 'sleepy']/name"
	▷ ^^.sortstr()
	▷ join( ", ", ^^ );
```

That reads as:

1. start with `den`,
2. query all sleepy friend names,
3. sort the names,
4. join them for display.

This is clearer than nesting those operations:

```zzs
let sleepy_names := join(
	", ",
	( den @@ "friends/*[mood eq \"sleepy\"]/name" ).sortstr()
);
```

Both versions are valid. The chain version is easier to extend one step at
a time.


## 13.7 Chains with `default`

The `default` operator from Chapter 4 also fits naturally into chains when
you are preparing option-like data.

```zzs
let settings := raw_settings
	▷ ^^ default {
		light: "dim",
		snack: "berries",
		blanket: true,
	}
	▷ validate_settings(^^);
```

The first step fills in missing keys. The second step validates the
completed settings Dict.

Use parentheses if a step is visually busy:

```zzs
let settings := raw_settings
	▷ ( ^^ default defaults )
	▷ validate_settings(^^);
```


## 13.8 When to stop chaining

Chains are for readability, not for proving how many operations can fit in
one expression.

This is fine:

```zzs
let names := den
	▷ ^^ @@ "friends/*/name"
	▷ ^^.sortstr();
```

This is probably too much:

```zzs
let output := den
	▷ ^^ @@ "friends/*[mood ne 'busy' and naps > 0]/name"
	▷ ^^.sortstr()
	▷ join( ", ", ^^ )
	▷ "[" _ ^^ _ "]"
	▷ render_badge( ^^, theme default { colour: "green" } )
	▷ cache_and_return( cache, "friend-badge", ^^ );
```

At that point, split out names:

```zzs
let available_names := den @@ "friends/*[mood ne 'busy' and naps > 0]/name";
let sorted_names := available_names.sortstr();
let label := join( ", ", sorted_names );

let output := label
	▷ `[${^^}]`
	▷ render_badge( ^^, theme default { colour: "green" } )
	▷ cache_and_return( cache, "friend-badge", ^^ );
```

Use a chain when it makes the data flow easier to see. Use named
intermediate values when they explain the program better.


## 13.9 `^^` is local to the step

The placeholder is introduced by the chain expression. It is not an
ordinary variable you declare.

```zzs
let result := "Zia"
	▷ ^^ _ " is sleepy"
	▷ "[" _ ^^ _ "]";

say result;                 // [Zia is sleepy]
```

The second step receives the whole result of the first step:

```zzs
// Step 1 receives: "Zia"
// Step 1 returns:  "Zia is sleepy"
// Step 2 receives: "Zia is sleepy"
// Step 2 returns:  "[Zia is sleepy]"
```

If you need both an early value and a later value, give the early value a
name before the chain:

```zzs
let original := "Zia";

let result := original
	▷ ^^ _ " is sleepy"
	▷ `${original}: ${^^}`;
```

That is clearer than trying to make one placeholder do two jobs.


## 13.10 The reverse chain operator

The reverse chain operator is `◁`.

It evaluates the expression on the right, makes that value available as
`^^` on the left, and returns the value of the left-hand expression.

```zzs
let label := trim(^^) ◁ "  Zia  ";
say label;                  // Zia
```

You can use it when the final shape is easier to see on the left:

```zzs
let message := "heard: " _ ^^ ◁ "Zenia is awake";
```

You can also chain it right-to-left:

```zzs
let label := "friend-" _ ^^ ◁ kebab(^^) ◁ "Sleepy Raccoon";
say label;                  // friend-sleepy-raccoon
```

Most ZuzuScript code is easier to read with `▷`. Reach for `◁` when it
genuinely makes the expression read better from right to left.


## 13.11 ASCII aliases

The Unicode chain operators are the canonical spellings:

- `▷` for forward chains,
- `◁` for reverse chains.

There are ASCII aliases for environments where entering Unicode is awkward:

- `|>` means `▷`,
- `<|` means `◁`.

```zzs
let label := "  Sleepy Raccoon  "
	|> trim(^^)
	|> kebab(^^);
```

Generated ZuzuScript source prefers the Unicode spellings, so this guide
uses `▷` and `◁`.


## 13.12 Mixing directions

Do not mix `▷` and `◁` at the same unparenthesized chain level.

This is confusing and rejected:

```zzs
// Bad shape:
value ▷ step_one(^^) ◁ other_value
```

If you really need both directions, add parentheses so the grouping is
explicit:

```zzs
let result := ( value ▷ step_one(^^) )
	▷ combine( ^^, other_value );
```

Most of the time, choosing one direction for the whole expression is the
right answer.


## 13.13 Internals

The Rust implementation of ZuzuScript will often internally rewrite
chains as standard nested function calls.

For example, using `--dump-zuzu` to see how Rust has interpreted
some code:

```sh
zuzu-rust --dump-zuzu -e'function bar; 1 ▷ bar(^^)'
```

The output is:

```zzs
function bar;
bar( 1 );
```


## 13.14 Chapter recap

You now have the practical model for chains:

- `left ▷ right` evaluates `left`, binds it to `^^` in `right`, and
  returns `right`,
- `left ◁ right` evaluates `right`, binds it to `^^` in `left`, and
  returns `left`,
- `^^` is a const-like placeholder for the value at the current step,
- chains are good for linear data transformations,
- named intermediate variables are better when they explain intent,
- `|>` and `<|` are ASCII aliases, but `▷` and `◁` are canonical.

Next up: concurrency, where work starts happening in more than one place.
Chains make one path through data easier to read; tasks and workers help
when there are several paths in progress at once.
