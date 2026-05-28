# Chapter 25: Roundup and Where to Go Next

You have reached the end of the guided tour.

The early chapters started with small pieces: values, names, expressions,
strings, collections, truth, and control flow. Then the guide widened out:
functions, objects, errors, modules, patterns, nested data, chains,
concurrency, files, processes, command-line tools, packages, databases,
HTTP clients, web apps, GUIs, and security.

That is a lot of ground. The point was not to memorize every method name
or option. The point was to build a map. You should now be able to look at
a scripting problem and ask useful questions:

- What data do I have?
- What shape should it be in?
- What can fail?
- What should be a function, class, or module?
- What belongs in configuration?
- What needs a test?
- What boundary am I crossing: filesystem, process, database, network,
  browser, GUI, or cryptographic system?

Once you can ask those questions, the appendices and standard-library
reference become much more useful. You no longer need to know everything
from memory. You need to know where to look.


## 25.1 The Shape of the Guide

Chapters 1-6 gave you the core language:

- running scripts,
- numbers and operators,
- strings,
- arrays, hashes, sets, and bags,
- `Null` and booleans,
- `if`, `while`, `for`, `given`, `when`, `break`, `continue`, and
  `return`.

Those chapters are the everyday syntax you will use even in tiny scripts.
When a program feels confusing, it often helps to come back to these
basics and ask whether the values are simple enough.

Chapters 7-13 showed how to organize larger thoughts:

- functions,
- classes, methods, roles, and objects,
- weak references and bound method values,
- exceptions, `Result`, `debug`, and `assert`,
- modules and imports,
- regular expressions,
- paths, destructuring, and nested structures,
- chains for readable data flow.

This is where scripts become maintainable. You learnt to name repeated
work, isolate errors, put reusable code into modules, and turn deep data
access into something a reader can follow.

Chapters 14-20 moved out into the world:

- concurrent tasks and workers,
- filesystem IO,
- JSON, XML, and other structured data,
- diagnostic dumping,
- child processes,
- environment and system state,
- time and date handling,
- command-line options,
- configuration,
- tests,
- ZDF-1 packages,
- Zuzuzoo installation,
- SQL databases,
- CSV,
- client-side HTTP and URLs.

This is the practical automation layer. Most useful scripts cross at least
one of these boundaries.

Chapters 21-24 then looked at user-facing and trust-sensitive programs:

- raw web protocol handlers,
- routing, requests, responses, templates, cookies, and forms,
- GUI windows, widgets, bindings, and event handlers,
- randomness, password hashing, encryption, signing, certificates, and
  security capability checks.

By the end of those chapters, the same pattern should be visible
everywhere: keep the core logic clear, keep boundaries explicit, and use
the runtime's helpers instead of inventing protocols by hand.


## 25.2 A Practical Problem-Solving Checklist

When you start a new ZuzuScript program, try this sequence.

1. **Write the smallest useful version first.**<br>
   A script that reads one file, transforms one value, or prints one
   useful result is better than a perfect design that never runs.

2. **Name the data shape.**<br>
   Decide whether your main data is an array, hash, object, database row,
   CSV record, JSON document, XML tree, or something else. Many hard
   problems get easier once the shape is honest.

3. **Move repeated work into functions.**<br>
   If you copy three lines twice, stop and name them. A good function name
   is a note to your future self.

4. **Choose the right boundary.**<br>
   A one-file script is fine. A module is better when code is shared. A
   distribution is better when other people should install it. A web app
   or GUI may be a good idea when users need an interface.

5. **Plan failure early.**<br>
   Use exceptions for exceptional failure, `Result` when callers should
   branch on success or failure, and clear exit codes for command-line
   scripts.

6. **Add tests before the script gets clever.**<br>
   `test/more` is not only for libraries. It is also a way to lock down
   examples, parsers, data transforms, and bug fixes.

7. **Check portability before depending on host features.**<br>
   Processes, databases, GUIs, browser APIs, and cryptographic algorithms
   may not be available everywhere. Use capability checks and keep
   fallbacks simple.


## 25.3 Project Ideas

The best next project is small enough that you can finish it, but large
enough that you have to use more than one chapter.

### Small Scripts

Try these when you want quick practice:

- a file renamer that previews changes before applying them,
- a CSV cleaner that trims fields, normalizes names, and reports bad rows,
- a log summarizer that groups messages by level or source,
- a directory inventory tool that writes JSON,
- a command-line calculator for a hobby, bill, score, or schedule,
- a link checker that reads URLs from a file and makes `HEAD` requests.

These projects are good places to practise strings, collections, regexps,
paths, files, HTTP, options, and error messages.

### Medium Tools

When you want something with structure, try:

- a personal notes index with tags, search, and JSON storage,
- a SQLite-backed habit tracker,
- a CSV-to-database importer with a `--dry-run` mode,
- a command-line bookmark manager that can export HTML or CSV,
- a small HTTP API client with configuration profiles,
- a report generator that reads a database and writes Markdown.

These projects will push you toward modules, tests, configuration,
database code, and clearer error handling.

### User-Facing Projects

When you are ready to build something with an interface, try:

- a tiny web app for browsing a SQLite database,
- a form-driven web tool for validating pasted CSV,
- a GUI front end for one of your command-line scripts,
- a local dashboard that combines files, processes, and HTTP data,
- a package of reusable helper modules for your own scripts.

The useful habit is to keep the core logic independent of the interface.
If the same module can be used from a command-line script, a web route, and
a GUI event handler, you are probably on the right path.

### Stretch Projects

These are deliberately more ambitious:

- a static site generator,
- a small test runner for your own examples,
- a migration tool that copies data between CSV and SQL,
- a package publishing workflow around ZDF-1 archives,
- a secure token tool that stores only hashed or encrypted data,
- a multi-worker batch processor for slow network tasks.

Stretch projects teach design pressure. Keep them boring at first. Make
one path work, test it, then widen it.


## 25.4 Quick Paths Through the Appendices

The appendices are not only for careful reading. They are also quick lookup
tools.

Use **Appendix A**,
[ZuzuScript Grammar](AA-bnf.md), when:

- a syntax error surprises you,
- you want to know whether a construct is part of the language,
- you are working on a parser, highlighter, formatter, or editor tool.

Use **Appendix B**,
[Operators](AB-operator-precedence.md), when:

- an expression has more than one operator,
- you are unsure about precedence or associativity,
- you are reading dense code and want to remove ambiguity.

Use **Appendix C**,
[Standard Library Reference](AC-stdlib-reference.md), when:

- you know the module you want but not the exact function name,
- you need a quick list of exports,
- you are checking whether a helper already exists,
- you want to find modules such as `std/dump`, `std/eval`, or `std/time`
  quickly.

Use **Appendix D**,
[Zuzu Distribution Packaging Format](AD-distribution-format.md), when:

- you are building or inspecting a ZDF-1 package,
- you need the manifest fields,
- you want to understand package contents, hashes, and metadata.

Use **Appendix E**,
[Implementation Test Status](AE-implementation-test-status.md), when:

- portability matters,
- you need to know which runtimes currently pass which shared tests,
- behaviour appears different between implementations.

Use **Appendix F**,
[GUI Widget Reference](AF-gui-widget-reference.md), when:

- you are building with `std/gui`,
- you need widget property names,
- you want to know which controls and layout widgets exist.

Use **Appendix G**,
[Collection Methods](AG-collection-methods.md), when:

- you are working with arrays, hashes, sets, or bags,
- you remember the operation but not the method name,
- a chain would be clearer than a loop.

Use **Appendix H**,
[`std/secure` Feature Support](AH-secure-feature-support.md), when:

- a security feature must work across runtimes,
- you need to choose a portable cryptographic algorithm,
- browser support, async support, or capability probing matters.

Use **Appendix I**,
[Keywords](AI-keywords.md), when:

- you are choosing names for variables, functions, methods, or modules,
- generated code needs to avoid reserved words,
- syntax highlighting or parsing needs a definitive keyword list,
- you need a quick reminder of special names such as `__argc__`,
  `__file__`, or `__global__`.

Use **Appendix J**,
[Built-In Classes and Types](AJ-built-in-classes-and-types.md), when:

- you need to check the basic type model,
- you are unsure what operations belong to a built-in class,
- you are documenting or debugging values at runtime,
- you see `AssertionException`, `Method`, or another runtime type name.


## 25.5 Where to Look First

Here is a faster version.

| Need | Start With |
| --- | --- |
| Syntax shape | Appendix A |
| Operator precedence | Appendix B |
| Standard-library function names | Appendix C |
| Dumping, eval, or time helpers | Appendix C |
| Package format details | Appendix D |
| Runtime compatibility | Appendix E |
| GUI widget properties | Appendix F |
| Collection method names | Appendix G |
| Security capability support | Appendix H |
| Reserved words | Appendix I |
| Special keyword names | Appendix I |
| Built-in types | Appendix J |

For ordinary programming, Appendix C and Appendix G are likely to be the
ones you open most often. For language tooling, start with Appendices A,
B, I, and J. For shipping code to other people, keep Appendices D and E
nearby. For GUI or security work, Appendices F and H will save time.


## 25.6 How to Keep Improving

The next stage is repetition with slightly larger problems.

Write scripts you actually use. When one gets messy, split out a function.
When two scripts share code, make a module. When a module becomes useful,
add tests. When someone else should install it, package it. When the
program needs a human interface, choose a command line, web route, or GUI
based on the job.

Most of the guide can be read again in that order:

- chapters 2-6 when values or control flow feel unclear,
- chapters 7-13 when code needs structure,
- chapters 14-20 when the program talks to the outside world,
- chapters 21-23 when users need an interface,
- chapter 24 when secrets, identities, or trust boundaries are involved.

Do not wait until you feel ready to build something real. Build something
small, let it be awkward, then improve it with the tools you now have.


## 25.7 Final Recap

You have seen ZuzuScript as:

- a calculator,
- a string and collection language,
- a control-flow language,
- a module system,
- an object system,
- a language with weak references and bound methods,
- a pattern-matching tool,
- a data wrangling tool,
- a concurrent task runner,
- a filesystem and process automation language,
- a runtime-inspection and debugging language,
- a command-line scripting language,
- a packageable library language,
- a database and CSV language,
- an HTTP client,
- a web server language,
- a GUI language,
- and a language with portable security helpers.

That is the real lesson of the guide: ZuzuScript is not one trick. It is a
practical scripting language for turning small pieces of logic into tools
you can run, test, share, and improve.

Keep the appendices close, keep examples small, and keep building.
