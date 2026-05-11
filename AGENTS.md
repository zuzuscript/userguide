# ZuzuScript User Guide

This repository contains the ZuzuScript user guide, language reference,
operator tables, BNF, implementation-status appendix, and introductions for
developers coming from other languages.

Use Oxford English: mostly standard British English, with `-ize` word
endings.

## Relationship To Other Projects

The user guide is reference material for runtimes, examples, syntax
highlighters, editor packages, and the website. The BNF and operator
appendix are especially important inputs for parser and syntax-tool work.

When documentation and implementation disagree, do not silently guess. Check
the runtimes, `languagetests`, `stdlib`, and current project intent before
changing normative reference text.

## Project Shape

- `zuzuscript-guide/` contains the main guide and appendices.
- `zuzuscript-guide/AA-bnf.md` is the grammar reference.
- `zuzuscript-guide/AB-operator-precedence.md` documents precedence.
- `zuzuscript-guide/AC-stdlib-reference.md` documents the standard library.
- `zuzuscript-guide/AE-implementation-test-status.md` mirrors matrix status.
- `operators-table.html` is used by docs and syntax-tooling reference work.
- `intros-from-other-languages/` contains audience-specific introductions.

## Documentation Rules

- Keep examples valid ZuzuScript and aligned with the shared style.
- Prefer concise, teachable examples over exhaustive edge-case lists.
- Keep normative syntax claims aligned with BNF, operator precedence,
  language tests, and runtime behaviour.
- Mark implementation limitations clearly when behaviour is not yet
  universal across runtimes.
- Do not manually invent matrix status. Use matrix output when refreshing
  implementation-status documentation.

## Validation

For content-only changes, read the changed Markdown for broken headings,
links, code fences, and stale cross-references.

For syntax, operator, or stdlib-reference changes, also check relevant
`languagetests`, `stdlib` tests, parser code, and syntax tooling consumers.
If examples are executable, run them through the appropriate runtime.

POD examples should use spaces, not tabs. Blank lines inside POD code
samples may need indentation to avoid ending the block.
