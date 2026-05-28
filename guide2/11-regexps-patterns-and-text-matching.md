# Chapter 11: Regexps, Patterns, and Text Matching

Chapter 10 showed how to split code into modules.

Now we are going to spend a chapter on a small but powerful tool:
regular expressions, usually shortened to **regexps**.

A regexp is a pattern for text.

You use regexps when you want to ask questions like:

- does this string contain a number?
- does this line look like `name:age`?
- where are the words in this paragraph?
- can I replace all matching pieces with something else?

If you have used regexps in another language, some of this will feel
familiar. If you have not, do not worry. We will build up from ordinary
string matching to captures and substitutions.

In this chapter we will cover:

- regexp literals such as `/raccoon/`,
- matching with `expr ~ /regexp/`,
- captures and the shape of match results,
- the `/i` and `/g` flags,
- substitution with `var ~= /regexp/ → expr`,
- `do` blocks,
- regexp helper functions from `std/string`,
- converting values to regexps,
- and writing regexps that behave well across ZuzuScript runtimes.


## 11.1 A first regexp

The simplest regexp is just the text you want to find.

```zzs
let note := "Zia is a sleepy raccoon.";

if ( note ~ /sleepy/ ) {
	say "nap detected";
}
```

The operator is `~`.

Read this:

```zzs
note ~ /sleepy/
```

as:

> Does `note` match the regexp `/sleepy/`?

If it matches, the result is truthy.
If it does not match, the result is `false`.

```zzs
say "Zia" ~ /Zia/;      // match result
say "Zia" ~ /Zenia/;    // false
```

The slashes mark a regexp literal, a little like quotes mark a string
literal.

```zzs
/sleepy/
```

means "the pattern `sleepy`".


## 11.2 Matching is not the same as equality

This is equality:

```zzs
"Zia" eq "Zia"
```

It asks whether the whole string is exactly the same text.

This is regexp matching:

```zzs
"Zia is sleepy" ~ /sleepy/
```

It asks whether the pattern appears somewhere in the string.

So this matches:

```zzs
"Zia is sleepy" ~ /Zia/
```

and this also matches:

```zzs
"Zia is sleepy" ~ /sleepy/
```

To require the whole string to match, use anchors.

```zzs
"Zia" ~ /^Zia$/;           // matches
"Zia is sleepy" ~ /^Zia$/; // false
```

`^` means "start of the string".
`$` means "end of the string".


## 11.3 A few useful pattern pieces

Regexps have small pieces of syntax for common text shapes.

Here are enough to start:

| Pattern | Meaning |
|---|---|
| `.` | any single character |
| `[abc]` | one character: `a`, `b`, or `c` |
| `[A-Z]` | one uppercase ASCII letter |
| `[0-9]` | one ASCII digit |
| `*` | zero or more of the previous thing |
| `+` | one or more of the previous thing |
| `?` | zero or one of the previous thing |
| `{3}` | exactly three of the previous thing |
| `{2,5}` | two to five of the previous thing |
| `(...)` | a capture group |
| `a|b` | either `a` or `b` |

Examples:

```zzs
"Zia" ~ /Z.a/;          // Z, any character, a
"Zenia" ~ /Z[ae]nia/;   // Za... or Ze...
"Zachary" ~ /^Za/;      // starts with Za
"room 42" ~ /[0-9]+/;   // one or more digits
```

When a character has special meaning in a regexp, put `\` before it to
mean the literal character.

```zzs
"3.14" ~ /3\.14/;       // literal dot
"a/b" ~ /a\/b/;         // literal slash inside /.../
```

The slash needs escaping because slash also ends the regexp literal.


## 11.4 Match results and captures

The result of `~` is more useful than just true or false.

When a match succeeds, it returns an array:

- `m[0]` is the full matched text,
- `m[1]` is the first capture,
- `m[2]` is the second capture,
- and so on.

```zzs
let m := "Zia:12" ~ /^([A-Za-z]+):([0-9]+)$/;

if ( m ) {
	say m[0];  // Zia:12
	say m[1];  // Zia
	say m[2];  // 12
}
```

Parentheses create captures.

In this pattern:

```zzs
/^([A-Za-z]+):([0-9]+)$/
```

there are two captures:

1. `([A-Za-z]+)` captures the name,
2. `([0-9]+)` captures the number.

That is why `m[1]` is `"Zia"` and `m[2]` is `"12"`.

A common pattern is:

```zzs
let line := "Zenia:awake";
let m := line ~ /^([A-Za-z]+):([A-Za-z]+)$/;

if ( m ) {
	say m[1] _ " is " _ m[2];
}
```

Always check the match result before reading captures unless you already
know the text matched.

You can do this in one step like this:

```zzs
let line := "Zia:lazy";

if ( let m := line ~ /^([A-Za-z]+):([A-Za-z]+)$/ ) {
	say m[1] _ " is " _ m[2];
}
```


## 11.5 The `/i` flag: case-insensitive matching

Put `i` after the closing slash to ignore case.

```zzs
"zia" ~ /ZIA/i;      // matches
"Zenia" ~ /zenia/i;  // matches
```

For cross-platform code, be careful with non-ASCII case matching.
The runtimes use different underlying regexp engines, and Unicode
case-folding details may vary. If you need fully predictable behaviour,
normalize text yourself or keep case-insensitive patterns to ASCII.


## 11.6 The `/g` flag: all matches

Without `/g`, `~` returns the first match.

```zzs
let first := "Zia Zenia Zachary" ~ /Z[a-z]+/;

say first[0];  // Zia
```

With `/g`, `~` returns all matches.

The result is an array of match arrays:

```zzs
let all := "Zia Zenia Zachary" ~ /Z[a-z]+/g;

say all[0][0];  // Zia
say all[1][0];  // Zenia
say all[2][0];  // Zachary
```

Captures still work. Each match has its own capture array.

```zzs
let pairs := "Zia=12 Zenia=9 Zachary=14" ~ /([A-Za-z]+)=([0-9]+)/g;

say pairs[0][1];  // Zia
say pairs[0][2];  // 12
say pairs[2][1];  // Zachary
say pairs[2][2];  // 14
```

If a `/g` match finds nothing, it returns `false`.

You can combine flags:

```zzs
let names := "zia ZENIA Zachary" ~ /z[a-z]+/ig;
```

That means "find all matches, ignoring case".


## 11.7 Interpolation inside regexp literals

Regexp literals can interpolate values with `${...}`, like template
strings.

```zzs
let digits := "[0-9]+";
let m := "item-123" ~ /item-${digits}/;

say m[0];  // item-123
```

Regexp escapes still work:

```zzs
"x9" ~ /x\d/;
```

and escaped `${` is treated as literal text:

```zzs
"${name}" ~ /\$\{name\}/;
```

Be careful when interpolating user-supplied text.
If the text contains regexp punctuation such as `.`, `*`, `+`, or `[`,
it will be treated as regexp syntax. For user input, prefer ordinary
string functions such as `contains`, or escape the input before building
a regexp.

Use `quotemeta` from `std/string` when you need to interpolate literal
text into a regexp:

```zzs
from std/string import quotemeta;

let label := "Zia.+";
let line := "zia.+: sleepy raccoon";

if ( line ~ /^${quotemeta(label)}:/i ) {
	say "literal label matched";
}
```

Without `quotemeta`, the `.` and `+` in `label` would be regexp syntax.
With `quotemeta`, they are matched as ordinary characters.


## 11.8 Substitution with `~=`

Matching asks a question.

Substitution changes text.

The regexp substitution assignment syntax is `... ~= ... → ...`.
As it operates on three things, it's technically a ternary operator. (Just not
"*the* ternary operator".)

```zzs
let status := "Zia is awake";

status ~= /awake/ → "sleepy";

say status;  // Zia is sleepy
```

You may also write the arrow as `->`:

```zzs
status ~= /sleepy/ -> "very sleepy";
```

The general shape is:

```zzs
variable ~= /pattern/ → replacement_expression;
```

The left side must be assignable, like a variable.

The replacement expression is evaluated for each match that is replaced.
During that expression, a local variable named `m` contains the match
captures:

```zzs
let label := "Zia=12";

label ~= /([A-Za-z]+)=([0-9]+)/ → `${m[1]} has score ${m[2]}`;

say label;  // Zia has score 12
```

Here:

- `m[0]` is `"Zia=12"`,
- `m[1]` is `"Zia"`,
- `m[2]` is `"12"`.

Without `/g`, only the first match is replaced.

```zzs
let moods := "awake awake awake";

moods ~= /awake/ → "sleepy";

say moods;  // sleepy awake awake
```

With `/g`, every match is replaced.

```zzs
let moods := "awake awake awake";

moods ~= /awake/g → "sleepy";

say moods;  // sleepy sleepy sleepy
```


## 11.9 Replacement expressions can compute

The replacement does not need to be a plain string.
It can be any expression.

```zzs
let names := "zia zenia zachary";

names ~= /([a-z]+)/g → uc m[1];

say names;  // ZIA ZENIA ZACHARY
```

You can build replacement text from captures:

```zzs
let scores := "Zia:10 Zenia:20 Zachary:30";

scores ~= /([A-Za-z]+):([0-9]+)/g → `${m[1]}(${m[2]})`;

say scores;  // Zia(10) Zenia(20) Zachary(30)
```

The replacement expression has its own `m`.
It does not overwrite an outer variable named `m`.

```zzs
let m := "outside";
let text := "Zia";

text ~= /Zia/ → m[0] _ "!";

say text;  // Zia!
say m;     // outside
```


## 11.10 `do` blocks

Sometimes a replacement needs more than one step.

For that, use a `do` block.

A `do` block is a block used as an expression:

```zzs
let answer := do {
	let n := 40;
	n + 2;
};

say answer;  // 42
```

The value of a `do` block is the value of its last expression.

Use `do` when the language expects one expression, but you need local
statements to calculate that expression.

Although `do` blocks can be used almost anywhere you'd expect to find an
expression, regexp substitution is one especially handy place to use them.

```zzs
let text := "zia zenia zachary";

text ~= /([a-z]+)/g → do {
	let name := m[1];

	if ( name eq "zia" ) {
		"Zia the sleepy raccoon";
	}
	else {
		uc name;
	}
};

say text;  // Zia the sleepy raccoon ZENIA ZACHARY
```

Inside the replacement `do` block, `m` is still the current match.

You can also perform nested substitutions:

```zzs
let text := "zia zenia";

text ~= /([a-z]+)/g → do {
	let name := m[1];
	name ~= /^z/ → "Z";
	name;
};

say text;  // Zia Zenia
```

Use `do` blocks when the calculation is still local and readable.
If the replacement logic grows large, move it into a named function.


## 11.11 `std/string` regexp helpers

The `std/string` module provides helper functions that accept regexps.

```zzs
from std/string import
	search,
	matches,
	replace,
	split,
	pattern_to_regexp,
	quotemeta;
```

### `search`

`search(text, pattern, flags?)` returns the first matched text, or `null`
when nothing matches.

```zzs
from std/string import search;

say search( "Zia sleeps at 14:00", /[0-9]+:[0-9]+/ );  // 14:00
say search( "Zia sleeps", /[0-9]+/ );                  // null
```

This is simpler than `~` when you only want the matched text and do not
need captures.

### `matches`

`matches(text, pattern, flags?)` returns a Boolean.

```zzs
from std/string import matches;

if ( matches( "Zenia", /^Zen/ ) ) {
	say "Zenia is here.";
}
```

This is useful when you want a clear true/false helper.

### `replace`

`replace(text, pattern, replacement, flags?)` returns a changed string.
It does not modify the original variable.

```zzs
from std/string import replace;

let old := "Zia is awake";
let new := replace( old, /awake/, "sleepy" );

say old;  // Zia is awake
say new;  // Zia is sleepy
```

Pass `"g"` as the flags argument to replace all matches:

```zzs
from std/string import replace;

say replace( "Zia Zenia Zachary", /Z/, "z", "g" );
```

If the regexp literal already has `/i`, the extra flags can still add
`g`:

```zzs
from std/string import replace;

say replace( "zia ZIA", /zia/i, "Zia", "g" );  // Zia Zia
```

Use `~=` when replacement needs match captures and computation.
Consider `replace` when a direct replacement string is enough.

### `split`

`split(text, separator_or_regexp, limit?)` can split on a regexp.

```zzs
from std/string import split;

let parts := split( "Zia, Zenia; Zachary", /[,;]\s*/ );

say parts[0];  // Zia
say parts[1];  // Zenia
say parts[2];  // Zachary
```

### `pattern_to_regexp`

`pattern_to_regexp(pattern, case_insensitive?)` builds a `Regexp` value
from a string.

```zzs
from std/string import pattern_to_regexp, matches;

let rx := pattern_to_regexp( "^zia$", true );

say matches( "ZIA", rx );  // true
```

This is useful when the pattern comes from configuration.
Remember that the string is still regexp syntax, not plain text.

### `quotemeta`

`quotemeta(text)` escapes regexp metacharacters so text can be safely
inserted into a regexp as literal text.

```zzs
from std/string import quotemeta;

let label := "Zenia?";
let rx := /^${quotemeta(label)}:/i;

say "zenia?: awake" ~ rx;  // matches
say "zeniaa: awake" ~ rx;  // false
```

Use this when the label, prefix, or other pattern fragment comes from
outside your source code and should not be treated as regexp syntax.


## 11.12 Regexp coercion

The right side of `~` does not have to be a regexp literal.

If needed, `expr1 ~ expr2` coerces `expr2` to a regexp.

```zzs
let pattern := "Z[a-z]+";

say "Zenia" ~ pattern;  // matches
```

That is shorthand for "turn the right side into a regexp, then match".

This also means accidental regexp syntax matters:

```zzs
let pattern := "Z.";

"Zia" ~ pattern;  // matches Zi, because . means any character
```

For explicit conversion, import `to_Regexp` from `std/internals`:

```zzs
from std/internals import to_Regexp;

let rx := to_Regexp( "Z[a-z]+" );

say "Zachary" ~ rx;
```

`to_Regexp` preserves existing regexps and compiles strings into regexp
values. It calls `to_String` on objects that provide that method. It
rejects values that cannot be sensibly converted, such as binary strings.

There is also an internal `to_Regexp_with_flags` helper, but normal code
usually reads better with regexp literals or `std/string.pattern_to_regexp`.


## 11.13 Literal text vs regexp syntax

A common beginner mistake is treating a regexp like a plain string search.

This pattern:

```zzs
/Z.a/
```

does not mean the literal text `Z.a`.
It means:

- `Z`,
- then any character,
- then `a`.

So it matches `"Zia"` and `"Z.a"`.

For a literal dot, escape it:

```zzs
/Z\.a/
```

The same applies to many punctuation characters:

```text
. * + ? ( ) [ ] { } ^ $ | \ /
```

When those characters should mean themselves, escape them.

For user-supplied text, consider avoiding regexps unless you really need
regexp behaviour.

```zzs
from std/string import contains;

if ( contains( "Zia.Zenia", "." ) ) {
	say "literal dot found";
}
```


## 11.14 Cross-platform regexp behaviour

ZuzuScript exposes the host implementation's regexp engine:

- `zuzu.pl` uses Perl regexps,
- `zuzu-js` uses ECMAScript `RegExp`,
- `zuzu-rust` uses Rust's `regex` crate.

That is powerful, but it means advanced regexp features are not perfectly
portable.

For cross-platform-safe regexps, prefer the common core:

- literal text,
- `.`,
- `^` and `$`,
- character classes like `[A-Za-z0-9_]`,
- grouping with `(...)`,
- non-capturing groups `(?:...)`,
- alternation with `|`,
- quantifiers such as `*`, `+`, `?`, `{3}`, and `{2,5}`,
- the `/i` and `/g` flags.

Be cautious with:

- lookahead and lookbehind,
- backreferences such as `\1`,
- named captures,
- engine-specific Unicode properties,
- inline flag syntax beyond simple cases,
- locale-sensitive or Unicode-heavy case-insensitive matching,
- exact meanings of `\w`, `\d`, and `\s` for non-ASCII text.

Rust's `regex` crate deliberately rejects some features that Perl and
ECMAScript may support, especially features that require backtracking such
as backreferences and look-around.

If code must run everywhere, keep patterns boring and explicit:

```zzs
// More portable.
let rx := /^[A-Za-z][A-Za-z0-9_]*$/;

// Less portable if you rely on Unicode or engine-specific details.
let broad := /^\w+$/;
```

For ASCII digits, prefer `[0-9]` when portability matters.
For ASCII whitespace, write the exact characters you intend when that
matters:

```zzs
/[ \t]+/
```

instead of assuming every engine treats every whitespace character the
same way.


## 11.15 Practical examples

### Parsing simple records

```zzs
let line := "Zia:sleepy";
let m := line ~ /^([A-Za-z]+):([A-Za-z]+)$/;

if ( m ) {
	say "name=" _ m[1];
	say "mood=" _ m[2];
}
```

### Finding every friend

```zzs
let story := "Zia naps. Zenia reads. Zachary makes tea.";
let names := story ~ /\bZ[a-z]+\b/g;

if ( names ) {
	for ( let match in names ) {
		say match[0];
	}
}
```

### Rewriting names

```zzs
let story := "Zia, Zenia, and Zachary";

story ~= /\b(Zia|Zenia|Zachary)\b/g → do {
	if ( m[1] eq "Zia" ) {
		"Zia the sleepy raccoon";
	}
	else {
		m[1] _ " the friend";
	}
};

say story;
```

### Validating a small identifier

```zzs
function is_identifier (text) {
	return text ~ /^[A-Za-z_][A-Za-z0-9_]*$/ ? true : false;
}

say is_identifier("Zia_1");  // true
say is_identifier("1_Zia");  // false
```


## 11.16 Chapter summary

You now know how to:

- write regexp literals with `/.../`,
- match text with `expr ~ /regexp/`,
- read match arrays and capture groups,
- use `/i` for case-insensitive matching,
- use `/g` for all matches or all substitutions,
- substitute with `var ~= /regexp/ → expr`,
- use `do` blocks when an expression needs local statements,
- use `search`, `matches`, `replace`, `split`, `pattern_to_regexp`, and
  `quotemeta` from `std/string`,
- use `to_Regexp` from `std/internals` for explicit conversion,
- and keep patterns portable across Perl, JavaScript, and Rust runtimes.

Regexps are sharp tools. Start with small patterns, test them with real
input, and keep cross-platform patterns deliberately simple.

Patterns help you find meaning inside text. Chapter 12 applies the same
idea to structured data, where the interesting value may be several arrays
and dictionaries deep.
