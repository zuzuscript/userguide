# Appendix B: Operators

This document lists ZuzuScript expression operators, ordered from highest
precedence to lowest precedence.

## Precedence summary table (highest to lowest)

### Notes

- Higher rows bind tighter than lower rows.
- Most binary operators are left-associative.
- `**` is right-associative.
- `▷` is left-associative; `◁` is right-associative. Their ASCII-safe
  aliases are `|>` and `<|`. The Unicode forms are canonical.
- `«` (left shift) and `»` (right shift) have the ASCII-safe aliases
  `<<` and `>>`. The Unicode forms are canonical. Inside a set literal,
  a bare `>>` or `»` always closes the literal rather than acting as a
  shift; parenthesize shift expressions used as set elements.
- `⊤` and `⊥` are boolean literals, not operators, so they do not
  appear in the precedence table.
- `?:`, `? :`, chain operators, and assignment operators are parsed
  after the main Pratt precedence table. Chain operators bind looser
  than ternary expressions and tighter than assignment.
- `default` is a left-associative binary operator at the type-aware
  equality tier.
- Postfix forms (call/member/index/dict access/postfix `++`/`--`) are
  parsed immediately after primaries and therefore bind tighter than all
  infix operators.
- Argument spread is call syntax, not an infix operator. In
  `fn(...opts default fallback)`, the spread operand is the whole
  `opts default fallback` expression.
- At expression start, `->` and `→` begin a leading-arrow lambda. In
  infix position, `->` remains the existing comparison-level operator.

| Level | Associativity | Operators |
|---|---|---|
| Postfix | left-to-right chaining | `(...)` (call), `.name`, `.(expr)(...)`, `[index]`, `[start:length]`, `{key}`, postfix `++`, postfix `--` |
| Prefix | right-to-left nesting | unary `+`, unary `-`, `!`, `¬`, unary `~`, unary `√`, `⌊...⌋`, `⌈...⌉`, unary `\\` (reference), `not`, `abs`, `sqrt`, `floor`, `ceil`, `round`, `int`, `uc`, `lc`, `length`, `typeof`, prefix `++`, prefix `--` |
| 14 | right-to-left | `**` |
| 13 | left-to-right | `*`, `/`, `×`, `÷`, `mod` |
| 12 | left-to-right | `+`, `-` |
| 11 | left-to-right | `_` |
| 10 | left-to-right | `union`, `⋃`, `intersection`, `⋂`, `\\`, `∖` |
| 9 | left-to-right | `«`, `»` |
| 8 | left-to-right | `&` |
| 7 | left-to-right | `^` |
| 6 | left-to-right | `\|` |
| 5 | left-to-right | `=`, `≠`, `<`, `>`, `<=`, `≤`, `>=`, `≥`, `<=>`, `≶`, `≷`, `eq`, `ne`, `gt`, `ge`, `lt`, `le`, `cmp`, `eqi`, `nei`, `gti`, `gei`, `lti`, `lei`, `cmpi`, `in`, `∈`, `∉`, `subsetof`, `⊂`, `supersetof`, `⊃`, `equivalentof`, `⊂⊃`, `instanceof`, `does`, `can`, binary `~`, `@`, `@?`, `@@` |
| 4 | left-to-right | `==`, `≡`, `!=`, `≢`, `default` |
| 3 | left-to-right | `and`, `⋀`, `nand`, `⊼` |
| 2 | left-to-right | `xor`, `⊻` |
| 1 | left-to-right | `or`, `⋁` |
| Ternary | right-to-left grouping in practice | `? :`, `?:` |
| Chain | by direction | `▷`, `◁`, `\|>`, `<\|` |
| Assignment | right-to-left grouping in practice | `:=`, `~=`, `+=`, `-=`, `*=`, `×=`, `/=`, `÷=`, `**=`, `_=`, `?:=` |

## Detailed table

INCLUDE:../operators-table.html

## Ambiguous tokens by role

Some tokens are valid in more than one role:

- `~` is both unary bitwise-not (prefix) and binary regexp-match.
- `\\` is both unary reference (prefix) and binary set difference.
- `+` and `-` can be unary (prefix) or binary arithmetic.
- `++` and `--` can be prefix or postfix.
- `...` is a range operator inside collection literals, a variadic marker
  in parameter lists, and argument spread inside call argument lists.

When mixing forms, parentheses are recommended for readability.

This matters especially for path lvalues. When applying prefix/postfix
`++` or `--`, or unary reference `\\`, to `data @ path`, `data @@ path`,
or `data @? path`, parenthesize the path expression:

```zzs
( data @ "/meta/count" )++;
++ ( data @@ "/users/*/age" );
\( data @? "/meta/title" );
```
