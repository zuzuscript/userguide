# Chapter 4: Collections

<img src="https://zuzulang.org/img/zia-collection.jpeg" alt="An impressive collection, Zia!" class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />

Numbers and strings are useful on their own, but programs rarely stop at one
value. As soon as you have several names, several scores, several lines of
input, or several options from a user, you need a collection.

Collection types are how ZuzuScript keeps track of related values. The
two collection types you'll be using all the time are `Array` and `Dict`,
but ZuzuScript has a few more built-in collection types can also be useful.


## 4.1 Arrays

Arrays are ordered lists of values. Array literals consist of `[` followed
by a comma-separated list of values, finished with `]`.

```zzs
// This is an array of strings
let tasks := [ "coffee", "code", "sleep" ];

// Arrays can be indexed into like strings
say tasks[0];

// You can easily add items to the end of an existing array
tasks.add( "repeat" );
```

When defining numeric arrays, it can be useful to use ranges:

```zzs
let possible_scores := [ 1 ... 5, 11 ];

let countdown := [ 10 ... 1 ];
```

## 4.2 Dicts

Dicts are a key–value mapping. Array literals consist of `{` followed
by a comma-separated list of key–value pairs with a colon separating
the key from the value, finished with `}`.

```zzs
let scores := {
	"Zia":      92,
	"Zachary":  78,
	"Zenia":    76,
};

say "Zia's score is: " _ scores{"Zia"};
```

The keys in dicts are always strings. The values may be any type. Unlike
arrays which are inherently ordered, dicts are an unordered collection:
when you put key–value pairs in, there's no guarantee they will come out
in the same order. Dicts cannot contain duplicate keys: Zia can only have
one score in our example.

As a shortcut when dealing with dicts, any string which looks "word-like"
doesn't need to be quoted:

```zzs
let scores := {
	Zia:      92,
	Zachary:  78,
	Zenia:    76,
};

say "Zia's score is: " _ scores{Zia};
```

They are still `String` type; the quotes are just omitted for readability.

This does have one drawback though:

```zzs
let scores := {
	Zia:      92,
	Zachary:  78,
	Zenia:    76,
};

let player = "Zia";
say `${ player }'s score is: ${ scores{player} }`; // WRONG!
```

The problem is that `scores{player}` is treated like `scores{"player"}`
when we want it to be treated like `scores{"Zia"}`.

The solution is to make the key expression look non-word-like:

```zzs
say scores{ (player) };        // works
say scores{ player _ "" };     // also works
say scores{ `${player}` };     // also works
```

They all work, but Zia prefers the first one.

```zzs
let nobody_is_looking = true;

if ( nobody_is_looking ) {
	scores{Zia}++;
}
```

Stop cheating, Zia!


## 4.3 Sets and Bags

The `Set` and `Bag` data types are conceptually similar to `Array`, except
neither of them is inherently ordered (values might not come out in the same
order you put them in) and `Set` cannot contain duplicate items.

```zzs
// set1 and set2 are equal!
let set1 := « 1, 2, 3, 3, 3 »;
let set2 := << 3, 2, 1 >>;

// bag1 and bag2 are equal!
let bag1 := <<< 1, 2, 3, 3, 3 >>>;
let bag2 := <<< 3, 1, 3, 2, 3 >>>;

// but bag3 is different
let bag1 := <<< 1, 2, 3, 3 >>>;
```

Note that `Set` is created with a double angle bracket, and `Bag` is a triple.

There is a special shortcut for creating an empty set:

```zzs
let myset := ∅;

let numbers := ∅.add(1).add(2).add(3);
```

### Arrays versus Sets and Bags

| Type | Contains | Ordered | Duplicates |
|---|---|---|---|
| `Array` | values | Yes | Allowed |
| `Bag` | values | No | Allowed |
| `Set` | values | No | No |

Bags and sets take away the idea of ordering and duplicates from arrays.
But dicts are already unordered and disallow duplicate keys, so now we
will look at a data type that adds ordering and duplicates to it.


## 4.4 PairLists

PairLists are ordered lists of key–value pairs. They use `{{ ... }}`
instead of `{ ... }`.

```zzs
let scores := {{
	Zia:      92,
	Zachary:  78,
	Zenia:    76,
	Zia:      93,
}};

say "Zia's score is: " _ scores{Zia};
```

In this example it may not be obvious if Zia's score will be printed as
92 or 93. But it's 92. The first value is preferred.

We can access both though:

```zzs
say "Zia's first score is: "  _ scores.get_all("Zia")[0];
say "Zia's second score is: " _ scores.get_all("Zia")[1];
```

As you may be able to tell, `pairlist.get_all(key)` is an expression which
gives you an `Array` of all values associated with that key.


### Dicts versus PairLists

| Type | Contains | Ordered | Duplicates |
|---|---|---|---|
| `Dict` | key–value pairs | No | No |
| `PairList` | key–value pairs | Yes | Allowed |


## 4.5 Collection operators

Like numbers and strings, collections have a lot of operators which you can
use to work with them. Some operators will act slightly differently depending
on what type of collection (`Array`, `Dict`, `Set`, `Bag`, or `PairList`)
you use them with.

One of the most useful is `∈` which has alternative spelling `in`.

```zzs
let tasks := [ "coffee", "code", "sleep" ];

if ( "coffee" ∈ tasks ) {
	say "Drinking coffee now!";
}

if ( "code" in tasks ) {
	say "Writing code...";
	say "Compiling...";
	say "Running...";
}
```

If you use `∈` on a `Dict` or `PairList`, it will tell you if the *key*
exists in the dict/pairlist; it doesn't search *values*.

```zzs
a ∈ b;                // "Exists in" operator
a in b;               // "Exists in" operator, but easier to type
a ∉ b;                // "Not in" operator
not( a in b );        // Easier to type
```

Other collection operators want to work on `Set` values, and will
coerce other collections to sets.

```zzs
a ⋃ b;                // Union of a and b
a union b;            // Union of a and b, but easier to type
a ⋂ b;                // Intersection of a and b
a intersection b;     // Intersection of a and b, but easier to type
a ∖ b;                // Difference of a and b
a \ b;                // Difference of a and b, but easier to type
```


## 4.6 Collection comparison operators

There are a few operators for comparing collections. Again these want
to work on sets and will coerce other collection types. These return
booleans.

```zzs
a ⊂ b;                // a is a subset of b
a subsetof b;         // a is a subset of b, but easier to type
a ⊃ b;                // a is a superset of b
a supersetof b;       // a is a superset of b, but easier to type
a ⊂⊃ b;               // a is the same set as b
a equivalentof b;     // a is the same set as b, but easier to type
```

Note that a set is always considered a subset of itself.


## 4.7 The `default` operator

The `default` operator is a binary infix operator for Dicts and PairLists.
It creates a copy of the first Dict or PairList and uses the second one to
fill in any missing keys.

```zzs
let fallback_settings := {
	enable_foo: true,
	enable_bar: true,
	foo_level: 42,
};

let loaded_from_config := {
	enable_bar: false;
};

let combined_setings := loaded_from_config default fallback_settings;

// Result:
// {
//   enable_foo: true,
//   enable_bar: false,
//   foo_level: 42,
// }
```

## 4.8 Collection methods

Collections are a type of object, which means it's possible to call
methods on them using the dot (`.`) operator. We've already seen
this be used a couple of times:

```zzs
tasks.add( "repeat" );

say scores.get_all("Zia")[1];
```

A lot of the functionality of collections happens through method calls
instead of operators. A full list of collection methods is in Appendix G,
but here are some highlights.


### Array methods

```zzs
let myarr := [];

myarr.add( "one" );        // Adds a new item

myarr.push( "two" );       // Adds a new item at the end
let got := myarr.pop();    // Removes an item from the end

myarr.unshift( "zero" );   // Adds a new item at the start
let got := myarr.shift();  // Removes an item from the start

// Get the fourth item from the array (0 is the first item)
let got := myarr.get(3);

// Fallback if there's no fourth item
let got := myarr.get(3, "fallback");

// Set the fourth item to a new value
myarr.set(3, "four");

if ( not myarr.empty ) {
	myarr.clear();
}

say myarr.length(); // says 0

// Add multiple items at once
myarr.add( 1, 2, 7, 9 );

// Make a copy of the array
let yourarr := myarr.copy();

// Alternative to the ∈ operator
if ( yourarr.contains(7) ) {
	say yourarr.sortnum().reverse();  // says "9, 7, 2, 1"
}
```


### Set methods

`Set` mostly provides the same methods as `Array`, except methods which
rely on the collection being in a particular order. There are also some
set-related methods related to combining and comparing sets.

```zzs
let myset := « 1, 2, 3 »;

if ( myset.contains(3) ) {

	// Remove an item from the set
	myset.remove(3);
}

let otherset := « 1, 4, 5 »;

if ( not myset.is_disjoint(otherset) ) {
	// « 2, 4, 5 »
	let difference := myset.symmetric_difference(otherset);
}
```


### Bag methods

Bags provide a far more limited interface.

```zzs
let mybag := « 1, 2, 3, 3, 3, 3 »;

say mybag.length;  // 6

mybag.remove(3);       // Only removes one copy of the value 3
say mybag.length;      // 5

mybag.remove_all(3);   // Removes all copies of the value 3
say mybag.length;      // 2
```


### Dict methods

Dicts provide a rich interface.

```zzs
let scores := {
	Zia:      92,
	Zachary:  78,
	Zenia:    76,
};

let set_of_keys   := scores.keys();
let bag_of_values := scores.values();

if ( scores.exists("Zenia") ) {
	say "Zenia's score is " _ scores.get("Zenia");
}

// Fallback
say "Zizzi's score is " _ scores.get("Zizzi", "unknown");
```


### PairList methods

PairLists provide similar, but not identical, methods to Dicts.

```zzs
let scores := {{
	Zia:      92,
	Zachary:  78,
	Zenia:    76,
	Zia:      93,
}};

let array_of_keys   := scores.keys();
let array_of_values := scores.values();

let zia_first_score     := scores.get("Zia", "unknown");
let array_of_zia_scores := scores.get_all("Zia");
```


### Common methods

Some methods work the same (or approximately the same) on all types of
collection.

- `collection.get(index)` (Array, Set, Bag) or `collection.get(key)` (Dict, PairList)
- `collection.set(index, value)` (Array, Set, Bag) or `collection.set(key, value)` (Dict, PairList)
- `collection.add(value)` (Array, Set, Bag) or `collection.add(key, value)` (Dict, PairList)
- `collection.length()`
- `collection.empty()`
- `collection.copy()`
- `collection.clear()`
- `collection.to_Array()` (except `Array`)
- `collection.to_Iterator()`


## 4.9 Recap

Now we've learnt about:

- the `Array`, `Bag`, `Set`, `Dict`, and `PairList` data types
- collection operators
- collection comparison operators
- collection methods

In Chapter 5, we will look at booleans and truthiness.
That is where collections, numbers, and strings start influencing which code
runs next.
