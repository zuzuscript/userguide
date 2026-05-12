# Appendix A: ZuzuScript Grammar (BNF-style)

This file describes a language-level grammar for ZuzuScript.

Notes:

* This is a human-readable BNF/EBNF hybrid.
* `*` means zero or more, `+` means one or more, `?` means optional.
* Literal tokens are quoted.
* Some constructs are described lexically (comments, strings, numbers).

## 1. Top-level

```
<program> ::= <statement-list> <eof>

<statement-list> ::= ( <statement> ( ";" )* )*

<statement> ::= <block>
	| <let-decl>
	| <const-decl>
	| <function-def>
	| <class-def>
	| <trait-def>
	| <if-stmt>
	| <while-stmt>
	| <for-stmt>
	| <switch-stmt>
	| <try-catch-stmt>
	| <return-stmt>
	| <next-stmt>
	| <continue-stmt>
	| <last-stmt>
	| <throw-stmt>
	| <die-stmt>
	| <import-stmt>
	| <assignment-stmt>
	| <expr-stmt>
	| <postfix-conditional-stmt>

<block> ::= "{" <statement-list> "}"

<expr-stmt> ::= <expression>

<postfix-conditional-stmt> ::= <simple-statement> "if" <expression>
	| <simple-statement> "unless" <expression>

<simple-statement> ::= <expr-stmt>
	| <assignment-stmt>
	| <next-stmt>
	| <continue-stmt>
	| <last-stmt>
	| <return-stmt>
```

## 2. Declarations and assignments

```
<let-decl> ::= "let" <typed-identifier>
	( <weak-storage-modifier>
	| ":=" <expression> <weak-storage-modifier>?
	)?

<const-decl> ::= "const" <typed-identifier>
	( <weak-storage-modifier>
	| ":=" <expression> <weak-storage-modifier>?
	)?

<weak-storage-modifier> ::= "but" "weak"

<typed-identifier> ::= <identifier>
	| <identifier> <identifier>

<assignment-stmt> ::= <assignable> ":=" <expression> <weak-write-modifier>?
	| <assignable> <compound-assign-op> <expression>
	| <assignable> "~=" <expression> <lambda-arrow> <expression>

<assign-op> ::= ":=" | <compound-assign-op>

<compound-assign-op> ::= "+=" | "-=" | "*=" | "×=" | "/=" | "÷="
	| "**=" | "_=" | "?:="

<weak-write-modifier> ::= "but" "weak"

<assignable> ::= <identifier>
	| <index-expr>
	| <slice-expr>
	| <dict-access-expr>
	| <path-first-expr>
	| <path-all-expr>
	; "@?" expressions are intentionally excluded from assignable targets.

<index-expr> ::= <postfix-expr> "[" <expression> "]"

<slice-expr> ::= <postfix-expr> "[" <expression>? ":" <expression>? "]"

<dict-access-expr> ::= <postfix-expr> "{" <dict-key-expr> "}"
```

## 3. Control flow

```
<if-stmt> ::= "if" "(" <expression> ")" <block>
	( "else" "if" "(" <expression> ")" <block> )*
	( "else" <block> )?

<while-stmt> ::= "while" "(" <expression> ")" <block>

<for-stmt> ::= "for" "(" ( "let" | "const" )? <identifier> "in" <expression> ")" <block>
	( "else" <block> )?

<switch-stmt> ::= "switch" "(" <expression> ( ":" <switch-operator> )? ")"
	"{" <switch-case>+ <switch-default>? "}"

<switch-case> ::= "case" <expression-list> ":" <statement-list>

<switch-default> ::= "default" ":" <statement-list>

<switch-operator> ::= <binary-op>

<expression-list> ::= <expression> ( "," <expression> )*

<return-stmt> ::= "return" <expression>?

<next-stmt> ::= "next"

<continue-stmt> ::= "continue"

<last-stmt> ::= "last"

<try-catch-stmt> ::= "try" <block> <catch-clause>+

<catch-clause> ::= "catch" <catch-signature>? <block>

<catch-signature> ::= "(" <catch-binding>? ")"

<catch-binding> ::= <type-expr> <identifier>
	| <identifier>

<throw-stmt> ::= "throw" <expression>

<die-stmt> ::= "die" <expression>
```

## 4. Imports

```
<import-stmt> ::= "from" <module-path> ( "try" )? "import" <import-list>
	( ( "if" | "unless" ) <expression> )?

<module-path> ::= <identifier> ( "/" <identifier> )*

<import-list> ::= <import-select> ( "," <import-select> )*

<import-select> ::= "*"
	| <import-item>

<import-item> ::= <identifier> ( "as" <identifier> )?
```

## 5. Functions, classes, traits

```
<async-modifier> ::= "async"

<function-def> ::= <async-modifier>? "function" <identifier> "(" <param-list>? ")"
	<return-annotation>? <block>

<return-annotation> ::= <lambda-arrow> <type-expr>

<param-list> ::= <required-param-list> ( "," <optional-param-list> )?
	( "," <capture-section> )?
	| <optional-param-list> ( "," <capture-section> )?
	| <capture-section>

<required-param-list> ::= <required-param> ( "," <required-param> )*

<optional-param-list> ::= <optional-param> ( "," <optional-param> )*

<required-param> ::= <identifier>
	| <type-expr> <identifier>

<optional-param> ::= <required-param> "?"
	| <required-param> ":=" <expression>

<capture-section> ::= "..." <capture-param-list>

<capture-param-list> ::= <capture-param>
	| <capture-param> "," <capture-param>

<capture-param> ::= <identifier>
	| <type-expr> <identifier>

<lambda-expr> ::= <async-modifier>? "fn" <lambda-params> <lambda-arrow> <expression>
	| <async-modifier>? "function" "(" <param-list>? ")"
		<return-annotation>? <block>

<lambda-params> ::= <identifier>
	| <type-expr> <identifier>
	| "(" <param-list>? ")"

<lambda-arrow> ::= "->" | "→"

<class-def> ::= "class" <identifier> <class-tail>

<class-tail> ::= ( "extends" <type-expr> )?
	( ( "with" | "but" ) <type-expr> ( "," <type-expr> )* )?
	( <block> | ";" )

<trait-def> ::= "trait" <identifier> ( <block> | ";" )

<class-member> ::= <class-field-decl>
	| <method-def>
	| <static-method-def>
	| <class-def>

<class-field-decl> ::= ( "let" | "const" ) <typed-identifier>
	<field-accessor-list>?
	( <weak-storage-modifier>
	| ":=" <expression> <weak-storage-modifier>?
	)?

<field-accessor-list> ::= "with" <field-accessor-kind>
	( "," <field-accessor-kind> )*

<field-accessor-kind> ::= "get" | "set" | "clear" | "has"

<method-def> ::= <async-modifier>? "method" <identifier> "(" <param-list>? ")"
	<return-annotation>? <block>

<static-method-def> ::= <async-modifier>? "static" "method" <identifier>
	"(" <param-list>? ")" <return-annotation>? <block>

<super-call-expr> ::= "super" "(" <arg-list>? ")"
```

Canonical async static methods use `async static method`. Perl accepts
`static async method` for compatibility, but warns.

## 6. Expressions

```
<expression> ::= <let-expr>
	| <try-catch-expr>
	| <do-expr>
	| <await-expr>
	| <spawn-expr>
	| <assign-expr>
	| <ternary-expr>

<let-expr> ::= "let" <typed-identifier>
	( <weak-storage-modifier>
	| ":=" <expression> <weak-storage-modifier>?
	)?
	| "const" <typed-identifier>
	( <weak-storage-modifier>
	| ":=" <expression> <weak-storage-modifier>?
	)?

<try-catch-expr> ::= "try" <block> <catch-clause>+

<do-expr> ::= "do" <block>

<await-expr> ::= "await" <block>

<spawn-expr> ::= "spawn" <block>

<assign-expr> ::= <assignable> <assign-op> <expression>
	| <assignable> "~=" <expression> <lambda-arrow> <expression>

<ternary-expr> ::= <binary-expr>
	( "?" <expression> ":" <expression>
	| "?:" <expression> )?

<binary-expr> ::= <unary-expr> ( <binary-op> <unary-expr> )*

<binary-op> ::= "+" | "-" | "*" | "×" | "/" | "÷" | "**" | "mod"
	| "=" | "!=" | "≠" | "<" | ">" | "<=" | "≤" | ">=" | "≥"
	| "<=>" | "≶" | "≷"
	| "==" | "≡" | "!=" | "≢"
	| <concat-operator>
	| "&" | "|" | "^"
	| "eq" | "ne" | "gt" | "ge" | "lt" | "le" | "cmp"
	| "eqi" | "nei" | "gti" | "gei" | "lti" | "lei" | "cmpi"
	| "and" | "or" | "xor" | "nand"
	| "⋀" | "⋁" | "⊻" | "⊼"
	| "in" | "∈" | "∉"
	| "union" | "⋃"
	| "intersection" | "⋂"
	| "subsetof" | "⊂"
	| "supersetof" | "⊃"
	| "equivalentof" | "⊂⊃"
	| "\\" | "∖"
	| "instanceof" | "does" | "can"
	| "@" | "@?" | "@@"
	| "~"

<unary-expr> ::= <prefix-op> <unary-expr>
	| <lvalue-ref-expr>
	| <incdec-prefix-expr>
	| <postfix-expr>

<lvalue-ref-expr> ::= "\\" <assignable>

<incdec-prefix-expr> ::= ( "++" | "--" ) <assignable>

<prefix-op> ::= "+" | "-" | "!" | "not" | "¬"
	| "~"
	| "abs" | "sqrt" | "√"
	| "floor" | "ceil" | "round" | "int"
	| "uc" | "lc" | "length"
	| "typeof"
	| "new"

<postfix-expr> ::= <primary-expr> ( <postfix-suffix> )*

<postfix-suffix> ::= <call-suffix>
	| <member-call-suffix>
	| <index-suffix>
	| <slice-suffix>
	| <dict-access-suffix>
	| <incdec-suffix>

<call-suffix> ::= "(" <arg-list>? ")"

<member-call-suffix> ::= "." <identifier> ( "(" <arg-list>? ")" )?
	| ".(" <expression> ")" "(" <arg-list>? ")"

<index-suffix> ::= "[" <expression> "]"

<slice-suffix> ::= "[" <expression>? ":" <expression>? "]"

<dict-access-suffix> ::= "{" <dict-key-expr> "}"

<incdec-suffix> ::= "++" | "--"

<arg-list> ::= <named-arg>
	| <expression>
	| <arg-list> "," <named-arg>
	| <arg-list> "," <expression>

<named-arg> ::= <identifier> ":" <expression>
	| <string-literal> ":" <expression>
	| <template-literal> ":" <expression>
	| "(" <expression> ")" ":" <expression>

<primary-expr> ::= <literal>
	| <identifier>
	| <super-call-expr>
	| "(" <expression> ")"
	| <array-literal>
	| <dict-literal>
	| <pairlist-literal>
	| <set-literal>
	| <bag-literal>
	| <floor-bracket-expr>
	| <ceil-bracket-expr>
	| <lambda-expr>

<type-expr> ::= <identifier>

<path-first-expr> ::= <postfix-expr> "@" <expression>

<path-exists-expr> ::= <postfix-expr> "@?" <expression>

<path-all-expr> ::= <postfix-expr> "@@" <expression>

<dict-key-expr> ::= <identifier>
	| <string-literal>
	| <template-literal>
	| "(" <expression> ")"
```

## 7. Literals

```
<literal> ::= <null-literal>
	| <bool-literal>
	| <number-literal>
	| <string-literal>
	| <binary-string-literal>
	| <regexp-literal>
	| <template-literal>
	| <empty-set-literal>

<null-literal> ::= "null"

<bool-literal> ::= "true" | "false" | "⊤" | "⊥"

<number-literal> ::= <integer-literal> | <float-literal>

<string-literal> ::= <dq-string> | <triple-dq-string>
<binary-string-literal> ::= <sq-binary-string>
	| <triple-sq-binary-string>
<regexp-literal> ::= "/" <regexp-char>* "/" <regexp-flag>?
<regexp-flag> ::= "i"

<template-literal> ::= <bt-string> | <triple-bt-string>

<array-literal> ::= "[" <array-items>? "]"

<array-items> ::= ( <array-item> | "," )+
	; duplicate and trailing commas are allowed

<array-item> ::= <expression>
	| <expression> "..." <expression>
	; inclusive integer range, ascending or descending

<dict-literal> ::= "{" <dict-pairs>? "}"

<dict-pairs> ::= <dict-pair> ( "," <dict-pair> )* ","?

<dict-pair> ::= <dict-key-expr> ":" <expression>

<pairlist-literal> ::= "{{" <pairlist-pairs>? "}}"

<pairlist-pairs> ::= <dict-pair> ( "," <dict-pair> )* ","?
	; duplicate keys and trailing commas are allowed

<set-literal> ::= "<<" <set-items>? ">>"
	| "«" <set-items>? "»"

<set-items> ::= <collection-item> ( "," <collection-item> )* ","?

<bag-literal> ::= "<<<" <bag-items>? ">>>"

<bag-items> ::= <collection-item> ( "," <collection-item> )* ","?

<collection-item> ::= <expression>
	| <expression> "..." <expression>
	; inclusive integer range, ascending or descending

<empty-set-literal> ::= "∅"

<floor-bracket-expr> ::= "⌊" <expression> "⌋"

<ceil-bracket-expr> ::= "⌈" <expression> "⌉"
```

## 8. Operator inventory (all spellings)

### 8.1 Arithmetic and numeric comparison

```
"+"  "-"  "*"  "×"  "/"  "÷"  "**"  "mod"
"="  "!="  "≠"  "<"  ">"  "<="  "≤"  ">="  "≥"
"<=>"  "≶"  "≷"
```

### 8.2 Type-aware equality

```
"=="  "≡"  "!="  "≢"
```

### 8.3 String operations

```
"_"  ; concatenation
"~"  ; regexp match
"eq" "ne" "gt" "ge" "lt" "le" "cmp"
"eqi" "nei" "gti" "gei" "lti" "lei" "cmpi"
```

### 8.4 Bitwise operations

```
"&" "|" "^"
"~" ; unary bytewise/numeric invert
```

### 8.5 Boolean operations

```
"and" "or" "xor" "nand"
"⋀" "⋁" "⊻" "⊼"
"!" "not" "¬"
```

### 8.6 Set / collection operations

```
"in" "∈" "∉"
"union" "⋃"
"intersection" "⋂"
"subsetof" "⊂"
"supersetof" "⊃"
"equivalentof" "⊂⊃"
```
### 8.7 Mutation operators

Binary:

```
":=" "~=" "+=" "-=" "*=" "×=" "/=" "÷=" "**=" "_=" "?:="
```

Unary:

```
"++" "--"
```

### 8.8 Other expression-level operators and delimiters

```
"\\" "∖"
"instanceof" "does" "can"
"?" ":" "?:"
"@" "@?" "@@"
"." ".(" ")"
"..."
"[" "]" "{" "}"
"(" ")" "," ";"
"<<" ">>" "«" "»"
"⌊" "⌋" "⌈" "⌉"
"->" "→"
```

## 9. Reserved keywords

The following are reserved and cannot be used as identifiers:

```
"let" "const" "function" "method" "static" "class" "trait"
"async" "await" "spawn"
"extends" "with" "but"
"if" "else" "unless" "while" "for" "in" "return" "next" "continue" "last"
"new" "self" "super" "fn"
"null" "true" "false"
"and" "or" "xor" "nand" "not"
"mod" "abs" "sqrt" "floor" "ceil" "round" "int" "length" "uc" "lc"
"typeof" "instanceof" "does" "can"
"union" "intersection" "subsetof" "supersetof" "equivalentof"
"eq" "ne" "gt" "ge" "lt" "le" "cmp"
"eqi" "nei" "gti" "gei" "lti" "lei" "cmpi"
"from" "import" "as"
"try" "catch" "throw" "die" "do"
"warn" "say" "print" "debug" "assert"
"__argc__" "__file__" "__global__" "__system__"
```

## 10. Lexical rules (summary)

```
<identifier> ::= <identifier-start> <identifier-char>*
	; Unicode-aware identifiers, including leading underscore names.
	; A single "_" is excluded because it is the concatenation operator.

<identifier-start> ::= <xid-start>
	| "_" <identifier-char>

<identifier-char> ::= <xid-continue>
	| "_"

<operator-token> ::= any single operator token accepted by lexer,
	an operator keyword (for example "eq" or "nei"), or an identifier
	used as an operator name.

<concat-operator> ::= "_"
	; Must not be immediately followed by <identifier-char>.

<comment> ::= "//" <until-eol>
	| "/*" <any-char>* "*/"

<integer-literal> ::= <digit>+
<float-literal> ::= <digit>+ "." <digit>+
<hex-digit> ::= <digit> | "a".."f" | "A".."F"

<dq-string> ::= '"' <dq-char>* '"'
<triple-dq-string> ::= '"""' <any-char>* '"""'
<sq-binary-string> ::= "'" <sq-binary-char>* "'"
<triple-sq-binary-string> ::= "'''" <any-char>* "'''"
<sq-binary-char> ::= <binary-plain-char> | <binary-escape>
<binary-plain-char> ::= any character except "'" and "\\"
<binary-escape> ::= "\\\\" | "\\'" | "\\n" | "\\r" | "\\t"
	| "\\x" <hex-digit> <hex-digit>
<bt-string> ::= '`' <template-char>* '`'
<triple-bt-string> ::= '```' <any-char>* '```'
```

Template strings allow interpolation:

```
<interpolation> ::= "${" <expression> "}"
```
