# Appendix J: Built-In Classes and Types

ZuzuScript has a small set of built-in classes and runtime type names.
They are used by `typeof`, `instanceof`, type annotations, constructors,
and exception matching. The tree below follows the built-in inheritance
hierarchy used by the runtime.

- `Any` - the top type; every value matches it.
  - `Null` - the `null` value.
  - `Boolean` - `true` and `false`.
  - `Number` - numeric values.
  - `String` - Unicode text strings.
  - `BinaryString` - byte strings.
  - `Task` - asynchronous task values.
  - `Object` - object values.
    - `Collection` - the abstract base for built-in collection values.
      - `Array` - ordered lists.
      - `Set` - unique unordered values.
      - `Dict` - string-keyed mappings.
      - `PairList` - ordered key/value pairs that may repeat keys.
      - `Bag` - counted multisets.
    - `Exception` - the base class for catchable exceptions.
      - `TypeException` - type mismatch failures.
      - `AssertionException` - assertion failures.
      - `ExhaustedException` - iterator or generator exhaustion.
      - `CancelledException` - cancelled asynchronous work.
        - `TimeoutException` - timeout cancellation.
      - `ChannelClosedException` - send or receive on a closed channel.
    - `Pair` - a key/value pair object.
    - `Regexp` - regular expression values.
  - `Function` - function values.
  - `Trait` - trait objects.
  - `Class` - class objects.

Some standard-library modules define additional exception subclasses.
These are not always present as ambient names, but they are part of the
standard library when their modules are imported:

- `Exception`
  - `MarshallingException` - thrown by `std/marshal.dump`.
  - `UnmarshallingException` - thrown by `std/marshal.load`.
  - `BailOutException` - thrown by `test/more` bailout helpers.
  - `SkipAllException` - thrown by `test/more` skip-all helpers.

`typeof` may also report `Method` for a bound method value. It is a
runtime type name rather than a built-in class in the core inheritance
tree.
