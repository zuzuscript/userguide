# Chapter 9: Query All the Things: Navigating Nested Data

In Chapter 8, we built a practical mental model for Arrays, Sets, Bags,
Dicts, PairLists, and Pairs.

Now we get a superpower: asking deeply nested data structures useful
questions *without* writing five nested loops and a sad weekend of
index bookkeeping.

In ZuzuScript (Perl implementation), this chapter is centered on the
runtime **path operators** and the `std/path/z` module:

- `@` for first match,
- `@?` for existence,
- `@@` for all matches,
- plus reusable `ZPath` query objects.

If Chapter 8 taught us which container to use, Chapter 9 teaches us how
to *move through containers fluently*.


## 9.1 Why path queries exist

Imagine a script that loads API payloads, config files, and generated
results. Most real payloads are “dict of arrays of dicts of arrays”.

Without path queries, code often becomes:

- brittle (hard-coded indexes everywhere),
- repetitive (same traversal in multiple places),
- noisy (logic hidden under navigation boilerplate).

With path queries, you can express intent directly:

- “Give me all user names”,
- “Does the second user have an email?”,
- “Set every matching value to this one”.

That is exactly the use-case covered by `std/path/z` and the `@` family
operators in the runtime.


## 9.2 Quick refresher: sample nested value

We will use this running structure:

```zzs
let report := {
	team: {
		name: "Night Shift",
		members: [
			{ name: "Zia", role: "ops", coffee: 3 },
			{ name: "Zachary", role: "dev", coffee: 1 },
			{ name: "Zenia", role: "dev", coffee: 2 },
		],
	},
	meta: {
		tags: [ "cozy", "on-call", "raccoon" ],
	},
};
```

You could reach values manually with indexing and key lookups. But this
chapter focuses on path-driven access instead.


## 9.3 The three runtime path operators

The parser/runtime support three binary operators:

1. `target @ path`
2. `target @? path`
3. `target @@ path`

When `path` is a string, the runtime uses `std/path/zz::ZZPath` by
default. Other path flavours can be selected lexically with their
`use()` methods.

### `@` — first match

Returns the first selected value, or `null` if nothing matched.

```zzs
let first_name := report @ "/team/members/*/name";
say first_name;   # "Zia"
```

Think: “I expect one useful thing; first is fine.”

### `@?` — exists

Returns a boolean-style result (`true`/`false`) indicating whether the
path matches anything.

```zzs
say report @? "/team/members/#1/name";   # true
say report @? "/team/members/#9/name";   # false
```

Think: “Before I proceed, does this path exist?”

### `@@` — all matches

Returns an Array of all matched values.

```zzs
let names := report @@ "/team/members/*/name";
say names;   # [ "Zia", "Zachary", "Zenia" ]
```

Think: “Give me the complete result set.”


## 9.4 Reading path expressions (ZPath basics)

The path string grammar comes from ZPath behaviour implemented in
`std/path/z`.

### Absolute vs relative

- `"/team/members"` starts at the root value.
- `"team/members"` is relative in contexts that support it.

For clarity in app code, absolute paths are often easier to reason
about.

### Segment forms you will use constantly

- `/name` — child key/name,
- `/*` — wildcard children,
- `/#0` — numeric index (0-based),
- `/**` — deep traversal (descendants + root behaviour per evaluator),
- `/..` — parent traversal,
- `/@attr` — attributes for supported object types.

Examples:

```zzs
let second := report @ "/team/members/#1/name";
let all_names := report @@ "/team/members/*/name";
let many := report @@ "/**";
```


## 9.5 Selecting and filtering

Filters are written in square brackets on a segment.

### Existence/truthy filter

```zzs
let caffeinated := report @@ "/team/members/*[coffee]/name";
```

This keeps members where `coffee` is truthy.

### Negated filter

```zzs
let no_coffee := report @@ "/team/members/*[!coffee]/name";
```

### Comparisons and logic

```zzs
let devs := report @@ "/team/members/*[role == 'dev']/name";
let heavy := report @@ "/team/members/*[coffee >= 2]/name";
let mixed := report @@ "/team/members/*[coffee >= 2 && role == 'dev']/name";
```

### Useful filter functions

From `std/path/z` tests and module behaviour, common helpers include:

- `is-first()`
- `is-last()`
- `index()`
- `key()`
- `value()`
- `type(...)`
- string helpers like `contains(...)` in filter expressions

Example:

```zzs
let first_only := report @@ "/team/members/*[is-first()]/name";
let second_only := report @@ "/team/members/*[index() == 1]/name";
```


## 9.6 “Single vs multiple” return contracts

This is the part that prevents subtle bugs.

- `@` returns **one value** (or `null`).
- `@@` returns an **Array** (possibly empty).
- `@?` returns a **boolean-like** value.

So write code that reflects that contract:

```zzs
let maybe_email := report @ "/team/members/#0/email";
if ( maybe_email ≡ null ) {
	say "No email found";
}

let emails := report @@ "/team/members/*/email";
for ( e in emails ) {
	say e;
}

if ( report @? "/team/members/#2/role" ) {
	say "Third member has a role";
}
```

A common mistake is treating `@@` as scalar.
If you chose `@@`, you chose collection semantics.


## 9.7 Composability with normal expressions

Path queries are expressions, so they compose naturally with Chapter 4,
5, 6, and 8 tools.

### In conditions

```zzs
if ( report @? "/team/members/*[role == 'ops']" ) {
	say "Ops coverage found";
}
```

### In loops

```zzs
for ( n in report @@ "/team/members/*/name" ) {
	say "hello " _ n;
}
```

### With collection operations

```zzs
let tags := report @@ "/meta/tags/*";
let unique_tags := tags.to_Set();
let freq := tags.to_Bag();
```

### Inside functions

```zzs
function member_names ( Dict data ) -> Array {
	return data @@ "/team/members/*/name";
}
```

This is where Chapter 8’s collection fluency pays off: query results are
just ordinary values you already know how to process.


## 9.8 Reusable queries with `std/path/z`

For repeated queries, build a `ZPath` once and reuse it.

```zzs
from std/path/z import ZPath;

let member_name_path := new ZPath( path: "/team/members/*/name" );

let a := member_name_path.query(report);
let b := member_name_path.first( report, "n/a" );
let c := member_name_path.exists(report);
```

A small helper wrapper can make one-off calls concise:

```zzs
from std/path/z import ZPath;

function query ( data, path ) {
  return new ZPath( path: path ).query(data);
}

function first ( data, path, fallback ) {
  return new ZPath( path: path ).first( data, fallback );
}

function exists ( data, path ) {
  return new ZPath( path: path ).exists(data);
}

let all_names := query( report, "/team/members/*/name" );
let one_name := first( report, "/team/members/#0/name", "n/a" );
let has_ops := exists( report, "/team/members/*[role == 'ops']" );
```

Use runtime operators when readability is strongest inline; use compiled
`ZPath` when query reuse or customization matters.

`std/path/z` keeps parent metadata for query nodes as non-owning
back-references. That is an implementation detail, but it matters for
module authors extending path support: parent links should be weak, and
the active query evaluation should keep any values it still needs alive
through ordinary strong local references.


## 9.9 Assignment through paths (`@` and `@@` as lvalues)

A powerful runtime feature: path expressions with `@` and `@@` can be
assignment targets. The `@?` operator is for existence/maybe-style
queries and is not an assignable target.

### Assign first match

```zzs
report @ "/team/members/#0/name" := "Captain Zia";
```

This updates the first matched node.

### Assign all matches

```zzs
report @@ "/team/members/*/role" := "engineer";
```

This updates all matching nodes.

Weak writes use the same assignable path forms:

```zzs
report @ "/team/lead" := lead_node but weak;
report @@ "/team/members/*/parent" := parent_node but weak;
```

`@` updates the first selected target weakly. `@@` updates every
selected target weakly. `@?` cannot be used for weak path assignment
because it is not assignable.

### Compound path assignment

Path lvalues support the ordinary assignment family, including numeric,
string, null-coalescing, and regexp-replace forms.

```zzs
report @ "/team/members/#0/score" += 5;
report @@ "/team/members/*/tags/#0" _= "-reviewed";
report @@ "/team/members/*/name" ~= / /g -> "_";
```

The modes differ in the same way they do for plain `:=`:

- `@` updates the first match and throws if there is no match.
- `@@` updates every match and tolerates the empty-match case.

### Update expressions and references

Path lvalues also work with prefix/postfix update operators and unary
reference creation.

```zzs
let old_count := ( report @ "/meta/retries" )++;
let new_ages := ++( report @@ "/team/members/*/age" );

let title_ref := \( report @ "/meta/title" );
let age_refs := \( report @@ "/team/members/*/age" );
```

Use parentheses around the path expression itself for `++`, `--`, and
`\`. This keeps the target explicit and matches how these operators bind
in the parser.

### Important guardrails

- `@` assignment throws if no match is found.
- `@@` assignment is tolerant when no matches are found.
- `@?` is not an assignment or update target.
- `\( data @ path )` returns one reference.
- `\( data @@ path )` returns an array of references.

This difference is intentional and useful:

- choose `@` when absence is exceptional,
- choose `@@` when mass update is optional,
- choose `@?` only for read/existence checks where a missing target is
  expected and should be handled without an exception.


## 9.10 Working across mixed structures

One chapter ago, we used multiple collection types deliberately.
Path queries work best when you stay aware of those semantics.

### Dicts and Arrays

These are the most common and most intuitive for path traversal:

```zzs
let x := report @ "/team/name";
let y := report @ "/team/members/#2/name";
```

### PairLists and Pairs

`std/path/z` supports PairList patterns, including key-based and indexed
access behaviour.

```zzs
let params := {{ tag: "perl", tag: "zuzu", page: 1 }};
let tags := params @@ "/tag/@value";
```

(You can also query by index-like forms depending on what exactly you
need to retrieve: key, value, or whole pair.)

### Special object attributes

Some object types expose path attributes (for example `Time`, `Path`,
XML node/document forms, and Pair key/value views).

```zzs
# Illustrative style, depending on object source:
# let year := some_time_object @ "/@year";
# let size := some_path_object @ "/@size";
```

When you query objects, remember you are reading *the path model* for
that object type, not arbitrary private class state.


## 9.11 Slicing and “narrow then process” patterns

You will often do this in two steps:

1. query a narrow result set,
2. process with regular Array/Set/Bag functions.

Example: find dev members with coffee >= 2, then sort names.

```zzs
let selected := report @@
	"/team/members/*[role == 'dev' && coffee >= 2]/name";
let sorted := selected.sortstr();
```

Another practical pattern: pull a broad slice, then convert semantics.

```zzs
let roles := report @@ "/team/members/*/role";
let role_counts := roles.to_Bag();
say role_counts.count( "dev" );
```

This “query + collection transform” style is one of the most productive
idioms in the Perl implementation today.


## 9.12 Error handling habits

Path queries are expressive, but still runtime behaviour. Be deliberate.

### Prefer `@?` when probing optional paths

```zzs
if ( report @? "/team/members/#5/name" ) {
	say "There is a sixth member";
}
else {
	say "No sixth member";
}
```

### Prefer `first(..., fallback)` when using `std/path/z`

```zzs
let maybe := new ZPath( path: "/team/members/#5/name" )
  .first( report, "unknown" );
```

### Reserve path assignment for clearly valid targets

For updates, it is usually wise to validate target existence first
(especially before `@` assignment), unless you explicitly want failure.


## 9.13 Tiny real-world mini-lab

Let Zia audit sleepy on-call data.

```zzs
let incidents := {
	items: [
		{ id: 101, owner: "Zia", severity: 3, status: "open" },
		{ id: 102, owner: "Zachary", severity: 1, status: "closed" },
		{ id: 103, owner: "Zia", severity: 2, status: "open" },
	],
};

# 1) list owners of open incidents
let open_owners := incidents @@ "/items/*[status == 'open']/owner";

# 2) does any critical incident exist?
let has_critical := incidents @? "/items/*[severity >= 3]";

# 3) normalize owner name typo on first match
incidents @ "/items/*[owner == 'Zia']/owner" := "Zia";

# 4) mark all open incidents as triaged
incidents @@ "/items/*[status == 'open']/status" := "triaged";
```

Notice how little traversal boilerplate appears. The code reads close to
requirements, which is the whole point.


## 9.14 Pitfalls to avoid

- **Using `@` when you really need all matches**.
- **Using `@@` and forgetting it returns an Array**.
- **Assigning with `@` without considering missing-match errors**.
- **Assuming every object exposes path attributes**.
- **Writing giant path expressions when two smaller steps are clearer**.

A good rule: if expression readability drops, split into intermediate
variables.


## 9.15 Chapter recap

You now have the practical model for nested-data querying in the Perl
implementation:

- `@` = first result,
- `@?` = existence check,
- `@@` = all results,
- `std/path/z` = reusable and feature-rich query engine,
- assignment via path for focused (`@`) and bulk (`@@`) updates.

Combined with Chapter 8 collections, this gives you an ergonomic
query-driven style for real scripts.

Next up: Chapter 10, where we talk about what happens when reality
strikes back — errors, exceptions, and recoverable regret.
