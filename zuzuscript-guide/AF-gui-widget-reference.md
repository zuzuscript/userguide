# Appendix F: GUI Widget Reference

This appendix lists the `std/gui` widget types and their constructor
properties. All constructors accept child widgets after named
properties unless otherwise noted.

All widgets expose the base `Widget` object API at runtime:

- `id()`, `set_id(id)`,
- `parent()`, `children()`, `add_child(widget)`,
  `remove_child(widget)`,
- `enabled()`, `set_enabled(value)`,
- `visible()`, `set_visible(value)`,
- `width(value?)`, `height(value?)`, `minwidth(value?)`,
  `minheight(value?)`, `maxwidth(value?)`, `maxheight(value?)`,
- `classes()`, `add_class(name)`, `remove_class(name)`,
- `style(key, value?)`, `meta(key, value?)`,
- `find_by_id(id)`,
- `on(name, handler)`, `once(name, handler)`, `off(token)`,
  `emit(name)`,
- event shortcuts such as `click`, `input`, `change`, `select`,
  `activate`, `open`, `close_request`, `closed`, `expand`, and
  `collapse` where meaningful.

Constructor helpers validate their named properties strictly. The
properties below are the properties accepted by the helper functions in
`std/gui`.


## Common Constructor Properties

Most widget constructors accept:

| Property | Type | Default | Meaning |
|---|---:|---:|---|
| `id` | String or Null | `null` | Optional tree lookup id. |
| `visible` | Bool | `true` | Whether the widget is visible. |
| `enabled` | Bool | `true` | Whether the widget accepts interaction. |
| `disabled` | Bool | `false` | Convenience inverse of `enabled`. |

Many widgets also accept geometry properties:

| Property | Type | Default | Meaning |
|---|---:|---:|---|
| `width` | Number or Null | `null` | Requested width. |
| `height` | Number or Null | `null` | Requested height. |
| `minwidth` | Number or Null | `null` | Minimum width. |
| `minheight` | Number or Null | `null` | Minimum height. |
| `maxwidth` | Number or Null | `null` | Maximum width. |
| `maxheight` | Number or Null | `null` | Maximum height. |

XML accepts geometry attributes for all widget elements. Constructor
helpers currently accept geometry only where listed in the tables below.


## Window

Top-level window.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `title` | String | `""` |
| `width` | Number | `800` |
| `height` | Number | `600` |
| `minwidth` | Number or Null | `null` |
| `minheight` | Number or Null | `null` |
| `maxwidth` | Number or Null | `null` |
| `maxheight` | Number or Null | `null` |
| `resizable` | Bool | `true` |
| `modal` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

Primary methods: `show()`, `call()`, `close(result?)`, `content()`,
`set_content(widget)`, `menus()`.


## Layout Widgets

### VBox

Vertical container.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `align` | String | `"top"` | `top`, `centre`, `bottom`, `stretch` |
| `gap` | Number | `0` | |
| `padding` | Number or Array | `0` | |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |

### HBox

Horizontal container.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `align` | String | `"left"` | `left`, `centre`, `right`, `stretch` |
| `gap` | Number | `0` | |
| `padding` | Number or Array | `0` | |
| `width`, `height`, `minwidth`, `minheight`, `maxwidth`, `maxheight` | Number or Null | `null` | |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |

### Frame

Labelled grouping container.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `label` | String | `""` |
| `collapsible` | Bool | `false` |
| `collapsed` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |


## Content Widgets

### Label

Static text label.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `text` | String | `""` |
| `for` | String or Null | `null` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

Primary methods: `text(value?)`, `for_id(value?)`, `set_for_id(value)`.

### Text

Plain text display.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `value` | String | `""` |
| `multiline` | Bool | `false` |
| `readonly` | Bool | `false` |
| `wrap` | Bool | `true` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

### RichText

Formatted text display. The value is HTML. Portable markup support is
limited to `<b>bold</b>`, `<i>italics</i>`, `<u>underline</u>`, and
`<a href="...">links</a>`. Other HTML may work on one backend but should
not be relied on across runtimes.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `value` | String | `""` | |
| `multiline` | Bool | `true` | |
| `readonly` | Bool | `true` | |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |

### Image

Image display.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `src` | String | `""` | |
| `alt` | String | `""` | |
| `fit` | String | `"none"` | `none`, `contain`, `cover`, `stretch` |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |


## Input Widgets

### Input

Text input.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `value` | String | `""` |
| `placeholder` | String | `""` |
| `multiline` | Bool | `false` |
| `readonly` | Bool | `false` |
| `password` | Bool | `false` |
| `required` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

Main events: `input`, `change`.

### DatePicker

Date input.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `value` | String or Null | `null` |
| `min` | String or Null | `null` |
| `max` | String or Null | `null` |
| `first_day_of_week` | Number | `0` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

Dates are represented as strings, normally `YYYY-MM-DD`.

### Checkbox

Boolean checkbox.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `label` | String | `""` |
| `checked` | Bool | `false` |
| `indeterminate` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

### Radio

Radio button.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `label` | String | `""` |
| `value` | Any | `""` |
| `group` | String or Null | `null` |
| `checked` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

### RadioGroup

Container for related radio buttons.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `name` | String | `""` |
| `value` | Any | `null` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

### Select

Drop-down selection control.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `value` | Any | `null` |
| `options` | Array | `[]` |
| `multiple` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

Option items may be strings or dicts with `label` and `value`.

### Button

Clickable button.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `text` | String | `""` | |
| `variant` | String | `"default"` | `default`, `primary`, `danger` |
| `width`, `height`, `minwidth`, `minheight`, `maxwidth`, `maxheight` | Number or Null | `null` | |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |

Main event: `click`.


## Menu Widgets

### Menu

Menu container, normally a direct `Window` child.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `text` | String | `""` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

### MenuItem

Menu command item.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `text` | String | `""` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |


## Extended Controls

### Separator

Visual divider.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `orientation` | String | `"horizontal"` | `horizontal`, `vertical` |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |

### Slider

Range slider.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `value` | Number | `0` | |
| `min` | Number | `0` | |
| `max` | Number | `100` | |
| `step` | Number | `1` | |
| `orientation` | String | `"horizontal"` | `horizontal`, `vertical` |
| `readonly` | Bool | `false` | |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |

### Progress

Progress indicator.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `value` | Number | `0` |
| `min` | Number | `0` |
| `max` | Number | `100` |
| `indeterminate` | Bool | `false` |
| `show_text` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |


## Tab Controls

### Tabs

Tab-page container.

| Property | Type | Default | Allowed values |
|---|---:|---:|---|
| `id` | String or Null | `null` | |
| `selected` | String or Null | `null` | |
| `placement` | String | `"top"` | `top`, `bottom`, `left`, `right` |
| `visible` | Bool | `true` | |
| `enabled` | Bool | `true` | |
| `disabled` | Bool | `false` | |

Primary methods: `selected(value?)`, `selected_tab()`.

### Tab

One page inside `Tabs`.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `title` | String | `""` |
| `value` | String | `""` |
| `selected` | Bool | `false` |
| `closable` | Bool | `false` |
| `icon` | String or Null | `null` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |


## Collection Controls

### ListView

Flat item list.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `items` | Array | `[]` |
| `selected_index` | Number or Null | `null` |
| `multiple` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

Primary methods: `items()`, `selected_index(value?)`,
`selected_item()`, `add_item(item)`, `clear_items()`,
`activate_index(index)`.

Main events: `select`, `activate`.

### TreeView

Hierarchical item tree.

| Property | Type | Default |
|---|---:|---:|
| `id` | String or Null | `null` |
| `items` | Array | `[]` |
| `selected_path` | Array | `[]` |
| `multiple` | Bool | `false` |
| `visible` | Bool | `true` |
| `enabled` | Bool | `true` |
| `disabled` | Bool | `false` |

Primary methods: `items()`, `selected_path(value?)`,
`selected_item()`, `add_item(item)`, `clear_items()`,
`activate_path(path)`, `expand_path(path)`, `collapse_path(path)`,
`is_expanded(path)`.

Tree items may include `children`. Main events are `select`,
`activate`, `expand`, and `collapse`.


## Event Objects

Handlers may receive an `Event` object. Important methods include:

| Member | Meaning |
|---|---|
| `name()` | Event name. |
| `target()` | Widget that emitted the event. |
| `current_target()` | Widget currently handling the event. |
| `window()` | Window that owns the event target. |
| `phase()` | Current event phase. |
| `timestamp()` | Event time. |
| `data()` | Arbitrary event payload. |
| `cancelled()` | Whether propagation or default handling was cancelled. |
| `default_prevented()` | Whether default handling was prevented. |
| `stop_propagation()` | Stop further propagation. |
| `prevent_default()` | Mark the default action as prevented. |


## Dialogue Helpers

`std/gui/dialogue` is not a widget module, but it is part of the GUI
surface. It exports:

- `alert`, `alert_window`,
- `confirm`, `confirm_window`,
- `prompt`, `prompt_window`,
- `file_open`, `file_open_window`,
- `file_save`, `file_save_window`,
- `directory_open`, `directory_open_window`,
- `directory_save`, `directory_save_window`,
- `colour_picker`, `colour_picker_window`.

The non-window helpers use terminal fallbacks when `__system__{deny_gui}`
is true.
