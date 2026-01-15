# SCAN-11
Yet another (11th, to be exact) Scheme Algebraic Notation

This document describes an algebraic surface syntax for Scheme code (NOT arbitrary S-expressions), inspired by the traditions of Haskell and ML. It maps directly to standard S-expressions but provides a cleaner, algebraic look and feel which some people prefer.

## 1. General Structure and Layout

The syntax relies on a Haskell-like **indentation-sensitive layout** rule (the "off-side rule") to minimize visual noise. While explicit braces `{ }`, semicolons `;`, and vertical bars `|` can be used to delimit blocks, statements, and clauses, they can be omitted if one pefers to use significant white space.

### Layout Triggers
Certain keywords trigger a layout context. If these keywords are **not** immediately followed by an opening brace `{`, the indentation of the next token establishes a column ("immediately" means nothing in between except white space and comments).
*   **Block Triggers** (`do`, `with`): Subsequent lines indented to this column are treated as statements separated by semicolons `;`.
*   **Alternative Triggers** (`of`, `cond`): Subsequent lines indented to this column are treated as clauses separated by vertical bars `|`.

Note: If an opening brace is not preceded by a trigger keyword, `do` is inserted automatically.

In layout mode:
1. The first token after the trigger keyword establishes the **reference column**
2. Lines starting at the reference column begin new entries (implicit separator)
3. Lines indented further continue the current entry
4. Lines indented less close the implicit block

This allows writing:

```
case x of
  RED -> 0
  GREEN -> 1
  BLUE -> 2
```

instead of:

```
case x of { RED -> 0 | GREEN -> 1 | BLUE -> 2 }
```

**Example:**

```haskell
-- Layout-based Syntax
fn x -> 
  if x < 0 then
    do
      log_error("Negative")
      -1
  else 
    x

-- Equivalent Explicit Syntax (note dropped do)
fn x -> 
  if x < 0 then {
    log_error("Negative");
    -1
  } else x
```

## 2. Syntax Overview

## Expressions

### Atoms and Operators

Atomic expressions include identifiers, literals (numbers, strings, booleans, characters), and infix/prefix operator expressions. Standard operators include arithmetic (`+`, `-`, `*`, `/`, `quo`, `rem`, `div`, `mod`), comparison (`<`, `<=`, `=`, `>=`, `>`), and logical (`and`, `or`, `not`).

### Parenthesized Expressions and Multiple Values

Parentheses have dual purpose—grouping for operator precedence and constructing multiple value expressions (unlike tuples, they are NOT first-class):

| Syntax | Meaning |
|--------|---------|
| `()` | Zero values (Scheme's `(values)`) |
| `(exp)` | Grouping / precedence override |
| `(e₁, e₂, …)` | Multiple values (Scheme's `(values e₁ e₂ …)`) |
| `(e₁, …, @ rest)` | Spliced values (Scheme's `(apply values e₁ … rest)`) |
| `(@ exp)` | Spread list as values (Scheme's `(apply values rest)`) |

### Function Application

Function application is written by juxtaposition, with the function followed by its arguments in parentheses, 
using the same syntax as shown above. Parentheses around arguments are required, even when there is just one argument:

```
f(x)
f(x, y, z)
map(double, xs)
```

When `@` appears in the argument list, the call becomes an `apply`:

```
f(a, b, @rest)   -- becomes (apply f a b rest)
```

Note that in Scheme many macro uses have the same form as function calls. They will parse correctly, as long as the `@`
notation is not used (there is no such thing as `apply` for a syntactic keyword).


### Lambda Expressions

Anonymous functions use the `fn` keyword:

```
fn x -> x + 1
fn (x, y) -> x * y
fn (head, @tail) -> tail
```

For case-lambda (multiple clauses with different arities), use `fn of`:

```
fn of {
  () -> 0
| (x) -> x
| (x, y) -> x + y
| (@many) -> error("too many arguments")
}
```

### Conditionals

**Two-armed if:**
```
if x < 0 then negate(x) else x
```

**Multi-way cond:**
```
cond {
  x < 0  -> "negative"
| x = 0  -> "zero"
| x > 0  -> "positive"
}
```

The final clause may omit its guard to serve as an `else`:
```
cond {
  x < 0 -> "negative"
| "non-negative"
}
```

Special clause forms using `.` provide access to the test result:

| Clause | Meaning |
|--------|---------|
| `exp -> .` | Return the test result itself if truthy |
| `exp -> . f` | Apply `f` to the test result (Scheme's `=>`) |

### Case Expressions

Pattern matching on datum values:

```
case color of {
  RED -> 0
| GREEN -> 1
| BLUE -> 2
}
```

Patterns can match multiple literals:
```
case c of {
  ('a' | 'e' | 'i' | 'o' | 'u') -> "vowel"
| -> . handle_consonant   -- else handle_consonant(c)
}
```

Note: the dot operator in infix position denotes a "reverse application", same as the `|>` operator in some 
functional languages. Here it is used in a similar role, but in a nullfix/prefix position.

### Do Blocks

The `do` keyword introduces a block containing definitions, commands, and a final expression:

```
do {
  val x = compute();
  print(x);
  x + 1
}
```

Depending on contents, this compiles to Scheme's `let`-body or `begin`.

### Let Expressions

Bindings scoped to a single expression use `in`:

```
let val x = 1, y = 2 in x + y

let double(x) = x * 2 in double(21)

letrec val even(n) = n = 0 or odd(n - 1),
           odd(n) = n > 0 and even(n - 1)
in even(10)
```

Note that here and in other `let` expressions the `var` keyword may be dropped; it is the default
type of binding.

**Multiple-value bindings** use `val` with tuple patterns:
```
let val (q, r) = divmod(17, 5) in q
```

**Named let** for iteration:
```
let loop(i = 0, acc = 1) in
  if i = n then acc
  else loop(i + 1, acc * i)
```

**Parameterize:**
```
parameterize current_output = file in
  print("hello")
```

**Local syntax:**
```
let syntax when = rules () of { (test, body) -> if test then body else () }
in when(ready(), go())
```

### Iteration with For

The `for` loop parsews as Scheme's `do`:

```
for i = 0 then i + 1,
    acc = 1 then acc * i
until i = n -> acc do {
  write(i); newline();
}
```

The `until` clause specifies termination conditions as `test -> result` clause.

### Exception Handling

The `guard` form handles exceptions:

```
guard exn of {
  is_file_error(exn) -> default_value
| -> . reraise
} do {
  read_file(path)
}
```

### List constructors

| Syntax | Meaning |
|--------|---------|
| `[]` | Empty list `'()` |
| `[a]` | `(list a)` |
| `[a, b, c]` | `(list a b c)` |
| `[a, b, @ rest]` | `(list* a b rest)` (improper/spliced) |
| `[@ xs]` | `(list-copy xs)` |

### Vector constructors

| Syntax | Meaning |
|--------|---------|
| `#[]` | Empty vector `#()` |
| `#[a]` | `(vector a)` |
| `#[a, b, c]` | `(vector a b c)` |
| `#[a, b, @ rest]` | `(apply vector a b rest)` |

### Peculiars

The `...` (ellipsis) and `_` (wildcard) notations are parsed as special expressions
to be used as meta-notation in syntax rules.


## Definitions

Definitions can be used on the top level of a program or a library, as well as at the beginning of the `do` block.

### Value Definitions

These are parsed as Scheme `define`:

```
val x = 42
val (a, b) = two_values()
val double(x) = x * 2
val fact(n) = if n = 0 then 1 else n * fact(n - 1)
```

The second form defines multiple values; the last two are shorthand for `fn`, i.e. lambda:

```
val double = fn x -> x * 2
```

### Record Type definitions

```
record Point = ...   -- define-record-type (details TBD)
```

### Syntax Definitions

```
syntax pipe_if = rules () of {
  pipe_if(test, f, alt) -> cond { test -> . f | alt }
| pipe_if(test, f) -> cond { test -> . f }
}
```

### Includes

```
include "helpers.scm"
include_ci "legacy.scm"
```

### Imports

Imports are written in a postfix, readable style similar to Haskell or Elm.
Library names use dot notation without spaces: `scheme.base`, `srfi.1`.

```haskell
import scheme.base                      -- (import (scheme base))
import scheme.list exposing (map, fold) -- (import (only (scheme list) map fold))
import scheme.list (map, fold)          -- same as above; exposing keyword is optional
import my.lib hiding (internal)         -- (import (except (my lib) internal))
import other.lib renaming (foo as bar)  -- (import (rename (other lib) (foo bar)))
import geometry qualifying (geo)        -- (import (prefix (geometry) geo-))
```

The postfix notation can be used recursively:
```haskell
import scheme.list (map, fold) renaming (fold as foldl)
```

### Library definitions (TBD)

```
library mylib.utils with {
  import scheme.base;
  export helper, main;
  
  val helper(x) = x + 1;
  val main() = helper(0)
}
```

## Lexical Syntax Overview

This section describes the low-level lexical structure of the surface syntax, including comments, literals, identifiers, and special tokens.

### Comments

**Line comments** begin with a double hyphen and extend to the end of the line:

```
val x = 42  -- this is a comment
```

**Block comments** are enclosed in `{-` and `-}` and may be nested:

```
{-
  This is a block comment.
  {- They can be nested. -}
  Useful for commenting out code.
-}
```

### Whitespace

Whitespace (spaces, tabs, newlines) is **optional** between tokens when the boundary is unambiguous:

```
x+2*3          -- same as: x + 2 * 3
f(x,y)         -- same as: f ( x , y )
[1,2,3]        -- same as: [ 1 , 2 , 3 ]
```

Within layout blocks, indentation determines structure. 
Only spaces can be used for significant identation.

### Identifiers

Identifiers name variables, functions, and syntactic keywords.

**Regular identifiers** begin with a lowercase letter and may contain letters, digits, and underscores:

```
x
foo
string_to_list
is_null
set_box
char_ready
really_long_name
```

#### Name Translation

Identifiers are automatically translated to Scheme symbols according to these rules, applied in order:

| Pattern | Scheme Form | Example |
|---------|-------------|---------|
| `is_xxx` | `xxx?` | `is_null` → `null?` |
| `xxx_to_yyy` | `xxx->yyy` | `string_to_list` → `string->list` |
| `xxx_with_yyy` | `xxx/yyy` | `call_with_cc` → `call/cc` |
| `xxx_lt` | `xxx<` | `char_lt` → `char<` |
| `xxx_le` | `xxx<=` | `char_le` → `char<=` |
| `xxx_eq` | `xxx=` | `char_eq` → `char=` |
| `xxx_ge` | `xxx>=` | `char_ge` → `char>=` |
| `xxx_gt` | `xxx>` | `char_gt` → `char>` |
| `xxx_add` | `xxx+` | `fx_add` → `fx+` |
| `xxx_sub` | `xxx-` | `fx_sub` → `fx-` |
| `xxx_mul` | `xxx*` | `fx_mul` → `fx*` |
| `xxx_div` | `xxx/` | `fx_div` → `fx/` |
| `xxx_set` | `xxx-set!` | `vector_set` → `vector-set!` |
| `set_xxx` | `set-xxx!` | `set_box` → `set-box!` |
| `xxx_yyy` | `xxx-yyy` | `vector_map` → `vector-map` |

The underscore-to-hyphen rule applies generally after the special patterns are checked.

### Symbolic Constants

**All-uppercase identifiers** are treated as symbolic constants and translate to quoted symbols (lowercase):

```
FOO            -- (quote foo)
RED            -- (quote red)
HTTP_NOT_FOUND -- (quote http-not-found)
```

This provides convenient notation for enumeration values and message symbols.
Note: the quote wrapper is removed when symbolic constansta are used in `case` patterns.

### Reserved Words

The following words have special meaning and cannot be used as ordinary identifiers:

```
fn  of  if  then  else  case  cond  do  let  letrec  in
for  until  guard  val  record  syntax  rules  include
include_ci  import  library  parameterize  and  or  not
true false
```

**Contextual keywords** have special meaning only in specific positions and may be used as identifiers elsewhere:

```
exposing  hiding  renaming  qualifying  as with
```

### Escape to Scheme Reader

A **backslash** switches to the standard Scheme reader for one complete S-expression:

```
\'foo              -- (quote foo)
\(a b c)           -- the list (a b c)
\#(1 2 3)          -- the vector #(1 2 3)
\#u16(1 2 3)       -- implementation-specific reader extensions
```

This provides an escape hatch for any Scheme datum, including reader extensions not otherwise supported by the surface syntax.
The returned S-expressions are embedded verbatim, so they should be quoted if they are used as data.

TODO: Extend the Scheme reader used after backslash to allow algebraic expressions in braces, to support algebraic
syntax in quasiquotations.

### Booleans

Boolean literals may use two notations:

| Literal | Meaning |
|---------|---------|
| `#t`, `true` | True |
| `#f`, `false` | False |


## Numbers

Numbers follow R7RS Scheme conventions.

### Exactness Prefixes

| Prefix | Meaning |
|--------|---------|
| `#e` | Exact (rational) |
| `#i` | Inexact (floating-point) |

### Radix Prefixes

| Prefix | Base | Digits |
|--------|------|--------|
| `#b` | Binary | `0`, `1` |
| `#o` | Octal | `0`–`7` |
| `#d` | Decimal (default) | `0`–`9` |
| `#x` | Hexadecimal | `0`–`9`, `a`–`f`, `A`–`F` |

Prefixes may appear in either order: `#x#e` or `#e#x`.

### Integers

A prefix minus or plus sign is considered part of the numeric literal:

```
42
-17
+5
#b101010       -- binary: 42
#o52           -- octal: 42
#x2A           -- hexadecimal: 42
#e#x2A         -- exact hexadecimal
-#xFF          -- -255
```

### Rationals

Exact fractions use a slash between integers:

```
1/2
-3/4
#x10/F         -- hexadecimal: 16/15
```

### Reals

Decimal notation with optional exponent:

```
3.14159
-2.5
6.02e23
1.38e-23
#i42           -- inexact integer: 42.0
```

**Special float values:**

```
+inf.0         -- positive infinity
-inf.0         -- negative infinity
+nan.0         -- not a number
-nan.0         -- also not a number
```

### Complex Numbers

Rectangular form uses an infix `+` or `-` followed by an imaginary part with `i` suffix. In this context, the infix operator is part of the numeric literal, not a separate token:

```
3+4i           -- complex: 3 + 4i
-2.5-1.5i      -- complex: -2.5 - 1.5i
0+1i           -- pure imaginary
3.0+0.0i       -- inexact complex with zero imaginary part
```

Polar form with `@` separator:

```
1.0@1.5708     -- magnitude @ angle (radians)
```

**Note:** If an explicit hash-letter numerical prefix is used, Scheme parser is used to read the number
(and normal Scheme termitator is expected). If there are no explicit hash-letter numerical prefix,
the algebraic parser first parses a numerical expression as an operator expression and post-processes it,
looking for prefix `+` and `-` operators, followed by a (signless) numerical constant, as well as for infix `+`, `=`,
`/`, and `@` operators with numerical constants or constant expressions as operands, reassembling them into Scheme 
numerical constants if they fit the corresponding pattern. The only 'bare' numerical form specially recognized by the lexer
is a number immediately followed by the `i` suffix, producing an "imaginary number" token. 
```
x+2            -- addition: x plus 2
x+2i           -- addition: x plus the imaginary number 2i
3+4i           -- single complex literal: 3+4i
2i             -- imaginary number 2i
-1i            -- imaginary number -i
-i             -- expression (- i)
+i             -- expression (+ i)
i              -- identifier i
```

## Characters and strings

Character literals are enclosed in **single quotes** and contain exactly one character (possibly escaped):

```
'a'            -- lowercase a
'Z'            -- uppercase Z
' '            -- space
'λ'            -- Greek lambda
```

Strings are enclosed in double quotes and use standard Scheme syntax:

```
"hello, world"
"line one\nline two"
""                      -- empty string
```

Characters and strings share Scheme string escape notations:

| Literal | Character |
|---------|-----------|
| `'\n'` | Newline |
| `'\t'` | Horizontal tab |
| `'\r'` | Carriage return |
| `'\\'` | Backslash |
| `'\''` | Single quote |
| `'\"'` | Double quote |
| `'\a'` | Alarm (bell) |
| `'\b'` | Backspace |
| `'\|'` | Vertical bar |
| `'\x1B;'` | Hex escape (ESC) |
| `'\x03BB;'` | Hex escape (λ) |

Hex escapes begin with `\x`, followed by one or more hexadecimal digits, and terminated by a semicolon.

A backslash before a newline allows strings to span lines; leading whitespace on the continuation line 
is consumed up to an optional second backslash:

```
"this is a \
   long string"          -- "this is a    long string"

"this is a \
   \long string"         -- "this is a long string"
```

### Delimiters

| Token | Purpose |
|-------|---------|
| `(` `)` | Grouping, argument/formals lists, value expressions |
| `[` `]` | List literals |
| `#[` `]` | Vector literals |
| `{` `}` | Explicit blocks |
| `,` | Separator in lists, arguments, bindings |
| `;` | Statement/definition separator |
| `\|` | Clause separator (in `of` and `cond`-like blocks) |
| `\` | Escape to Scheme reader |
| `=` | Binding separator, numerical equality operator |
| `->` | Function arrow, clause results |
| `:=` | Assignment command (`set!`) |
| `@` | Splice/rest marker, polar notation marker, append operator |
| `.` | Library name separator; special clause marker, reverse application operator |


### Operators (TBD)

Arithmetic:
```
+   -   *   /   quo   rem   div   mod
```

Comparison:
```
<   >   <=   >=   =   ==   eqv   eq
```

Logical:
```
and   or   not
```

Operators have conventional precedence, with parentheses available for explicit grouping. 
Since whitespace is optional, expressions like `a+b*c` parse correctly according to precedence rules.


# Formal Grammar

This section presents the complete formal grammar in extended BNF notation.

## Notation Conventions

| Symbol | Meaning |
|--------|---------|
| ⟨*name*⟩ | Nonterminal |
| `terminal` | Terminal (keyword or punctuation) |
| → | Production rule |
| *X*? | Zero or one *X* |
| *X*\* | Zero or more *X* |
| *X*⁺ | One or more *X* |
| *X*<sup>,\*</sup> | Zero or more *X*, separated by `,` |
| *X*<sup>,+</sup> | One or more *X*, separated by `,` |
| *X*<sup>\|\*</sup> | Zero or more *X*, separated by `\|` |
| *X*<sup>\|+</sup> | One or more *X*, separated by `\|` |



### Notation Key

*   **Non-terminals** are enclosed in angle brackets: $\langle name \rangle$
*   **Terminals** are **bold** (keywords) or `code` (punctuation).
*   **Production** is denoted by &rarr;
*   **Repetition** is denoted by superscripts:
    *   $^*$ : Zero or more
    *   $^+$ : One or more
    *   $^?$ : Zero or one (optional)
    *   $^{,*}$ : Zero or more, separated by commas
    *   $^{,+}$ : One or more, separated by commas
    *   $^{|*}$ : Zero or more, separated by vertical bars

---

### Program Structure

| Production Rule | Description |
| :--- | :--- |
| $\langle program \rangle$ &rarr; $\langle def\_or\_cmd \rangle^*$ | A program is a sequence of definitions or commands |
| $\langle def\_or\_cmd \rangle$ &rarr; | |
| &nbsp;&nbsp; $\langle def \rangle$ | |
| &nbsp;&nbsp; \| $\langle cmd \rangle$ | Except for bodyless `let` forms |

### Atomic and Operator Expressions

| Production Rule | Description |
| :--- | :--- |
| $\langle aexp \rangle$ &rarr; | **Atomic Expressions** |
| &nbsp;&nbsp; $\langle lit \rangle$ | Literals (numbers, strings, etc.) |
| &nbsp;&nbsp; \| **\_** | `syntax-rules` wildcard |
| &nbsp;&nbsp; \| **...** | `syntax-rules` ellipsis |
| $\langle oexp \rangle$ &rarr; | **Operator Expressions** |
| &nbsp;&nbsp; $\langle exp \rangle$ **@** $\langle exp \rangle$ | Priority 1, `append`, right-associative |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **:** $\langle exp \rangle$ | Priority 1, `cons`, right-associative |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **or** $\langle exp \rangle$ | Priority 2 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **and** $\langle exp \rangle$ | Priority 3 |
| &nbsp;&nbsp; \| **not** $\langle exp \rangle$ | Priority 4 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **==** $\langle exp \rangle$ | Priority 5, `equal?` |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **=** $\langle exp \rangle$ | Priority 5, numerical `=` |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **>** $\langle exp \rangle$ | Priority 5 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **>=** $\langle exp \rangle$ | Priority 5 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **<** $\langle exp \rangle$ | Priority 5 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **<=** $\langle exp \rangle$ | Priority 5 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **+** $\langle exp \rangle$ | Priority 6 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **-** $\langle exp \rangle$ | Priority 6 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **\*** $\langle exp \rangle$ | Priority 7 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **/** $\langle exp \rangle$ | Priority 7 |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **quo** $\langle exp \rangle$ | Priority 7, `quotient` (truncate) |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **rem** $\langle exp \rangle$ | Priority 7, `remainder` (truncate) |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **div** $\langle exp \rangle$ | Priority 7, `floor-quotient` |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **mod** $\langle exp \rangle$ | Priority 7, `modulo` (floor) |

### General Expressions

| Production Rule | Description |
| :--- | :--- |
| $\langle exp \rangle$ &rarr; | |
| &nbsp;&nbsp; $\langle aexp \rangle$ | Atomic expressions |
| &nbsp;&nbsp; \| $\langle oexp \rangle$ | Operator expressions |
| &nbsp;&nbsp; \| $\langle vals \rangle$ | Priority override or `values` form |
| &nbsp;&nbsp; \| $\langle exp \rangle$ $\langle vals \rangle$ | Application (or `apply` if `@` is in vals) |
| &nbsp;&nbsp; \| $\langle list \rangle$ | List constructor |
| &nbsp;&nbsp; \| $\langle vector \rangle$ | Vector constructor |
| &nbsp;&nbsp; \| **fn** $\langle formals \rangle$ **->** $\langle exp \rangle$ | Lambda expression |
| &nbsp;&nbsp; \| **fn** **of** **{** $\langle formals\_exp \rangle^{|\*} $ **}** | Case-lambda |
| &nbsp;&nbsp; \| **if** $\langle exp \rangle$ **then** $\langle exp \rangle$ **else** $\langle exp \rangle$ | Two-arm conditional |
| &nbsp;&nbsp; \| **case** $\langle exp \rangle$ **of** **{** $\langle pclause \rangle^{|\*} $ **}** | Pattern match `case` |
| &nbsp;&nbsp; \| **cond** **{** $\langle clause \rangle^{|\*} $ **}** | Conditional `cond` |
| &nbsp;&nbsp; \| **do** **{** $\langle def \rangle^*$ $\langle cmd \rangle^*$ $\langle exp \rangle$ **}** | Body or `begin` |
| &nbsp;&nbsp; \| $\langle let \rangle$ **in** $\langle exp \rangle$ | Let-like binding forms |
| &nbsp;&nbsp; \| **let** $\langle id \rangle$ **(** $\langle bnd \rangle^{,*} $ **)** **in** $\langle exp \rangle$ | Named `let` (single-value binds only) |
| &nbsp;&nbsp; \| **for** $\langle sbnd \rangle^{,*} $ **until** $\langle clause \rangle$ **do** **{** $\langle def \rangle^*$ $\langle cmd \rangle^*$ **}** | Scheme `do` iteration form |
| &nbsp;&nbsp; \| **guard** $\langle id \rangle$ **of** **{** $\langle clause \rangle^{|\*} $ **}** **do** **{** $\langle def \rangle^*$ $\langle cmd \rangle^*$ **}** | Exception handling |

### Clauses and Patterns

| Production Rule | Description |
| :--- | :--- |
| $\langle formals\_exp \rangle$ &rarr; $\langle formals \rangle$ **->** $\langle exp \rangle$ | |
| $\langle clause \rangle$ &rarr; | **Conditional Clauses** |
| &nbsp;&nbsp; $\langle exp \rangle$ **->** $\langle exp \rangle$ | Standard clause |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **->** **.** | Return result if true |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **->** **.** $\langle exp \rangle$ | Scheme's `=>` clause (apply to result) |
| &nbsp;&nbsp; \| **->** **.** $\langle exp \rangle$ | `else =>` clause (last, `case` only) |
| &nbsp;&nbsp; \| $\langle exp \rangle$ | `else` clause (last, `cond`/`case` only) |
| $\langle pclause \rangle$ &rarr; | **Pattern Clauses** |
| &nbsp;&nbsp; $\langle pat \rangle$ **->** $\langle exp \rangle$ | Standard pattern match |
| &nbsp;&nbsp; \| $\langle pat \rangle$ **->** **.** $\langle exp \rangle$ | Scheme's `=>` clause |
| &nbsp;&nbsp; \| **->** **.** $\langle exp \rangle$ | `else =>` clause (last, `case` only) |
| $\langle pat \rangle$ &rarr; | |
| &nbsp;&nbsp; $\langle lit \rangle$ | Shortcut for `( <lit> )` |
| &nbsp;&nbsp; \| **(** $\langle lit \rangle^{|\*} $ **)** | Alternatives (literals only) |

### Bindings and Let Forms

| Production Rule | Description |
| :--- | :--- |
| $\langle sbnd \rangle$ &rarr; | **Stepped Bindings (for loops)** |
| &nbsp;&nbsp; $\langle bnd \rangle$ **then** $\langle exp \rangle$ | With step expression |
| &nbsp;&nbsp; \| $\langle bnd \rangle$ | Without step |
| $\langle let \rangle$ &rarr; | **Binding Keyword Blocks** |
| &nbsp;&nbsp; **let** **val**$^?$ $\langle bnd \rangle^{,*} $ | `let` or `let-values` |
| &nbsp;&nbsp; \| **letrec** **val**$^?$ $\langle bnd \rangle^{,*} $ | `letrec` |
| &nbsp;&nbsp; \| **let** **syntax** $\langle id\_rules \rangle^{,*} $ | `let-syntax` |
| &nbsp;&nbsp; \| **letrec** **syntax** $\langle id\_rules \rangle^{,*} $ | `letrec-syntax` |
| &nbsp;&nbsp; \| **parameterize** $\langle bnd \rangle^{,*} $ | `parameterize` |
| $\langle bnd \rangle$ &rarr; | |
| &nbsp;&nbsp; $\langle formals \rangle$ **=** $\langle exp \rangle$ | Single/multi-value pattern |
| &nbsp;&nbsp; \| $\langle id \rangle$ $\langle formals \rangle$ **=** $\langle exp \rangle$ | Function shortcut syntax |

### Macros and Rules

| Production Rule | Description |
| :--- | :--- |
| $\langle id\_rules \rangle$ &rarr; $\langle id \rangle$ **=** $\langle rules \rangle$ | |
| $\langle rules \rangle$ &rarr; **rules** **(** $\langle id \rangle^{,*} $ **)** **of** **{** $\langle exp\_to\_exp \rangle^{|\*} $ **}** | `syntax-rules` definition |
| $\langle exp\_to\_exp \rangle$ &rarr; $\langle exp \rangle$ **->** $\langle exp \rangle$ | Pattern transformation |

### Definitions

| Production Rule | Description |
| :--- | :--- |
| $\langle def \rangle$ &rarr; | |
| &nbsp;&nbsp; **do** **{** $\langle def \rangle^*$ **}** **;** | Splicing `begin` (in def context) |
| &nbsp;&nbsp; \| **val** $\langle bnd \rangle$ **;** | `define` or `define-values` |
| &nbsp;&nbsp; \| **record** $\langle id \rangle$ **=** $\langle rdef \rangle$ **;** | `define-record-type` (rdef TBD) |
| &nbsp;&nbsp; \| **syntax** $\langle id \rangle$ **=** $\langle rules \rangle$ **;** | `define-syntax` |
| &nbsp;&nbsp; \| **include** $\langle str \rangle$ **;** | |
| &nbsp;&nbsp; \| **include\_ci** $\langle str \rangle$ **;** | |
| &nbsp;&nbsp; \| **import** $\langle iset \rangle$ **;** | |
| &nbsp;&nbsp; \| **library** $\langle lname \rangle$ **with** **{** $\langle ldef \rangle^*$ **}** **;** | `define-library` |

### Imports and Names

| Production Rule | Description |
| :--- | :--- |
| $\langle iset \rangle$ &rarr; | **Import Sets (Postfix)** |
| &nbsp;&nbsp; $\langle lname \rangle$ | Library name |
| &nbsp;&nbsp; \| $\langle iset \rangle$ **exposing**$^?$ **(** $\langle id \rangle^{,*} $ **)** | `only` (exposing kw optional) |
| &nbsp;&nbsp; \| $\langle iset \rangle$ **hiding** **(** $\langle id \rangle^{,*} $ **)** | `except` |
| &nbsp;&nbsp; \| $\langle iset \rangle$ **renaming** **(** $\langle id\_as\_id \rangle^{,*} $ **)** | `rename` |
| &nbsp;&nbsp; \| $\langle iset \rangle$ **qualifying** **(** $\langle prefix \rangle$ **)** | `prefix` |
| $\langle id\_as\_id \rangle$ &rarr; $\langle id \rangle$ **as** $\langle id \rangle$ | |
| $\langle lname \rangle$ &rarr; | |
| &nbsp;&nbsp; $\langle lseg \rangle$ | |
| &nbsp;&nbsp; \| $\langle lname \rangle$ **.** $\langle lseg \rangle$ | Dot notation (no spaces) |
| $\langle lseg \rangle$ &rarr; $\langle id \rangle$ \| $\langle int \rangle$ | Name segment |

### Commands (Statements)

| Production Rule | Description |
| :--- | :--- |
| $\langle cmd \rangle$ &rarr; | |
| &nbsp;&nbsp; $\langle let \rangle$ **;** | Let-like forms (block scoped) |
| &nbsp;&nbsp; \| $\langle id \rangle$ **:=** $\langle exp \rangle$ **;** | Assignment `set!` |
| &nbsp;&nbsp; \| **if** $\langle exp \rangle$ **then** $\langle exp \rangle$ **;** | Single-arm `if` |
| &nbsp;&nbsp; \| $\langle exp \rangle$ **;** | Expression for side effects |

### Data Constructors and Argument Lists

| Production Rule | Description |
| :--- | :--- |
| $\langle vals \rangle$ &rarr; | **Values / Parentheses** |
| &nbsp;&nbsp; **( )** | `(values)` |
| &nbsp;&nbsp; \| **(** $\langle exp \rangle$ **)** | Grouping (priority override) |
| &nbsp;&nbsp; \| **(** $\langle exp \rangle^{,+} $ **)** | `(values exp ...)` |
| &nbsp;&nbsp; \| **(** $\langle exp \rangle^{,*} $ **@** $\langle exp \rangle$ **)** | `(apply values ...)` |
| $\langle formals \rangle$ &rarr; | **Formal Arguments** |
| &nbsp;&nbsp; **( )** | No arguments |
| &nbsp;&nbsp; \| $\langle id \rangle$ | Single argument |
| &nbsp;&nbsp; \| **(** $\langle id \rangle$ **)** | Single argument (explicit) |
| &nbsp;&nbsp; \| **(** $\langle id \rangle^{,*} $ **)** | Multiple arguments |
| &nbsp;&nbsp; \| **(** $\langle id \rangle^{,*} $ **@** $\langle id \rangle$ **)** | Arguments with rest param |
| $\langle list \rangle$ &rarr; | **List Construction** |
| &nbsp;&nbsp; **[ ]** | Empty list `'()` |
| &nbsp;&nbsp; \| **[** $\langle exp \rangle^{,+} $ **]** | `(list ...)` |
| &nbsp;&nbsp; \| **[** $\langle exp \rangle^{,*} $ **@** $\langle exp \rangle$ **]** | `(list* ...)` / `(apply list ...)` |
| $\langle vector \rangle$ &rarr; | **Vector Construction** |
| &nbsp;&nbsp; **#[ ]** | Empty vector `'#()` |
| &nbsp;&nbsp; \| **#[** $\langle exp \rangle^{,+} $ **]** | `(vector ...)` |
| &nbsp;&nbsp; \| **#[** $\langle exp \rangle^{,*} $ **@** $\langle exp \rangle$ **]** | `(apply vector ...)` |


---

