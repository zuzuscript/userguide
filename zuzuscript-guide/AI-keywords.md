# Appendix I: Keywords

These are ZuzuScript's keywords. While some ZuzuScript implementations may
allow you to name consts, variables, functions, classes, traits, and
methods with these names, doing so may not be portable. For this appendix,
word-like operators such as `eq`, `typeof`, `instanceof`, `can`, `sqrt`,
and `subsetof` are treated as keywords, and context-sensitive grammar words
are included too.

## Alphabetical List

`abs`, `and`, `as`, `assert`, `async`, `await`, `but`, `can`, `case`,
`catch`, `ceil`, `class`, `clear`, `cmp`, `cmpi`, `const`, `continue`,
`debug`, `default`, `die`, `do`, `does`, `else`, `eq`, `eqi`,
`equivalentof`, `extends`, `false`, `floor`, `fn`, `for`, `from`,
`function`, `ge`, `gei`, `get`, `gt`, `gti`, `has`, `if`, `import`, `in`,
`instanceof`, `int`, `intersection`, `last`, `lc`, `le`, `lei`, `length`,
`let`, `lt`, `lti`, `method`, `mod`, `nand`, `ne`, `nei`, `new`, `next`,
`not`, `null`, `or`, `print`, `return`, `round`, `say`, `self`, `set`,
`spawn`, `sqrt`, `static`, `subsetof`, `supersetof`, `super`, `switch`,
`throw`, `trait`, `true`, `try`, `typeof`, `uc`, `union`, `unless`,
`warn`, `weak`, `while`, `with`, `xor`.

## Keyword Table

| Keyword | Description |
|---|---|
| `abs` | Prefix operator for absolute value. |
| `and` | Boolean conjunction operator. |
| `as` | Introduces an import alias. |
| `assert` | Debug assertion statement. |
| `async` | Marks an async function, method, or lambda. |
| `await` | Waits for an asynchronous block or task result. |
| `but` | Composes traits like `with`, and introduces `but weak`. |
| `can` | Tests whether a value supports a method name. |
| `case` | Starts a `switch` branch. |
| `catch` | Handles values thrown from a `try` block. |
| `ceil` | Prefix operator rounding up to an integer. |
| `class` | Declares a class. |
| `clear` | Generates a field clearer in `with clear`. |
| `cmp` | Case-sensitive string three-way comparison. |
| `cmpi` | Case-insensitive string three-way comparison. |
| `const` | Declares a non-reassignable binding. |
| `continue` | Continues switch execution into the next case/default. |
| `debug` | Emits debug output when the runtime debug level allows it. |
| `default` | Starts the fallback branch in a `switch`. |
| `die` | Throws an exception-style failure value. |
| `do` | Evaluates a block as an expression. |
| `does` | Tests whether a value composes a trait. |
| `else` | Introduces an alternate branch or loop fallback. |
| `eq` | Case-sensitive string equality operator. |
| `eqi` | Case-insensitive string equality operator. |
| `equivalentof` | Tests set or collection equivalence. |
| `extends` | Declares a superclass. |
| `false` | Boolean false literal. |
| `floor` | Prefix operator rounding down to an integer. |
| `fn` | Starts a short lambda expression. |
| `for` | Starts an iteration loop. |
| `from` | Starts an import statement's module path. |
| `function` | Declares a named or anonymous function. |
| `ge` | Case-sensitive string greater-than-or-equal operator. |
| `gei` | Case-insensitive string greater-than-or-equal operator. |
| `get` | Generates a field getter in `with get`. |
| `gt` | Case-sensitive string greater-than operator. |
| `gti` | Case-insensitive string greater-than operator. |
| `has` | Generates a field presence tester in `with has`. |
| `if` | Starts a conditional, or a postfix condition. |
| `import` | Introduces imported names. |
| `in` | Separates a loop variable from its iterable, and tests membership. |
| `instanceof` | Tests whether a value is an instance of a class. |
| `int` | Prefix operator converting to an integer value. |
| `intersection` | Set or collection intersection operator. |
| `last` | Exits the nearest loop. |
| `lc` | Prefix operator converting text to lowercase. |
| `le` | Case-sensitive string less-than-or-equal operator. |
| `lei` | Case-insensitive string less-than-or-equal operator. |
| `length` | Prefix operator returning string or collection length. |
| `let` | Declares a mutable binding. |
| `lt` | Case-sensitive string less-than operator. |
| `lti` | Case-insensitive string less-than operator. |
| `method` | Declares an instance method. |
| `mod` | Remainder operator. |
| `nand` | Boolean NAND operator. |
| `ne` | Case-sensitive string inequality operator. |
| `nei` | Case-insensitive string inequality operator. |
| `new` | Constructs an object from a class. |
| `next` | Skips to the next loop iteration. |
| `not` | Boolean negation operator. |
| `null` | Null literal. |
| `or` | Boolean disjunction operator. |
| `print` | Prints values without adding a newline. |
| `return` | Returns from the current function or method. |
| `round` | Prefix operator rounding to the nearest integer. |
| `say` | Prints values with a newline. |
| `self` | Refers to the current object or current class context. |
| `set` | Generates a field setter in `with set`. |
| `spawn` | Starts asynchronous work. |
| `sqrt` | Prefix operator for square root. |
| `static` | Marks a method as belonging to the class. |
| `subsetof` | Tests whether one set or collection is a subset of another. |
| `supersetof` | Tests whether one set or collection is a superset of another. |
| `super` | Calls the overridden method implementation. |
| `switch` | Starts a multi-branch comparison statement. |
| `throw` | Throws a value to be handled by `catch`. |
| `trait` | Declares a trait. |
| `true` | Boolean true literal. |
| `try` | Starts exception-handling or optional-import flow. |
| `typeof` | Prefix operator returning a value's type name. |
| `uc` | Prefix operator converting text to uppercase. |
| `union` | Set or collection union operator. |
| `unless` | Negated condition, including postfix conditions. |
| `warn` | Emits warning output. |
| `weak` | Marks weak storage in the `but weak` modifier. |
| `while` | Starts a conditional loop. |
| `with` | Adds traits to a class, or field accessors to a field. |
| `xor` | Boolean exclusive-or operator. |
