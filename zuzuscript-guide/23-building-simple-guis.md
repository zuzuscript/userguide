# Chapter 23: Building Simple GUIs

Chapters 21 and 22 showed ZuzuScript responding to web requests. A web app
is one way to give a script a user-facing surface; a small graphical window
is another.

Most of this guide has treated ZuzuScript as a command-line scripting
language. That is still its centre of gravity, but scripts sometimes
need a small window: a form, a confirmation step, a file picker, or a
tool that non-terminal users can run comfortably.

The `std/gui` module provides that surface. It is deliberately modest:
you build a tree of widgets, attach event handlers, show a window, and
keep application logic in ordinary ZuzuScript values and functions.

At the time of writing, `std/gui` is implemented by the Perl runtime
using Prima, by `zuzu-rust` using GTK4, and by `zuzu-js` when launched
under Electron. Plain Node.js and browser JavaScript runtimes deny GUI
access by default.


## 23.1 A Tiny Window

Import `std/gui` and compose widgets with constructor helpers:

```zzs
from std/gui import *;

let win := Window(
	title: "Zia's Snack Counter",
	width: 360,
	height: 180,
	VBox(
		padding: 12,
		gap: 8,
		Label( text: "How many berries did Zia find?" ),
		Input( id: "berries", value: "3" ),
		Button( id: "ok", text: "Record" ),
	),
);

win.find_by_id("ok").click( function () {
	let count := win.find_by_id("berries").value();
	say `Zia recorded ${count} berries.`;
	win.close("saved");
} );

win.call();
```

The constructor shape is intentionally regular:

- named properties come first,
- child widgets follow as positional arguments,
- a constructor returns the widget it created.

`Window.call()` starts the window's event loop and returns the value
passed to `close`. `Window.show()` displays a window without blocking.


## 23.2 The Widget Tree

Every visible GUI object is a `Widget`. Widgets form a parent/children
tree, and the object tree remains useful even before anything is shown.

Common operations include:

```zzs
let field := win.find_by_id("berries");

say field.parent().id();       // the containing VBox
say field.visible();           // true by default
field.set_enabled(false);      // mutators return the widget
field.set_enabled(true);
```

Use `find_by_id` for ordinary lookup. It searches depth-first and
returns the first matching widget or `null`.

You can also build trees incrementally:

```zzs
let box := VBox( gap: 4 );
box.add_child( Label( text: "Trail notes" ) );
box.add_child( Text( value: "Zia saw tiny pawprints.", wrap: true ) );
```

Widget trees use ordinary ownership rules: a parent keeps its children
alive, but a child should not keep its parent alive. GUI implementations
therefore use weak parent links and other non-owning back-references
internally. This means scripts should normally remove or replace
children to change the tree shape; they should not need manual
cycle-breaking code just to let a detached subtree be released.

For now, path-query operators such as `@` and `@@` are not part of the
GUI tree API. Keep GUI lookup explicit.


## 23.3 Layout Widgets

The basic layout widgets are `VBox`, `HBox`, and `Frame`.

`VBox` stacks children vertically:

```zzs
VBox(
	gap: 8,
	padding: 10,
	Label( text: "Name" ),
	Input( id: "name" ),
)
```

`HBox` places children side by side:

```zzs
HBox(
	align: "right",
	gap: 6,
	Button( text: "Cancel" ),
	Button( text: "OK", variant: "primary" ),
)
```

`Frame` gives a group a labelled boundary:

```zzs
Frame(
	label: "Raccoon supplies",
	VBox(
		Checkbox( id: "map", label: "Map", checked: true ),
		Checkbox( id: "torch", label: "Torch" ),
	),
)
```

The full property list for every widget type is in Appendix F.


## 23.4 Controls and State

Most controls expose simple getter/setter methods. With no argument, the
method reads the value. With an argument, it sets the value and returns
the widget.

```zzs
let name := Input( value: "Zia" );

say name.value();          // Zia
name.value("Zia R.");
say name.value();          // Zia R.
```

Common control families are:

- text/content: `Label`, `Text`, `RichText`, `Image`,
- input: `Input`, `DatePicker`, `Checkbox`, `Radio`, `RadioGroup`,
  `Select`, `Button`,
- display/progress: `Separator`, `Slider`, `Progress`,
- collection and navigation: `Tabs`, `Tab`, `ListView`, `TreeView`,
- menu: `Menu`, `MenuItem`.

`RichText` values are HTML. The portable subset is deliberately small:
`<b>bold</b>`, `<i>italics</i>`, `<u>underline</u>`, and
`<a href="...">links</a>`. GTK4 may accept additional markup, but that is
backend-specific and should not be treated as portable.

List and tree controls use item arrays. Simple string items are allowed,
and dict items may provide labels and values:

```zzs
let snacks := ListView(
	id: "snacks",
	items: [
		{ label: "Berries", value: "berries" },
		{ label: "Acorns", value: "acorns" },
		{ label: "Biscuits", value: "biscuits" },
	],
	selected_index: 0,
);

say snacks.selected_item(){value};  // berries
```

Tree items use nested `children` arrays:

```zzs
let tree := TreeView(
	items: [
		{
			label: "Forest",
			children: [
				{ label: "Oak" },
				{ label: "Stream" },
			],
		},
	],
	selected_path: [ 0, 1 ],
);
```


## 23.5 Events

Event handlers are ordinary functions. Register them with `on`, `once`,
or the event shortcut methods such as `click`, `change`, `input`,
`select`, and `activate`.

```zzs
let status := Label( id: "status", text: "Ready" );
let slider := Slider( id: "volume", value: 5, min: 0, max: 10 );

slider.change( function () {
	status.text( `Zia set volume to ${slider.value()}.` );
} );
```

Handlers may accept an event object:

```zzs
let button := Button( text: "Inspect" );

button.click( function (ev) {
	say ev.name();
} );
```

For most small GUIs, shortcuts are enough. Use `on` and `off` when you
need to keep listener tokens:

```zzs
let token := button.on( "click", function () {
	say "clicked";
} );

button.off(token);
```

Listener registries are another non-owning relationship. Runtime event
lists use weak storage where a listener or token should not keep its
owning widget alive. Keep your own retained callbacks strong only when
the callback's lifetime is intentionally tied to the current object.


## 23.6 Dialogues

`std/gui/dialogue` provides common modal helpers:

```zzs
from std/gui/dialogue import
	alert,
	confirm,
	prompt,
	file_open,
	colour_picker;

alert("Zia saved the notes.");

if ( confirm("Open the supply list?") ) {
	let path := file_open( title: "Open supply list" );
	say path if path ≢ null;
}

let name := prompt("Raccoon name:", value: "Zia");
let colour := colour_picker( value: "green" );  // returns #008000
```

If GUI access is denied with `--deny=gui`, the dialogue module uses
terminal fallbacks for alert, confirmation, prompts, file and directory
paths, and colour selection. File and directory fallbacks use
`std/tui` readline completion when the runtime supports it.


## 23.7 Data Binding

For simple forms, you can manually copy values between widgets and your
model. For repeated fields, use `bind` and `unbind` from `std/gui`.

```zzs
from std/gui import *;

let model := {
	zia: {
		name: "Zia",
		snack: "berries",
	},
};

let name := Input( id: "name" );
let snack := Input( id: "snack" );

let name_binding := bind( name, "value", model, "/zia/name" );
let snack_binding := bind( snack, "value", model, "/zia/snack" );

// Later, when the form is no longer active:
unbind(name_binding);
unbind(snack_binding);
```

Bindings are intentionally explicit. They do not create a full reactive
framework; they connect one widget property to one model path. The path
can be a compiled path object, or a string compiled using the active
lexical `paths` flavour.


## 23.8 GUI XML

For declarative layouts, `std/gui` can parse and serialize GUI XML:

```zzs
from std/gui import gui_from_xml, gui_to_xml;

let xml := """
<Window xmlns="https://zuzulang.org/ns/std/gui" title="Trail Notes">
	<VBox id="root" gap="6" padding="8">
		<Input id="title" value="Zia's Map" meta.model="note.title" />
		<Button id="save" text="Save" style.role="primary" />
	</VBox>
</Window>
""";

let ui := gui_from_xml(xml);
ui.find_by_id("save").click( function () {
	say ui.find_by_id("title").value();
} );

say gui_to_xml(ui);
```

Important XML rules:

- the supported namespace is
  `https://zuzulang.org/ns/std/gui`,
- unknown elements or attributes throw deterministic exceptions,
- boolean and number attributes are coerced,
- `meta.foo="bar"` becomes `widget.meta("foo", "bar")`,
- `style.foo="bar"` becomes `widget.style("foo", "bar")`,
- `gui_to_xml` preserves tracked `meta.*` and `style.*` attributes.

Use XML when the structure matters more than the construction logic. Use
plain ZuzuScript constructors when the layout is highly dynamic.


## 23.9 Styling and Metadata

Every widget has `style` and `meta` maps:

```zzs
let label := Label( text: "Quiet trail" );

label.style( "colour", "#336699" );
label.meta( "model", "trail.name" );
```

`style` is for visual hints. Backends may ignore unsupported style keys,
so do not put essential application state there.

`meta` is for application data that should travel with the widget. GUI
XML uses `meta.*` keys to preserve this information across round trips.


## 23.10 Capability Notes

GUI support is a runtime capability. Scripts can check:

```zzs
if ( __system__{deny_gui} ) {
	say "Running without GUI support.";
}
```

Plain JavaScript runtimes deny GUI access. The Electron launcher enables
GUI support explicitly. Perl and Rust default to allowing GUI access,
but the command line can deny it:

```bash
zuzu --deny=gui app.zzs
zuzu-rust --deny=gui app.zzs
zuzu-js --electron app.zzs
```

When writing reusable scripts, keep important work separate from the
window itself. Let the GUI collect values and call ordinary functions.
That makes it easier to test the same logic in command-line, GUI, and
headless environments.


## 23.11 Chapter Recap

You can now:

- build a GUI tree with `Window`, layout widgets, and controls,
- find widgets by id,
- handle events,
- use modal dialogue helpers,
- bind widget properties to model paths,
- read and write GUI XML,
- use `style` and `meta` for backend hints and app metadata,
- plan for `--deny=gui` and terminal fallbacks.

Zia's final GUI rule is boring on purpose: keep business logic outside
the window. The clearer the boundary, the easier it is to test, reuse,
and improve.

The same boundary matters everywhere in this guide: command-line tools,
packages, database scripts, web apps, and GUIs all stay healthier when the
core logic is separated from the interface.

The next technical topic is security. Chapter 24 looks at randomness,
password hashing, encryption, signing, certificates, and how to write code
that checks what the current runtime can safely support.
