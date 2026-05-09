# Chapter 8: Collections in the Wild: Arrays, Sets, Bags, Dicts, PairLists, and Pairs

In Chapter 7, we put behaviour into classes and traits.

Now we focus on the data structures that those classes and functions
pass around all day:

- **Array** for ordered values,
- **Set** for unique membership,
- **Bag** for multiplicity,
- **Dict** for key/value lookup,
- **PairList** for ordered key/value pairs with duplicate keys,
- **Pair** as a first-class key/value tuple object.

This chapter is intentionally aligned to the **Perl implementation**
syntax and APIs used by the test suite.


## 8.1 Choosing the right collection quickly

Ask two questions:

1. Do I care about **order**?
2. Do I care about **duplicate values** or **duplicate keys**?

| Type | Order | Duplicates | Best for |
| --- | --- | --- | --- |
| Array | yes | yes (values) | timelines, steps, queues |
| Set | not semantic | no (values) | membership, dedupe |
| Bag | not semantic | yes (counted values) | frequency analysis |
| Dict | key lookup | no (keys) | config / records |
| PairList | yes (pairs) | yes (keys allowed) | ordered query params |

A quick example:

```zzs
let arr := [ "latte", "mocha", "latte" ];
let set := << "latte", "mocha", "latte" >>;
let bag := <<< "latte", "mocha", "latte" >>>;
let dict := { latte: 2, mocha: 1 };
let pairlist := {{ latte: 1, latte: 2, mocha: 1 }};
```


## 8.2 Literal syntax and membership checks

### Arrays

```zzs
let queue := [ "Zia", "Zachary", "Zenia" ];
```

### Sets

```zzs
let active := << "Zia", "Zachary", "Zia" >>;
```

### Bags

```zzs
let beans := <<< "arabica", "arabica", "robusta" >>>;
```

### Dicts

```zzs
let profile := {
	name: "Zia",
	coffees: 2,
};
```

### PairLists

```zzs
let query := {{
	tag: "perl",
	tag: "collections",
	sort: "new",
}};
```

`PairList` preserves pair order and duplicate keys.

### Membership operators

`in`, `∈`, and `∉` work on collection membership.
For dicts and pairlists, membership checks keys.

```zzs
say "Zia" in queue;
say "Zachary" ∈ active;
say "decaf" ∉ beans;

say "name" in profile;
say "tag" in query;
```


## 8.3 Arrays: ordered, index-friendly, and flexible

Arrays preserve insertion order and duplicates.

```zzs
let tasks := [ "plan", "code", "test" ];
say tasks.get( 1, "missing" );

tasks.push( "ship" );
let doubled := tasks.map( fn x -> x _ "!" );
```

Useful array methods seen in tests:

- `push`, `pop`, `shift`, `unshift`, `append`, `prepend`
- `map`, `grep`, `any`, `all`, `first`
- `reduce`, `reductions`
- `sort`, `sortnum`, `sortstr`, `reverse`
- `to_Set`, `to_Bag`


## 8.4 Sets: unique values and membership-first code

Sets are ideal when your statement is “keep only unique values.”

```zzs
let seen := << >>;
seen.add( "line-1" );
seen.add( "line-1" );
say seen.length(); # 1
```

Set methods in active use include:

- `add`, `push`, `remove`
- `contains`, `is_subset`, `is_superset`, `is_disjoint`, `equals`
- `union`, `intersection`, `difference`, `symmetric_difference`
- `to_Array`, `to_Bag`


## 8.5 Bags: counted duplicates (multiset semantics)

Bags keep multiplicity and expose count-friendly methods.

```zzs
let labels := <<< "ui", "ui", "perf", "docs", "ui" >>>;
say labels.count( "ui" );
```

Bag methods in active use include:

- `add`, `push`, `remove`, `remove_first`
- `count(value)`, `contains`
- `map`, `grep`, `any`, `all`, `first`
- `uniq`, `to_Set`, `to_Array`

`remove` and `remove_first` each remove one matching occurrence in the
Perl runtime behaviour covered by tests.


## 8.6 Dicts: key/value lookup with unique keys

Dicts store one value per key.

```zzs
let cfg := {
	theme: "night",
	autosave: true,
	retries: 3,
};

say cfg.get( "theme", "light" );
cfg.set( "theme", "cozy-night" );
```

Common dict methods:

- `keys`, `values`, `enumerate`
- `has`, `exists`, `defined`, `get`
- `add`, `set`, `remove`, `clear`
- `kv`, `sorted_keys`, `to_Array`
- `for_each_key`, `for_each_value`, `for_each_pair`

A practical detail: `to_Array()` yields `Pair` objects in current
runtime behaviour.


## 8.7 PairLists: ordered pairs with duplicate keys

If you need repeated keys in-order, use `PairList`.

```zzs
let pl := {{
	foo: 1,
	bar: 2,
	foo: 3,
	foo: 4,
}};

say pl.length();
say pl.get( "foo" );      # first value
say pl.all( "foo" ).count();
```

`PairList` methods overlap with dicts, but semantics differ:

- keys are not unique,
- pair order is preserved,
- `get(key)` returns first matching value,
- `all(key)` returns all matching values.

This makes `PairList` a strong fit for HTTP query parameters,
command metadata streams, and ordered forms.


## 8.8 Pairs as first-class values

`Pair` represents a key/value tuple object.

```zzs
let p := new Pair( pair: [ "lang", "zuzu" ] );
say p.key();
say p.value();
```

You will encounter `Pair` when:

- constructing dicts/pairlists programmatically,
- reading `to_Array()` results from dict/pairlist,
- iterating `.enumerate()` and pair callbacks.

Pair-focused construction examples:

```zzs
let p1 := new Pair( pair: [ "foo", 10 ] );
let p2 := new Pair( pair: [ "foo", 20 ] );
let p3 := new Pair( pair: [ "bar", 30 ] );

let pl := new PairList( list: [ p1, p2, p3 ] );
let d := new Dict( list: [ p1, p3 ] );
```


## 8.9 Collection algebra and relation operators

The Perl implementation supports operator forms across core
collection types.

```zzs
let a := << 1, 2 >>;
let b := << 2, 3 >>;

let u1 := a union b;
let u2 := a ⋃ b;

let i1 := a intersection b;
let i2 := a ⋂ b;

let d1 := a \ << 2 >>;
let d2 := a ∖ << 2 >>;

say << 1, 2 >> subsetof << 1, 2, 3 >>;
say << 1, 2, 3 >> supersetof << 1, 2 >>;
say << 1, 2 >> equivalentof << 2, 1 >>;
```

Aliases:

- subset: `⊂`
- superset: `⊃`
- equivalent: `⊂⊃`


## 8.10 Conversions that clarify intent

Use conversions to shift semantics deliberately.

```zzs
let raw := [ "Zia", "Zenia", "Zia", "Zachary" ];
let unique := raw.to_Set();
let freq := raw.to_Bag();

let top := freq.uniq().to_Array().sortstr();
```

Other useful conversions:

```zzs
let s := << 1, 2, 3 >>;
let s_arr := s.to_Array();
let s_bag := s.to_Bag();

let b := <<< 1, 1, 2 >>>;
let b_set := b.to_Set();
let b_arr := b.to_Array();

let d := { a: 1, b: 2 };
let pairs := d.to_Array();

let pl := {{ a: 1, a: 2, b: 3 }};
let pl_arr := pl.to_Array();
```


## 8.11 Real-world patterns

### Dedupe while preserving first-seen order

```zzs
let input := [ "Zia", "Zenia", "Zia", "Zachary" ];
let seen := << >>;
let ordered := [];

for ( name in input ) {
	if ( name not in seen ) {
		seen.add( name );
		ordered.push( name );
	}
}
```

### Frequency report (Bag)

```zzs
let events := [ "save", "save", "open", "save", "open" ];
let freq := events.to_Bag();
let keys := freq.uniq().sortstr();

for ( k in keys ) {
	say k _ ":" _ to_string( freq.count(k) );
}
```

### Deterministic config display (Dict)

```zzs
let cfg := { retries: 3, mode: "quiet", color: false };
let keys := cfg.sorted_keys();

for ( k in keys ) {
	say k _ "=" _ to_string( cfg.get(k) );
}
```

### Ordered duplicate query params (PairList)

```zzs
let q := {{ tag: "perl", tag: "zuzu", page: 1 }};
let tags := q.all( "tag" );

for ( t in tags ) {
	say t;
}
```


## 8.12 Efficiency notes (practical, not micro-benchmarky)

1. Use **Set** for repeated membership checks.
2. Use **Bag** for frequency counting.
3. Use **Dict** for key lookup by name.
4. Use **PairList** when order and duplicate keys are both required.
5. Convert once at boundaries; avoid semantic bouncing.

Meaning first, then performance tuning with measurement.


## 8.13 Pitfalls and guardrails

- Choosing Set when counts matter (should be Bag).
- Choosing Dict when duplicate keys matter (should be PairList).
- Choosing Array for heavy key lookups (often should be Dict).
- Assuming Set/Bag iteration order is part of business logic.
- Forgetting that `to_Array()` from dict/pairlist yields `Pair` values.


## 8.14 Chapter recap

You now have a collection toolkit that matches the Perl runtime:

- `Array` for ordered values,
- `Set` for uniqueness,
- `Bag` for multiplicity,
- `Dict` for key/value records,
- `PairList` for ordered duplicate keys,
- `Pair` for key/value tuple objects.

That foundation matters for Chapter 9, where path operators query and
slice nested structures that frequently contain these types.
