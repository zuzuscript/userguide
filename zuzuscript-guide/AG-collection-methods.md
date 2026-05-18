# Appendix G: Collection Methods

This document lists the callable methods on ZuzuScript `Array`, `Bag`,
`Set`, `Dict`, and `PairList` values as exposed by the current Perl
implementation and mirrored by `zuzu-rust`.

`foo.bar` is method-call syntax in ZuzuScript, not property access. When a
method takes no arguments, `foo.bar` is shorthand for `foo.bar()`.

Callback signatures:

- Mapper callbacks are called as `fn value -> ...`.
- Predicate callbacks are called as `fn value -> ...` and should return a
  truthy or falsey value.
- Reducer callbacks are called as `fn ( acc, value ) -> ...`.
- Dict and PairList pair callbacks receive a `Pair`-like value.

Ordinary collection methods store values strongly. Weak collection
storage uses explicit method names such as `push_weak`, `set_weak`, and
`add_weak`. Weak method variants store reference-capable values weakly,
store scalar Zuzu values normally, and return the same collection value
as their strong counterparts. Reading a dead weak entry returns `null`;
ordinary methods such as `push`, `set`, and `add` do not inherit
weakness from a previous weak entry.

## Array

| Method | Parameters | Returns | Description |
| --- | --- | --- | --- |
| `append`, `push`, `add` | `...values` | `Array` | Appends one or more values to the end of the array and returns the same array. |
| `push_weak` | `value` | `Array` | Appends `value` through weak-storage rules and returns the same array. |
| `pop` | none | `Any` or `null` | Removes and returns the last element, or `null` if empty. |
| `prepend`, `unshift` | `...values` | `Array` | Inserts one or more values at the front and returns the same array. |
| `unshift_weak` | `value` | `Array` | Inserts `value` at the front through weak-storage rules and returns the same array. |
| `shift` | none | `Any` or `null` | Removes and returns the first element, or `null` if empty. |
| `length`, `count` | none | `Number` | Returns the number of elements. |
| `empty`, `is_empty` | none | `Boolean` | Returns whether the array has no elements. |
| `copy` | none | `Array` | Returns a shallow copy with independent outer storage and the same element values. |
| `get` | `index`, optional `default` | `Any` | Returns the element at `index`, or `default`/`null` if out of range. |
| `set` | `index`, `value` | `Array` | Stores `value` at `index` and returns the same array. |
| `set_weak` | `index`, `value` | `Array` | Stores `value` at `index` through weak-storage rules and returns the same array. |
| `clear` | none | `Array` | Removes all elements and returns the same array. |
| `to_Set` | none | `Set` | Returns a set containing the unique array values. |
| `to_Bag` | none | `Bag` | Returns a bag containing the array values, preserving duplicates. |
| `to_Iterator` | none | `Function` | Returns an iterator function over the array values. |
| `sort` | comparator callback | `Array` | Returns a new array sorted by the callback result. |
| `sortstr` | none | `Array` | Returns a new array sorted by string comparison. |
| `sortnum` | none | `Array` | Returns a new array sorted numerically. |
| `reverse` | none | `Array` | Returns a new array with the element order reversed. |
| `head` | optional `count` | `Array` | Returns a new array containing the first `count` values, default `1`. |
| `tail` | optional `count` | `Array` | Returns a new array containing the last `count` values, default `1`. |
| `sum` | none | `Number` | Returns the numeric sum of the elements. |
| `product` | none | `Number` | Returns the numeric product of the elements. |
| `shuffle` | none | `Array` | Returns a new array with the values in random order. |
| `sample` | optional `count` | `Array` | Returns a new array of randomly sampled values, default `1`. |
| `contains` | `value` | `Boolean` | Returns whether an equal value exists in the array. |
| `map` | mapper callback | `Array` | Returns a new array containing the callback result for each value. |
| `grep` | predicate callback | `Array` | Returns a new array of values for which the callback is truthy. |
| `any` | predicate callback | `Boolean` | Returns whether any element satisfies the callback. |
| `all` | predicate callback | `Boolean` | Returns whether every element satisfies the callback. |
| `first` | predicate callback | `Any` or `null` | Returns the first matching value, or `null` if none match. |
| `remove` | predicate callback | `Array` | Removes all values matching the callback and returns the same array. |
| `first_index` | predicate callback | `Number` | Returns the first matching index, or `-1` if none match. |
| `for_each_value` | mapper callback | `Array` | Calls the callback for each value and returns the same array. |
| `reduce` | reducer callback | `Any` or `null` | Folds the array left-to-right and returns the final accumulator, or `null` when empty. |
| `reductions` | reducer callback | `Array` | Returns each intermediate accumulator value as an array. |

## Bag

| Method | Parameters | Returns | Description |
| --- | --- | --- | --- |
| `add`, `push` | `...values` | `Bag` | Adds one or more values and returns the same bag. |
| `add_weak` | `value` | `Bag` | Adds `value` through weak-storage rules and returns the same bag. |
| `remove`, `remove_first` | `value` | `Bag` | Removes one matching value and returns the same bag. |
| `length` | none | `Number` | Returns the total number of items, including duplicates. |
| `count` | optional `value` | `Number` | With no argument, returns the total item count. With a value, returns how many matching items exist. |
| `empty` | none | `Boolean` | Returns whether the bag has no items. |
| `copy` | none | `Bag` | Returns a shallow copy with independent outer storage and the same item values. |
| `clear` | none | `Bag` | Removes all items and returns the same bag. |
| `contains` | `value` | `Boolean` | Returns whether the bag contains a matching value. |
| `to_Array` | none | `Array` | Returns an array containing the bag values. |
| `to_Set`, `uniq` | none | `Set` | Returns a set of the unique values in the bag. |
| `to_Iterator` | none | `Function` | Returns an iterator function over the bag values. |
| `sum` | none | `Number` | Returns the numeric sum of the values. |
| `product` | none | `Number` | Returns the numeric product of the values. |
| `sort` | comparator callback | `Array` | Returns a new array sorted by the callback result. |
| `sortstr` | none | `Array` | Returns a new array sorted by string comparison. |
| `sortnum` | none | `Array` | Returns a new array sorted numerically. |
| `map` | mapper callback | `Bag` | Returns a new bag containing the callback result for each value. |
| `grep` | predicate callback | `Bag` | Returns a new bag of values for which the callback is truthy. |
| `any` | predicate callback | `Boolean` | Returns whether any item satisfies the callback. |
| `all` | predicate callback | `Boolean` | Returns whether every item satisfies the callback. |
| `first` | predicate callback | `Any` or `null` | Returns the first matching value, or `null` if none match. |
| `remove_if` | predicate callback | `Bag` | Removes all values matching the callback and returns the same bag. |
| `for_each_value` | mapper callback | `Bag` | Calls the callback for each value and returns the same bag. |

## Set

| Method | Parameters | Returns | Description |
| --- | --- | --- | --- |
| `add`, `push` | `...values` | `Set` | Adds one or more values, deduplicating by set equality, and returns the same set. |
| `add_weak` | `value` | `Set` | Adds `value` through weak-storage rules, deduplicating by resolved equality, and returns the same set. |
| `remove` | `value` | `Set` | Removes a matching value and returns the same set. |
| `length`, `count` | none | `Number` | Returns the number of distinct items. |
| `empty`, `is_empty` | none | `Boolean` | Returns whether the set has no items. |
| `copy` | none | `Set` | Returns a shallow copy with independent outer storage and the same item values. |
| `clear` | none | `Set` | Removes all items and returns the same set. |
| `contains` | `value` | `Boolean` | Returns whether the set contains a matching value. |
| `to_Array` | none | `Array` | Returns an array containing the set items. |
| `to_Bag` | none | `Bag` | Returns a bag containing the set items. |
| `to_Iterator` | none | `Function` | Returns an iterator function over the set items. |
| `union` | `Set` | `Set` | Returns a new set containing items present in either set. |
| `intersection` | `Set` | `Set` | Returns a new set containing items present in both sets. |
| `difference` | `Set` | `Set` | Returns a new set containing items present only in the receiver. |
| `symmetric_difference` | `Set` | `Set` | Returns a new set containing items present in exactly one of the two sets. |
| `is_subset` | `Set` | `Boolean` | Returns whether every receiver item is present in the argument set. |
| `is_superset` | `Set` | `Boolean` | Returns whether the receiver contains every item from the argument set. |
| `is_disjoint` | `Set` | `Boolean` | Returns whether the two sets share no items. |
| `equals` | `Set` | `Boolean` | Returns whether both sets contain the same items. |
| `sort` | comparator callback | `Array` | Returns a new array sorted by the callback result. |
| `sortstr` | none | `Array` | Returns a new array sorted by string comparison. |
| `sortnum` | none | `Array` | Returns a new array sorted numerically. |
| `map` | mapper callback | `Set` | Returns a new set containing the mapped values, deduplicated. |
| `grep` | predicate callback | `Set` | Returns a new set of values for which the callback is truthy. |
| `any` | predicate callback | `Boolean` | Returns whether any item satisfies the callback. |
| `all` | predicate callback | `Boolean` | Returns whether every item satisfies the callback. |
| `first` | predicate callback | `Any` or `null` | Returns the first matching item, or `null` if none match. |
| `remove_if` | predicate callback | `Set` | Removes all items matching the callback and returns the same set. |
| `for_each_value` | mapper callback | `Set` | Calls the callback for each item and returns the same set. |

## Dict

| Method | Parameters | Returns | Description |
| --- | --- | --- | --- |
| `keys` | none | `Set` | Returns a set of keys. |
| `values` | none | `Bag` | Returns a bag of values. |
| `enumerate` | none | `Bag` | Returns a bag of pair values. |
| `has`, `exists` | `key` | `Boolean` | Returns whether the key exists. |
| `defined` | `key` | `Boolean` | Returns whether the key exists and its value is not `null`. |
| `get` | `key`, optional `default` | `Any` | Returns the value for `key`, or `default`/`null` if absent. |
| `copy` | none | `Dict` | Returns a shallow copy with independent outer storage and the same key/value entries. |
| `add` | `key`, `value` or array-like pair arguments | `Dict` | Adds or replaces entries and returns the same dict. |
| `add_weak` | `key`, `value` | `Dict` | Adds or replaces one entry through weak-storage rules and returns the same dict. |
| `set` | `key`, `value` | `Dict` | Sets one entry and returns the same dict. |
| `set_weak` | `key`, `value` | `Dict` | Sets one entry through weak-storage rules and returns the same dict. |
| `kv` | none | `Array` | Returns `[ key1, value1, key2, value2, ... ]`; do not rely on Dict key order. |
| `sorted_keys` | none | `Array` | Returns the sorted keys as an array. |
| `remove` | `key` or predicate callback | `Dict` | Removes one key, or removes each pair for which the callback is truthy, then returns the same dict. |
| `length`, `count` | none | `Number` | Returns the number of entries. |
| `empty` | none | `Boolean` | Returns whether the dict has no entries. |
| `clear` | none | `Dict` | Removes all entries and returns the same dict. |
| `to_Array` | none | `Array` | Returns an array of pair values; do not rely on Dict key order. |
| `to_Iterator` | none | `Function` | Returns an iterator function over keys. |
| `for_each_pair` | mapper callback | `Dict` | Calls the callback for each pair value and returns the same dict. |
| `for_each_key` | mapper callback | `Dict` | Calls the callback for each key and returns the same dict. |
| `for_each_value` | mapper callback | `Dict` | Calls the callback for each value and returns the same dict. |

## PairList

| Method | Parameters | Returns | Description |
| --- | --- | --- | --- |
| `keys` | none | `Array` | Returns the keys in pairlist order, preserving duplicates. |
| `values` | none | `Array` | Returns the values in pairlist order. |
| `enumerate` | none | `Bag` | Returns a bag of pair values in pairlist order. |
| `has`, `exists` | `key` | `Boolean` | Returns whether any pair has the given key. |
| `defined` | `key` | `Boolean` | Returns whether any pair with that key has a non-`null` value. |
| `get` | `key`, optional `default` | `Any` | Returns the first value for `key`, or `default`/`null` if absent. |
| `get_all`, `all` | `key` | `Array` | Returns all values for `key` in pairlist order. |
| `copy` | none | `PairList` | Returns a shallow copy with independent outer storage, preserving pair order and duplicate keys. |
| `add` | `key`, `value` or pair-like arguments | `PairList` | Appends one or more pairs and returns the same pairlist. |
| `add_weak` | `key`, `value` | `PairList` | Appends one pair through weak-storage rules and returns the same pairlist. |
| `set` | `key`, `value` | `PairList` | Alias-like append operation that adds a new pair and returns the same pairlist. |
| `set_weak` | `key`, `value` | `PairList` | Alias-like append operation that adds one weak-storage pair and returns the same pairlist. |
| `kv` | none | `Array` | Returns `[ key1, value1, key2, value2, ... ]` in pairlist order. |
| `sorted_keys` | none | `Array` | Returns the keys sorted as strings. |
| `remove` | `key` or predicate callback | `PairList` | Removes all pairs for a key, or removes each pair for which the callback is truthy, then returns the same pairlist. |
| `length`, `count` | none | `Number` | Returns the number of pairs. |
| `empty` | none | `Boolean` | Returns whether the pairlist has no pairs. |
| `clear` | none | `PairList` | Removes all pairs and returns the same pairlist. |
| `to_Array` | none | `Array` | Returns an array of pair values in pairlist order. |
| `to_Iterator` | none | `Function` | Returns an iterator function over keys in pairlist order. |
| `for_each_pair` | mapper callback | `PairList` | Calls the callback for each pair value and returns the same pairlist. |
| `for_each_key` | mapper callback | `PairList` | Calls the callback for each key and returns the same pairlist. |
| `for_each_value` | mapper callback | `PairList` | Calls the callback for each value and returns the same pairlist. |
