# Chapter 8: Objects, Roles, and Other Social Constructs

In Chapter 7, we learned to shape behaviour with functions.

Now we move one level up:

> “How do we organize state *and* behaviour together?”

That is where objects, classes, and traits step in.

If functions are great individual tools, classes are your labeled drawers,
traits are your reusable stickers, and objects are the tools currently
covered in coffee foam because Zia "was helping."

In this chapter we will cover:

- class and object basics,
- instance and static methods,
- inheritance and method dispatch,
- traits (roles) and composition,
- encapsulation practices,
- practical patterns for larger scripts.


## 8.1 Your first class: state plus behaviour

A class groups field declarations and methods.

```zzs
class Animal {
	let name;

	method get_name () {
		return name;
	}
}

let fox := new Animal( name: "Fenn" );
say fox.get_name();
```

What happened there:

1. `class Animal { ... }` defines a class value,
2. `let name;` defines an instance field,
3. `method get_name () { ... }` defines instance behaviour,
4. `new Animal( ... )` constructs an object,
5. `name: "Fenn"` initializes a field by name.

If you have worked in other languages, the constructor style may feel
familiar: named initialization keeps call sites easy to read.

### Field declarations inside classes

You can use both mutable and constant field declarations:

```zzs
class Badge {
	let owner;
	const kind := "CoffeePass";
}
```

- `let` fields can be updated,
- `const` fields are fixed after construction.

You can also request common accessor methods directly on a field:

```zzs
class Person {
	let String name with get, set, clear, has := "Anon";
}
```

This expands into methods named `get_name`, `set_name`,
`clear_name`, and `has_name`.

- `get` returns the field value.
- `set` stores the new value and returns `self`.
- `clear` sets the field to `null` and returns `self`.
- `has` returns `true` when the field is not `null`.

The `clear` accessor is intentionally allowed to assign `null`
even when the field has a declared type such as `String`.

Fields may also use `but weak` when the field is a non-owning reference:

```zzs
class Node {
	let parent with get, set, clear, has but weak;
	let children := [];
}
```

A weak field stores reference-capable values weakly, and stores scalar
Zuzu values normally. Generated setters preserve that field behaviour,
so `node.set_parent(parent)` still writes through weak-storage rules.
Generated getters return the live value while it exists, or `null` after
the referent has gone away. Generated `has_` methods return false once a
weak referent has gone away.

A class can also be declared in a compact form with no body block:

```zzs
class Empty;
```

That can be useful as a marker/base type.


### Class scoping

Classes are scoped just like variables and functions.

```zzs
{
	class Raccoon;
	let zia := new Raccoon;
}

let zachary := new Raccoon; // Error!
```


## 8.2 Constructing objects with `new`

Use `new` with a class expression to create instances.

```zzs
class Raccoon {
	let name;
	let mood := "sleepy";
}

let zia   := new Raccoon( name: "Zia" );
let zenia := new Raccoon( name: "Zenia", mood: "curious" );
```

A few practical notes:

- You can provide only the fields you want to override.
- Default field values (like `mood := "sleepy"`) are applied first,
  then named constructor values override them.
- Unknown fields fail at runtime with a clear error.

### Accessing field values

Inside methods, field names are directly in scope. Outside methods, use
member access forms such as object slot lookup:

```zzs
class Thermos {
	let cups := 2;
}

let t := new Thermos();
say t{cups};
```

That `obj{field}` form is handy for inspection, quick scripts, and
assertions in tests. However, it is considered bad form to use it
as part of your API: define accessors (`with get`, etc) instead.


## 8.3 Methods and dispatch

Instance methods are defined with `method`.

```zzs
class Counter {
	let n := 0;

	method inc () {
		n := n + 1;
		return n;
	}
}

let c := new Counter();
say c.inc();  # 1
say c.inc();  # 2
```

Method calls use dot syntax:

```zzs
obj.method_name( args... )
```

If a method call has no argument, you can leave off the parentheses entirely.

```zzs
let c := new Counter();
say c.inc;  # 1
say c.inc;  # 2
```

This is different from languages like JavaScript where leaving off the
parentheses means something entirely different.


### `self` for explicit receiver calls

Within methods, `self` refers to the current instance.

```zzs
class Pet {
	let name;

	method label () {
		return "pet:" _ self.get_name;
	}

	method get_name () {
		return name;
	}
}
```


## 8.4 Inheritance: extending classes

Use `extends` to inherit fields and methods.

```zzs
class Animal {
	let name;
	const species := "Canis";

	method get_name () {
		return name;
	}
}

class Fox extends Animal {
	let coat := "Unknown";

	method describe () {
		return self.get_name _ ":" _ coat _ ":" _ self{species};
	}
}

let fox := new Fox( name: "Fenn", coat: "Red" );
say fox.describe();
```

Key idea: subclass instances still satisfy parent-type checks.

```zzs
fox instanceof Animal
```

That returns true for objects whose class is `Animal` or a descendant of
`Animal`.

### Overriding methods

A subclass may define a method with the same name as an inherited one.
The subclass version wins for that class’s instances.

Use overriding when the concept is “same interface, specialized
behaviour.”


### The `instanceof` operator

The `instanceof` operator is a better and safer way of checking a
value's type than `typeof`.

```zzs
// Unreliable because it's technically possible to define
// multiple different classes with the same name "Widget".
if ( typeof x eq "Widget" ) {
	...;
}

// Better. This brings joy to Zia.
if ( x instanceof Widget ) {
	...;
}
```


## 8.5 `super()` in method overrides

When overriding, you may still want parent (or composed trait) behaviour.
Call `super()` from inside the overriding method.

```zzs
class Parent {
	static method label () {
		return "parent-static";
	}
}

class Child extends Parent {
	static method label () {
		return super() _ ":child-static";
	}
}

say Child.label();
```

`super()` is also useful with trait-provided methods (see Section 8.7).


## 8.6 Static methods: behaviour on the class itself

Define class-level methods using `static method`.

```zzs
class Counter {
	static method ten () {
		return 10;
	}

	static method add_two ( x ) {
		return x + 2;
	}
}

say Counter.ten();
say Counter.add_two(3);
```

Use static methods when logic does not depend on one object’s state,
for example:

- parsers/factory helpers,
- pure utility transforms bound to a domain type,
- shared constants exposed as callable helpers.

If code needs per-object fields, keep it as an instance method instead.


## 8.7 Traits (roles): shared behaviour without inheritance chains

Traits are reusable method bundles.

```zzs
trait Named {
	method tag () {
		return "tag:" _ self.get_name;
	}
}
```

Compose them into classes with `with` (or `but`, an alias):

```zzs
class Animal {
	let name;

	method get_name () {
		return name;
	}
}

class Owl extends Animal with Named;
class Fox extends Animal but Named;

let owl := new Owl( name: "Mochi" );
let fox := new Fox( name: "Rin" );

say owl.tag;
say fox.tag;
```

Traits are great for capabilities that cross-cut your class tree:

- tagging/log-label behaviour,
- serialization helpers,
- validation helpers,
- domain-specific feature packs.


### Type checks with traits

Use `does` to ask whether a class/object composes a trait:

```zzs
owl does Named
```

You can also test method availability with `can`:

```zzs
owl can "tag"
owl can tag
```

That is useful for defensive/plug-in style logic.


## 8.8 Inheritance vs composition: choosing the shape

A practical beginner rule:

- Use **inheritance** (`extends`) for “is-a” relationships.
- Use **traits/composition** (`with` / `but`) for “has-a capability.”

Example intuition:

- `Fox extends Animal` makes sense (“fox is an animal”),
- `Owl with Named` adds behaviour capability,
- a `ReportBuilder` probably should not `extend JSON`; it should hold
  or use one.

When in doubt, prefer shallower class trees plus traits. Deep
inheritance can become brittle because behaviour is spread across many
ancestors.


## 8.9 Lifecycle hooks: `__build__` and `__demolish__`

ZuzuScript supports object lifecycle hooks.

### `__build__`

If present, `__build__` runs automatically after `new` creates the
object.

```zzs
class Builder {
	let value := 1;

	method __build__ () {
		value := value + 9;
	}
}

let built := new Builder();
say built{value};  # 10
```

Use this for lightweight post-construction setup.

Advanced runtime modules can use `make_instance(klass, dict)` from
`std/internals` to allocate a user-defined object without running `__build__`.
This is intended for infrastructure such as unmarshalling object graphs, not
ordinary application construction.

### `__demolish__`

If present, `__demolish__` is called during object teardown/GC.

```zzs
class Temp {
	let marker := "";

	method __demolish__ () {
		say "cleaning " _ marker;
	}
}
```

Practical caution:

- teardown timing depends on runtime/GC behaviour,
- do not rely on exact timing for critical external operations,
- prefer explicit cleanup methods when deterministic ordering matters.


## 8.10 Nested classes and class-local structure

A class can contain nested class declarations.

```zzs
class Box {
	class Widget {
		let id;

		method get_id () {
			return id;
		}
	}

	method build ( x ) {
		return new Widget( id: x );
	}
}

let box := new Box();
let widget := box.build(42);
say widget.get_id();
```

This is useful when a helper type is conceptually owned by one parent
abstraction and not intended as a broad top-level type.


## 8.11 Dynamic member calls for advanced dispatch

Sometimes the method name is computed at runtime. Use dynamic call
syntax:

```zzs
class Adder {
	method plus ( x ) {
		return x + 10;
	}
}

let obj := new Adder();
let method_name := "plus";
say obj.(method_name)(5);
```

Use this sparingly:

- it is flexible for plug-ins/command routing,
- but harder to read than direct `obj.plus(...)` calls.

A good pattern is to keep dynamic dispatch near boundaries (CLI command
maps, adapters) and keep core domain code explicit.


## 8.12 Encapsulation in practice

ZuzuScript keeps object syntax lightweight; “encapsulation” is primarily
a design practice rather than heavy visibility keywords.

Practical habits that work well:

1. Expose intention-focused methods,
2. Keep raw slot reads/writes localized,
3. Use `const` for invariants,
4. Prefer method calls over reaching into fields from many files.

Example pattern:

```zzs
class CoffeeQueue {
	let items := [];

	method enqueue ( label ) {
		items.push(label);
	}

	method next () {
		if ( items.count = 0 ) {
			return null;
		}

		let out := items[0];
		items := items.slice( 1, items.count - 1 );
		return out;
	}
}
```

Even if fields are technically reachable, treating methods as the public
surface keeps invariants together and codebases calmer.


## 8.13 Practical patterns you can use today

### Pattern A: Capability traits

Keep reusable behaviour in traits, then compose where needed.

```zzs
trait Timestamped {
	method stamp ( msg ) {
		return "[log] " _ msg;
	}
}

class Task with Timestamped;
class Job with Timestamped;
```

### Pattern B: Thin base classes

Use base classes for minimal shared state/contract, not giant
kitchen-sink behaviour.

```zzs
class Animal {
	let name;
	method get_name () { return name; }
}
```

Then add optional behaviour with traits.

### Pattern C: Type-aware branch logic

Use `instanceof`, `does`, and `can` to support extensible systems.

```zzs
function describe ( x ) {
	if ( x does Named ) {
		return x.tag();
	}

	if ( x can to_String ) {
		return x.to_String;
	}

	return "(unknown)";
}
```

### Pattern D: Object + function blend

Not every abstraction must be a class. A good design often combines:

- classes for persistent state and clear identities,
- functions for pure transforms and pipelines.

Chapter 7 and Chapter 8 are meant to be used together.


## 8.14 Mini walkthrough: sleepy raccoon roster

Let’s combine classes, inheritance, traits, static helpers, and type
checks in one small example.

```zzs
trait Named {
	method badge () {
		return "name=" _ self.get_name;
	}
}

class Raccoon {
	let name;
	let coffee := 0;

	method get_name () {
		return name;
	}

	method sip () {
		coffee := coffee + 1;
		return coffee;
	}

	static method species () {
		return "Procyon lotor";
	}
}

class EngineerRaccoon extends Raccoon with Named {
	let language := "zuzu";

	method intro () {
		return self.badge()
			_ ", coffee=" _ coffee
			_ ", lang=" _ language;
	}
}

let zia := new EngineerRaccoon(
	name: "Zia",
	language: "zuzu"
);

zia.sip;
zia.sip;

say zia.intro;
say zia instanceof Raccoon;
say zia does Named;
say EngineerRaccoon.species;
```

Why this is a solid default architecture:

- base class keeps common identity/state,
- trait adds reusable “capability” behaviour,
- subclass adds specialization,
- static method exposes class-level fact,
- runtime type checks support extension points.

Zia approves this architecture with a sleepy nod.

<img src="https://zuzulang.org/img/zia-pbjection.jpeg" alt="Objection!" class="w-50 float-end d-none d-lg-block ms-3 mb-3 rounded" />


## 8.15 What comes next

You now have the tools to shape programs around domain objects,
capabilities, and relationships.

Next up: Chapter 9, where we talk about what happens when reality
strikes back — errors, exceptions, and recoverable regret.
