# Chapter 1: Hello, World — and Everything After

Welcome to ZuzuScript.

If this is your first time meeting it, imagine a language that likes
small scripts, practical automation, and code that stays readable after
midnight coffee. ZuzuScript is designed for real-world scripting work:
text cleanup, file handling, command-line tools, light data wrangling,
and "this would be annoying in plain shell" moments.

In this chapter, we will get you from zero to running code quickly, and
then we will take a short guided stroll through language features so you
know what is waiting for you in the chapters ahead.


## 1.1 What ZuzuScript is for

ZuzuScript sits in a sweet spot:

- more structured than ad-hoc shell scripts,
- lighter and quicker than starting a full application framework,
- expressive enough to model real program logic without ceremony.

Think of it as a friendly multitool for:

- automation scripts,
- developer tooling,
- command-line data transforms,
- quick integrations with HTTP APIs,
- practical glue code between systems.

### Design goals (the short version)

ZuzuScript aims to be:

1. **Readable first**<br>
   You should be able to return to a script in six months and still
   understand what sleepy-past-you meant.
2. **Expressive for scripting tasks**<br>
   Collections, string handling, regex support, modules, and OO features
   are part of the everyday language, not afterthoughts.
3. **Pragmatic, not precious**<br>
   The language is designed for getting work done. It favours practical
   constructs that help with automation and small tools.
4. **Comfortable at small and medium size**<br>
   One-file scripts feel natural, but you can also grow into modules,
   classes, and larger projects.

## 1.2 Installing and running ZuzuScript

There are a few different implementations of ZuzuScript. You don't need them
all to get started. Installing the Rust or Perl implementation is probably
the best place to start for a new user.

### Installing zuzu-rust

There are pre-built deb files for installing zuzu-rust on Linux systems
at https://zuzulang.org/downloads/.

### Installing zuzu.pl

At minimum, you need Perl available on your machine, and a tool for
installing Perl modules (such as cpanm).

You can install Zuzu using:

  cpanm Zuzu

Once your environment is set up, the main command you will use is:

```text
zuzu
```

That command runs `.zzs` script files and can also start the REPL.

### Quick confidence check

From the repository root, run:

```bash
zuzu -v
```

If everything is wired correctly, you should see a version line.

For a more detailed environment report:

```bash
zuzu -V
```

That prints version info plus library search paths and builtin modules.
This is very handy when module loading is not doing what you expected.

### Useful startup options you will use often

`zuzu` supports a practical set of options. The most beginner-useful
ones are:

- `-I/path/to/lib`  
  Add an extra module search directory.

- `-e 'code'`  
  Run inline snippet(s) without creating a script file.

- `-R` or `--repl`  
  Start the interactive REPL.

- `-d` or `-dN`  
  Enable runtime debug mode (`-d` defaults to level 1).

- `-h`  
  Show help and usage.

- `-v` / `-V`  
  Show version information.

As we progress through the guide, these options will become part of your
muscle memory.


## 1.3 Your first program

Let us do the classic first step.

Create a file named `hello.zzs`:

```zzs
say "Hello, world!";
```

Run it:

```bash
zuzu hello.zzs
```

You should see:

```text
Hello, world!
```

Excellent. You have crossed the first bridge.

### A slightly less traditional hello

ZuzuScript's mascot is Zia, the caffeine-fuelled raccoon.

<img src="https://zuzulang.org/img/zia-sleeping.png" alt="Zia sleeping. Wake up!" class="img-fluid my-2 w-25" />

Now let us give Zia a line:

```zzs
let name := "Zia";
say `Hello from ${name} the raccoon.`;
```

This tiny script already shows a few important things:

- statements end with semicolons,
- double-quoted strings are straightforward,
- `let` introduces a variable binding (`:=`),
- `say` prints a line,
- backtick-quoted strings interpolate variables.

You can leave notes for yourself with comments. Use `//` for a comment
that runs to the end of the line, and `/* ... */` for a comment that can
span several lines:

```zzs
// One-line note.
say "awake";

/*
This block can hold a longer explanation,
or temporarily hide a few lines while you experiment.
*/
say "still awake";
```

For output, `say` is the one you will use most often. It writes to
standard output and adds a newline. `print` also writes to standard
output, but does not add a newline. `warn` writes to standard error and
adds a newline, so it is better for diagnostics than ordinary results:

```zzs
print "status: ";
say "ok";
warn "using default config";
```

### Inline execution with `-e`

Sometimes you just want to try one tiny idea quickly:

```bash
zuzu -e 'say `Zia says hi from inline mode.`;'
```

This is useful for experimentation, testing one expression, or adding
small checks to shell scripts.


## 1.4 REPL vs script files

Both are valuable. Use each where it shines.

### REPL (`zuzu -R`)

REPL means **Read–Eval–Print Loop**: type code, run it immediately,
see output, repeat.

Start it with:

```bash
zuzu -R
```

Great for:

- trying syntax quickly,
- checking behaviour of small expressions,
- exploring APIs,
- teaching and learning interactively.

#### Example REPL session

```text
zuzu (^_^)> let cups := 2;
zuzu (^_^)> say `Zia has ${cups} coffees.`;
Zia has 2 coffees.
```

(Your prompt style may vary by terminal settings.)

### Script files (`*.zzs`)

Script files are your normal home for real tasks.

Great for:

- reusable automation,
- version-controlled tools,
- multi-function programs,
- anything you need to run repeatedly.

### Rule of thumb

- Use the REPL to discover.
- Use files to build.

You can prototype in REPL and then move polished code into `.zzs` files.
That workflow feels very natural in ZuzuScript.


## 1.5 File extensions and execution model

ZuzuScript uses two primary source file extensions:

- `.zzs` — scripts
- `.zzm` — modules

### `.zzs` script files

These are entrypoint scripts you run directly with `zuzu`.

Example:

```bash
zuzu tasks/report.zzs
```

If your script defines `__main__`, extra command-line arguments are
passed to that function.

### `.zzm` module files

Modules package reusable code and can be imported from scripts or other
modules.

You do not typically execute a `.zzm` directly; you import it.

A simple mental model:

- `.zzs` = "do this now"
- `.zzm` = "here is reusable logic"

Both are technically the same format, but using this naming convention
will help you structure your code better.

### How execution works (high level)

When you run `zuzu your-script.zzs`, the runtime roughly does this:

1. reads source text,
2. parses it,
3. evaluates top-level code,
4. calls `__main__` with CLI args if present.

You do not need to know parser internals to be productive, but this
execution model explains many day-to-day behaviours:

- top-level code runs on script load,
- imports happen during evaluation,
- runtime errors appear as execution reaches failing code.


## 1.6 A quick feature tour teaser

Later chapters dive deeply. Here we do short snapshots so your brain has
hooks to attach details onto.

### Values and collections

```zzs
let sleepy := true;
let coffees := [ "espresso", "latte", "mystery thermos" ];
let skills := « "coding", "cartwheels", "napping" »;
let profile := {
	name: "Zia",
	mood: "sleepy",
	coffee_count: 3,
};
```

You get practical value types and rich collection literals out of the
box.

### Control flow

```zzs
if ( profile{coffee_count} > 0 ) {
	say "Zia is operational.";
}
else {
	say "Deploy emergency beans.";
}
```

Readable, familiar, and useful for normal scripting logic.

### Loops

```zzs
for ( let drink in coffees ) {
	say `Inventory: ${drink}`;
}
```

Direct iteration is a core daily feature in scripting work.

### Functions

```zzs
function greet ( name ) {
	return `Hi, ${name}!`;
}

say greet("Zia");
```

Functions make scripts composable and testable.

### Classes and methods

```zzs
class Raccoon {
	let name;

	method announce () {
		say `${name} says hello.`;
	}
}

let zia := new Raccoon( name: "Zia" );
zia.announce();
```

Object-oriented code is useful when your script grows beyond tiny
procedural snippets.

### Modules

```zzs
from std/time import Time;

let now := new Time();
say `Unix time: ${now.epoch}`;
```

ZuzuScript ships with practical modules for tasks like I/O, data,
networking, and process work.


## 1.7 A realistic mini-script

Let us combine a few ideas into something that feels like an actual
useful script.

Imagine a tiny morning report for Zia.

Create `zia-morning.zzs`:

```zzs
from std/time import Time;

function energy_label ( cups ) {
	return "hibernation mode"  if cups ≤ 0;
	return "gentle startup"    if cups = 1;
	return "stable raccoon"    if cups = 2;
	return "zoomies imminent";
}

function __main__ ( args ) {
	let name  := "Zia";
	let cups  := 2;
	let stamp := new Time();

	say `Morning report for ${name}`;
	say `timestamp: ${stamp.datetime}`;
	say `coffee cups: ${cups}`;
	say `status: ${ energy_label(cups) }`;

	for ( let note in args ) {
		say `note: ${note}`;
	}
}
```

Run it:

```bash
zuzu zia-morning.zzs "remember cartwheels"
```

What this demonstrates:

- importing a standard module,
- defining helper functions,
- using `__main__` as an entrypoint,
- reading command-line args,
- simple branching and output.

That is already enough structure for many practical utility scripts.

## 1.8 Common first-day pitfalls (and easy fixes)

### "Command not found" when running `zuzu`

If `zuzu` is not installed globally, run via repo path:

```bash
/path/to/zuzu your-script.zzs
```

### Script cannot find your module

Use `-I` to add your module directory:

```bash
zuzu -I./modules your-script.zzs
```

### REPL errors after pasting partial code

If a block is incomplete, continue typing until syntax closes. If you
get into a messy state, a single ";" on a line should fix things.

### Unicode or terminal weirdness

The CLI is UTF-8 aware. If output looks wrong, check terminal encoding
settings first.

### Unsure what options exist

Use help:

```bash
zuzu -h
```

No heroics required.


## 1.9 How to use this guide effectively

A recommended rhythm:

1. Read one section.
2. Type every sample yourself.
3. Change one thing and rerun.
4. Break it on purpose.
5. Fix it.

That "change + break + fix" loop is where the learning sticks.


## 1.10 Tiny practice exercises

Try these now before moving to Chapter 2.

1. **Hello with variable**<br>
   Write a script that stores your name in `let who := ...` and prints a
   greeting with `say`.
2. **REPL warmup**<br>
   In REPL mode, create a list with three items and print each item in a
   loop.
3. **`__main__` argument check**<br>
   Write a script that prints `args[0]` if present, otherwise prints
   "no argument".
4. **Mini status report**<br>
   Build a 10-20 line script that prints:
   - current timestamp,
   - a computed status string,
   - one value from command-line input.

If you can do these without looking everything up, you are in great
shape for the next chapter.


## 1.11 Chapter recap

You now know:

- what ZuzuScript is trying to optimize for,
- how to run scripts with `zuzu`,
- how comments, `say`, `print`, and `warn` work at the first-script level,
- when to use REPL vs files,
- the meaning of `.zzs` and `.zzm`,
- the high-level execution model,
- a preview of functions, control flow, collections, classes, and modules.

In Chapter 2, we will go deeper into numbers and mathematical operators.

Bring coffee. Zia definitely will.
