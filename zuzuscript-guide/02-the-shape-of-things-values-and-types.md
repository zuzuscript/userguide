# Chapter 2: The Shape of Things: Values & Types

In Chapter 1, you got ZuzuScript running, printed some output, and met
our permanently sleepy mascot engineer, Zia.

Now we move from “how to run code” to “what code is made of.”

In practice, every script is a value pipeline:

- read values,
- transform values,
- compare values,
- group values,
- emit values.

This chapter stays grounded in the **Perl implementation** as the
canonical runtime.


## 2.1 Why this chapter matters

Most early bugs are value-shape bugs, not syntax bugs.

For example:

- “Why did this comparison return false?”
- “Why did duplicates vanish here?”
- “Why did this thing count as true?”
- “Why did my dict key lookup return null?”

So this chapter builds a practical mental model for:

- primitive values,
- composite values,
- sets and bags (and how they differ),
- type-aware equality,
- mutation and binding,
- coercion (when it happens and when it does not).


## 2.2 Primitive values: your everyday atoms

### Numbers

Numbers are used for counting, arithmetic, and numeric comparison.

```zzs
let cups := 2;
let ratio := 9 ÷ 4;
let area := 3 × 7;

say cups;
say ratio;
say area;
```

Zuzu supports both ASCII and Unicode operator spellings for many
numeric operators. In this guide, when a Unicode symbol exists, we use
that symbol.

Examples:

- `×` (multiply),
- `÷` (divide),
- `≤`, `≥` (numeric comparisons),
- `≠` (numeric not-equal).

### Strings

Strings are text values.

```zzs
let mascot := "Zia";
let mood := "sleepy";

say `Mascot: ${mascot}, mood: ${mood}`;
```

Zuzu uses **dedicated string operators** for lexical string work:

- `eq`, `ne`, `gt`, `ge`, `lt`, `le`, `cmp`,
- plus case-insensitive forms like `eqi`, `cmpi`.

```zzs
say "abc" eq "abc";
say "A" eqi "a";
say "b" gt "a";
```

This separation from numeric operators removes a lot of ambiguity that
exists in other scripting languages.

### Booleans

Booleans are `true` and `false`.

```zzs
let caffeinated := true;
let napping := false;

if ( caffeinated and not napping ) {
  say "Zia can review pull requests.";
}
```

### Null

`null` means “no value”.

```zzs
let next_task := null;

if ( next_task == null ) {
  say "No queued task.";
}
```


## 2.3 Composite values: grouping data

### Arrays

Arrays are ordered sequences. Duplicates are allowed.

```zzs
let tasks := [ "lint", "test", "ship" ];
say tasks[0];
```

Use arrays when order or position matters.

### Dicts

Dicts hold key/value pairs.

```zzs
let zia := {
  name: "Zia",
  coffees: 2,
  team: "tooling",
};

say zia{name};
say zia{coffees};
```

Use dicts when data is field-oriented (named properties).


## 2.4 Sets and bags: correct literal syntax and semantics

This is where the previous draft often goes wrong, so let’s be precise.

## Sets

Set literal syntax:

- ASCII: `<< ... >>`
- Unicode quotes are also supported: `« ... »`

A set stores **unique membership**.

```zzs
let skills := « "coding", "reading", "coding" »;
say skills.length();     # 2
say skills.contains("coding");
```

Duplicates are deduplicated by set semantics.

## Bags (multisets)

Bag literal syntax:

- `<<< ... >>>`

A bag stores **multiplicity** (counts of repeated values).

```zzs
let snacks := <<< "cookie", "cookie", "tea" >>>;
say snacks.length();     # 3 total entries
say snacks.count("cookie");
```

### Fast intuition

- Set answers: “Is it present?”
- Bag answers: “How many times is it present?”

### Empty set literal

Zuzu also has an explicit empty-set literal:

```zzs
let none := ∅;
```


## 2.5 Collection operators (including Unicode forms)

Zuzu provides collection operators with both word and symbol spellings.

```zzs
let a := « 1, 2 »;
let b := « 2, 3 »;

let u1 := a union b;
let u2 := a ⋃ b;

let i1 := a intersection b;
let i2 := a ⋂ b;

let d1 := a \ << 2 >>;
let d2 := a ∖ << 2 >>;

say « 1, 2 » ⊂ « 1, 2, 3 »;
say « 1, 2, 3 » ⊃ « 1, 2 »;
say « 1, 2 » ⊂⊃ « 2, 1 »;
```

Membership operators also have Unicode forms:

```zzs
say 2 ∈ « 1, 2, 3 »;
say 4 ∉ « 1, 2, 3 »;
```


## 2.6 Equality, type-awareness, and comparisons

Zuzu’s `≡` (or `==`) is **type-aware equality**.

```zzs
say 1 == 1;        # true
say 1 ≡ 1;         # true (unicode alias)
say "1" == 1;     # false
say 1 ≢ "1";      # true
```

This is extremely helpful: many accidental cross-type matches that other
languages allow are simply rejected as unequal here.

Also remember the operator split:

- numeric comparisons: `=`, `≠`, `<`, `≤`, `>`, `≥`,
- string comparisons: `eq`, `ne`, `gt`, `ge`, `lt`, `le`, `cmp`.
- case-insensitive string comparisons: `eqi`, `nei`, `gti`, `gei`, `lti`, `lei`, `cmpi`.

Use the operator family that matches your intent.

## 2.7 Mutability vs binding

Two different questions:

1. Can a **value** be mutated?
2. Can a **name** be rebound?

These are related but not identical.

Arrays, dicts, sets, and bags all expose mutating methods. For example:

```zzs
let s := « 1, 2 »;
s.add(3);
s.remove(2);
```

```zzs
let b := <<< 1, 2, 2 >>>;
b.remove_first(2);
b.add(9);
```

In Chapter 3 we’ll separate binding rules (`let`, scope, shadowing) from
value mutation rules in detail.


## 2.8 Coercion: less spooky than in many languages

Because Zuzu separates numeric and string operators, a lot of classic
“type-mix surprise” issues are reduced.

Still, coercion exists in specific places (for example, at API
boundaries, explicit conversion calls, or contexts that define coercion
behaviour).

A safe, boring pattern is:

- normalize inputs at boundaries,
- keep core logic strongly intentional.

```zzs
function normalize_limit ( raw ) {
  if ( raw == null ) {
    return 0;
  }
  return 0 + raw;
}
```


## 2.9 A tiny “set vs bag” mini-lab (REPL)

Start REPL with `zuzu -R`, then try:

```zzs
let suspects_set := « "otter", "otter", "badger" »;
let suspects_bag := <<< "otter", "otter", "badger" >>>;

say suspects_set.length();
say suspects_bag.length();
say suspects_bag.count("otter");
```

Then try operators:

```zzs
say 2 ∈ « 1, 2, 3 »;
say « 1, 2 » ⋃ « 2, 3 »;
say « 1, 2 » ⋂ « 2, 3 »;
```

Observe:

- set deduplicates,
- bag retains multiplicity,
- symbolic operators are first-class syntax, not decoration.


## 2.10 Practical patterns that age well

### Pattern 1: choose data shape before writing loops

Ask first: array, dict, set, or bag?

Often the correct algorithm becomes obvious afterward.

### Pattern 2: use string operators for string intent

```zzs
if ( user_input eq "yes" ) {
  say "confirmed";
}
```

Don’t rely on numeric operators for lexical comparisons.

### Pattern 3: lean on type-aware equality

```zzs
if ( value == expected ) {
  say "exact type-aware match";
}
```

That extra strictness prevents subtle bugs.

### Pattern 4: use Unicode forms where available

For readability in docs and teaching materials, prefer symbols like
`≡`, `≢`, `≤`, `≥`, `∈`, `∉`, `⋃`, `⋂`, `⊂`, and `⊃`.


## 2.11 Recap

You now have the core value model:

- primitives: numbers, strings, booleans, `null`,
- composites: arrays and dicts,
- collections with distinct semantics:
  - sets (`<< >>`) for uniqueness,
  - bags (`<<< >>>`) for multiplicity,
- type-aware equality (`==` / `≡`),
- separate numeric vs string operator families,
- mutation as a value concern, binding as a naming concern.

If Chapter 1 got you running code, Chapter 2 gives you the mental model
for understanding code behaviour.

In Chapter 3, we build directly on this: names, scope, shadowing, and
how values move through program structure.
