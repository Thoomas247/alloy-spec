# Alloy Language — Formal Language Specification

**Status:** draft · **Revision:** 2026-08-23 · **Source of truth for:** the `alloyc` compiler implementations that consume this repository as a submodule.

Code samples and parenthetical asides are explanatory, not normative — where an example and a rule disagree, the rule governs.

## Table of Contents

1. [A Complete Program](#1-a-complete-program)
2. [Lexical Grammar](#2-lexical-grammar)
3. [Syntactic Grammar](#3-syntactic-grammar)
4. [Type System & Static Semantics](#4-type-system--static-semantics)
5. [Execution Semantics](#5-execution-semantics)
6. [Standard Library & Primitives](#6-standard-library--primitives)
7. [Compile-Time Evaluation & Macros](#7-compile-time-evaluation--macros)

---

## 1. A Complete Program

One program exercising the pieces the rest of this document specifies in
isolation: imports (§6.4), an interface and a conforming type (§6.2), an
extension function (§5.5), a generic container from the standard library
(§6.1a), a `for` loop over a custom iterable (§5.3), and explicit borrowing of
reference-typed results (§5.2).

```alloy
import std::io;
import std::string;
import std::vector;

interface Named {
    fn name(self: &) -> &[u8];
}

type Point : Named = struct {
    x: i32,
    y: i32,
};

// Extension function on Point; satisfies Named.
fn name(self p: &Point) -> &[u8] {
    return "point";
}

fn render(self p: &Point) -> String {
    var out = String::empty();
    out.append(&p.name());        // reference-typed result: borrowed explicitly
    out.append(" (");
    out.append_integer(p.x);
    out.append(", ");
    out.append_integer(p.y);
    out.append(")");
    return out;
}

fn main() -> i32 {
    var points = Vector::empty<Point>();
    points.push(Point { .x = 1, .y = 2 });
    points.push(Point { .x = 3, .y = 4 });

    for (points) |p: &| {
        var line = p.render();
        print_line(&line.view());
    }

    return 0;
}
```

Printed output:

```
point (1, 2)
point (3, 4)
```

Every `String` built in the loop owns its buffer and is reclaimed at the end of
the iteration that built it (§5.2); nothing here frees memory by hand.

---

## 2. Lexical Grammar

### 2.1 Character Set

Source files are encoded in **UTF-8**. Identifiers and literal strings fully support Unicode character mappings.

### 2.2 Whitespace & Comments

```
whitespace     ::= ' ' | '\t' | '\n' | '\r'
line_comment   ::= '//' <any char except '\n'>*
block_comment  ::= '/*' ( block_comment | <any sequence not containing '/*' or '*/'> )* '*/'
```

Whitespace and comments are not significant to the grammar and are ignored between tokens. Newlines are ordinary whitespace; statements and declarations are terminated by the `;` separator (§3.1). A line comment ends at the newline.

Block comments nest arbitrarily. The compiler tracks the nesting depth, and a block comment is only considered terminated when all opened comment blocks are closed. An unterminated block comment is a compile-time error.

### 2.3 Identifiers

```
identifier     ::= ( letter | '_' ) ( letter | digit | '_' )*
letter         ::= [a-zA-Z] | <any non-ASCII UTF-8 sequence>
digit          ::= [0-9]
```

Any identifier that matches a reserved keyword (§2.4) is treated as that keyword and may not be used as a user-defined name.

Any non-ASCII UTF-8 byte sequence is currently identifier-legal in both start and continuation positions; restricting identifiers to UAX #31 -like character categories is planned.

### 2.4 Keywords

Reserved — cannot be used as identifiers:

```
import   as       extern   type     enum     struct
const    var      fn       if       else     while
for      match    break    yield    return   new    move
self     pub      exp      true     false    macro  interface
is       to
```

### 2.5 Operators & Punctuation

Complete list of symbolic tokens:

| Category                | Symbols                                              |
| ----------------------- | ---------------------------------------------------- |
| Arithmetic              | `+` `-` `*` `/` `%`                                  |
| Compound assign         | `+=` `-=` `*=` `/=` `%=` `<<=` `>>=` `&=` `\|=` `^=` |
| Comparison              | `==` `!=` `<` `<=` `>` `>=`                          |
| Logical                 | `&&` `\|\|` `!`                                      |
| Bitwise                 | `&` `\|` `^` `~` `<<` `>>`                           |
| Assignment              | `=`                                                  |
| Arrow                   | `->`                                                 |
| Path separator          | `::`                                                 |
| Member access           | `.`                                                  |
| Range                   | `..`                                                 |
| Variadic                | `...`                                                |
| Type annotation         | `:`                                                  |
| Separator               | `,` `;`                                              |
| Delimiters              | `(` `)` `{` `}` `[` `]`                              |
| Comptime / Macro prefix | `#`                                                  |

When multiple symbolic tokens share a common prefix, the longest matching token is always chosen.

### 2.6 Literals

```
integer_literal  ::= decimal_literal | hex_literal | binary_literal | octal_literal
decimal_literal  ::= [0-9]+
hex_digit        ::= [0-9a-fA-F]
hex_literal      ::= '0x' hex_digit+
binary_literal   ::= '0b' [01]+
octal_literal    ::= '0o' [0-7]+

float_literal    ::= [0-9]+ '.' [0-9]+ [ exponent ] | [0-9]+ exponent
exponent         ::= ( 'e' | 'E' ) [ '+' | '-' ] [0-9]+
string_literal   ::= '"' ( escape_seq | <any char except '"' or '\n'> )* '"'
char_literal     ::= '\'' ( escape_seq | <any char except '\'' or '\n'> )+ '\''
escape_seq       ::= '\\' [nrt0\\'"] | '\\x' hex_digit{2} | '\\u{' hex_digit+ '}'
```

Integer literals support standard **decimal**, **hexadecimal** (`0x`), **binary** (`0b`), and **octal** (`0o`) radix representations.

A float literal requires **at least one digit after the dot**, so `1.` is not a literal and `1.foo()` is unambiguously member access on an integer literal. Scientific notation is written with `e` / `E` (`6.02e23`, `1e-9`, `2.5E+3`); the exponent digits are decimal and may carry a sign.

String and character literals are single-line: a raw newline inside a literal is a compile-time error (the literal is reported as unterminated). A line break in string data is written with the `\n` escape. An escape introducer outside the recognized set is a compile-time error.

#### Escape Sequences

The compiler recognizes standard escape sequences within string and character literals:

- `\n` (line feed), `\r` (carriage return), `\t` (tab), `\0` (null byte)
- `\\` (backslash), `\"` (double quote), `\'` (single quote)
- `\xHH` (arbitrary 1-byte hex value)
- `\u{HHHH}` (arbitrary multi-byte Unicode scalar value)

#### String Literal Typing

A string literal has type `&[u8]`: a slice viewing the literal's bytes in
static program memory, valid for the whole program run. A string literal never
allocates; an owned heap copy is made explicitly (`new "text"`, type `*[u8]`).

#### Character Literal Typing

The specific underlying primitive type of a character literal dynamically matches its size in bytes:

- Single quotes can bind a single character or a sequence of sequential characters up to 8 bytes long.
- A standard 1-byte ASCII character literal (e.g., `'a'`) evaluates to a `u8`.
- A multi-character packed constant sequence (e.g., `'abcdefgh'`) spans 8 bytes and evaluates to a `u64`.
- Variable-width Unicode characters evaluate to the smallest unsigned primitive integer width capable of holding their entire byte representation (typically `u32` for standard individual scalar sequences).

---

## 3. Syntactic Grammar

### 3.1 Full EBNF

**Notation.** `=` defines a rule and `;` ends it; `|` separates alternatives;
`{ x }` is zero or more repetitions of `x`; `[ x ]` marks `x` optional;
`"..."` is a literal token; `(* ... *)` is a comment; and `<...>` states in
prose a condition the grammar does not express. Lexical rules (§2) use `::=`
with the same conventions.

**Trailing commas.** Every comma-separated list below — parameters, call
arguments, type parameters and type arguments, struct and enum members, member
initializers, array elements, capture lists, and interface markers — accepts an
optional trailing comma before its closing delimiter. The productions omit it
for readability.

```ebnf
(* Top level *)
module          = { import_decl } { definition } EOF ;

import_decl     = "import" identifier { "::" identifier } [ "as" identifier ] terminator ;

definition      = [ "pub" | "exp" ] ( type_def | fn_def | extern_def | interface_def | macro_def ) ;
type_def        = "type" identifier [ "<" type_param { "," type_param } ">" ] [ ":" interface_marker { "," interface_marker } ] "=" type terminator ;
interface_def   = "interface" identifier [ "<" type_param { "," type_param } ">" ] "{" { interface_fn } "}" ;
interface_fn    = "fn" identifier "(" receiver { "," param } ")" [ "->" type ] terminator ;
receiver        = "self" ":" ( "&" | "&" "var" | "*" | "*" "var" ) ;
interface_marker = identifier [ "<" type { "," type } ">" ] ;
fn_def          = "fn" [ identifier "::" ] identifier [ "<" type_param { "," type_param } ">" ] function ;
macro_def       = "macro" identifier "(" [ macro_param { "," macro_param } ] ")"
                  ( stmt_block | terminator ) ;
macro_param     = identifier [ ":" type ] ;
extern_def      = "extern" identifier "(" extern_params ")" [ "->" type ] terminator ;
extern_params   = /* empty */
                | param { "," param } [ "," "..." ]
                | "..." ;

type_param      = identifier [ ":" interface_marker ] ;

(* Functions *)
function        = "(" [ param { "," param } ] ")" [ "->" type ] stmt_block ;
param           = [ "self" ] identifier ":" type ;

(* Types *)
type            = type_modifier ( base_type | type ) ;
type_modifier   = /* none */ | "*" | "*" "var" | "&" | "&" "var" ;

base_type       = named_type
                | struct_type
                | enum_type
                | array_type
                | fn_type
                | comptime_expr ;

named_type      = identifier { "::" identifier } [ "<" type { "," type } ">" ] ;
struct_type     = "struct" "{" [ struct_member { "," struct_member } ] "}" ;
struct_member   = identifier ":" type ;
enum_type       = "enum" "{" [ enum_member { "," enum_member } ] "}" ;
enum_member     = identifier [ ":" type ] ;
array_type      = "[" type [ ":" expression ] "]" ;  (* sizeless form only under "&" or "*" *)
fn_type         = "(" [ type { "," type } ] ")" [ "->" type ] ;

(* Statements *)
statement       = var_def
                | stmt_block
                | if_expr
                | for_expr
                | while_expr
                | match_expr
                | break_stmt
                | yield_stmt
                | return_stmt
                | expr_stmt
                | ";" ;                                                      (* empty statement *)

var_def         = ( "var" | "const" ) identifier [ ":" type ] "=" expression terminator ;
stmt_block      = "{" { statement } "}" ;
break_stmt      = "break" [ expression ] ( terminator | <block-like operand> ) ;
yield_stmt      = "yield" expression ( terminator | <block-like operand> ) ;
return_stmt     = "return" [ expression ] terminator ;
expr_stmt       = expression ( assign_op expression terminator | terminator ) ;

terminator      = ";" | <implicit before "}" / "else" / EOF> ;

assign_op       = "=" | "+=" | "-=" | "*=" | "/=" | "%="
                | "<<=" | ">>=" | "&=" | "|=" | "^=" ;

(* Expressions *)
expression      = cast_expr { binary_op cast_expr } ;  (* grouping per the §3.2 precedence table *)

binary_op       = "+" | "-" | "*" | "/" | "%"
                | "==" | "!=" | "<" | "<=" | ">" | ">="
                | "&&" | "||"
                | "&" | "|" | "^" | "<<" | ">>" ;

cast_expr       = unary_expr { "is" ( implied_variant | named_type ) [ "|" capture "|" ] | "as" type | "to" type } ;

unary_expr      = unary_op unary_expr | postfix_expr ;
unary_op        = "-" | "~" | "!" | "&" | "new" | "move" ;

postfix_expr    = primary_expr { postfix_suffix } ;
postfix_suffix  = "(" [ expr { "," expr } ] ")"                            (* call *)
                | "<" type { "," type } ">" "(" [ expr { "," expr } ] ")"  (* generic call *)
                | "." identifier                                                  (* member access *)
                | "[" expression "]"                                         (* array index *)
                | "[" [ expression ] ".." expression "]" ;                   (* subslice *)

primary_expr    = literal
                | identifier_expr
                | implied_variant
                | named_struct_init
                | anon_struct_init
                | "(" expression ")"
                | array_fill
                | array_range
                | array_literal
                | if_expr
                | for_expr
                | while_expr
                | match_expr
                | lambda_expr
                | comptime_expr ;

identifier_expr   = identifier { "::" identifier } ;
implied_variant   = "::" identifier ;
literal           = integer_literal | float_literal | string_literal | char_literal | "true" | "false" ;

array_literal     = "[" [ expression { "," expression } ] "]" ;
array_fill        = "[" expression ":" expression "]" ;
array_range       = "[" [ expression ] ".." expression "]" ;

named_struct_init = identifier { "::" identifier } [ "<" type { "," type } ">" ] "{" [ member_init { "," member_init } ] "}" ;
anon_struct_init  = "{" [ member_init { "," member_init } ] "}" ;
member_init       = "." identifier "=" expression ;

lambda_expr       = [ "|" capture { "," capture } "|" ] function ;
capture           = identifier [ ":" ( type | type_modifier ) ] ;

if_expr     = "if" "(" expression ")" statement [ "else" statement ] ;

for_expr    = "for" "(" expression { "," expression } ")"
              [ "|" capture { "," capture } "|" ]
              statement [ "else" statement ] ;

while_expr  = "while" "(" expression ")" statement [ "else" statement ] ;

match_expr  = "match" "(" expression ")" "{"
              { ( expression | "else" ) [ "|" capture "|" ] statement }
              "}" [ "else" statement ] ;

comptime_expr   = "#" ( postfix_expr | struct_type | enum_type ) ;
```

**Comptime prefix.** The `#` token marks any value-yielding expression for
compile-time evaluation (§7). It binds as a postfix expression: `#f(x)` is
`#(f(x))`, but `#a + b` is `(#a) + b` — parenthesise to mark a whole expression
(`#(a + b)`). A `#` may also prefix an inline `struct { … }` or `enum { … }`
layout, which yields the `#Type` describing that layout (§4.4) — the anonymous
counterpart of `#u32` / `#Packet`.

**Semicolon termination.** The `;` separator terminates statements and
declarations. Newlines are plain whitespace, so a statement may span any
number of lines and several statements may share one. A statement also
terminates implicitly before `}`, before `else`, and at end of file, which
permits forms like `{ yield 10 }` and `yield a else yield b` without a
trailing `;`. Redundant semicolons are permitted and parse as empty
statements.

**Array types.** `[T : N]` is the fixed array; the count is any expression the
compiler can evaluate at compile time (§4.2), not only a literal. The sizeless
form `[T]` is not a type on its own — it appears only under a `&` or `*`
modifier (`&[T]`, `*[T]`), never bare.

**Array-fill count.** The count in `[value : count]` is any expression. When
the fill produces a stack-allocated fixed array (`[T : N]`), the count must be
compile-time evaluatable — verified by a later compilation stage, not by the
grammar. A runtime count is valid only for heap allocation (`new [value : count]`,
§5.2).

**Empty array literal.** `[]` is the array literal of length zero: the
canonical empty view (null data pointer, zero length — safe because a
zero-length view is never dereferenced). It has no elements to infer from, so
it is only valid **where a slice `&[T]` is expected**, adopting that element
type; fixed arrays keep their at-least-one-element rule, and `[]` anywhere
without a slice-typed context is a compile-time error.

**Range generators.** `[start..end]` generates the array of consecutive
integers from `start` (inclusive) to `end` (exclusive): `[0..5]` is
`[0, 1, 2, 3, 4]`. Omitting the start (`[..end]`) starts at `0`. Both bounds
must be integers (§4.3 rules apply; untyped literal bounds default to `i32`),
the bounds must agree on one integer type, and a literal `end` smaller than a
literal `start` is a compile-time error. Like the array fill, literal bounds
produce a stack array `[T : end - start]`; runtime bounds require `new`
(yielding `*[T]`) — except as a `for` subject, where the range never
materializes and runtime bounds are always valid (§5.3).

**Subslicing.** `arr[start..end]` borrows a slice (`&[T]`) viewing the
subject's elements `start` (inclusive) to `end` (exclusive) **in place** — no
copy is made. Omitting the start (`arr[..end]`) starts at `0`. The subject may
be any array form (`[T : N]`, `&[T]`, `*[T]`); the view is mutable when the
subject location is. Bounds must satisfy `start <= end <= length`; a violation
is a runtime fault in checked builds. Like any reference, the view is
invalidated when the subject is dropped, moved, or reallocated.

**`break` and `yield` operand termination.** When a `break`'s or `yield`'s
operand is a block-like expression (`if` / `while` / `for` / `match`), that
operand is self-terminating: `break if (c) yield a else yield b`.

**Disambiguation.** Two forms need bounded lookahead, and an implementation
**must** resolve them this way:

- A `<` following a name opens a **generic argument list** only when the
  matching `>` is immediately followed by `(` or `{`; otherwise the `<` is the
  less-than operator. So `f<a, b>(c)` is a generic call and `Vector<u8> { … }`
  a generic struct literal, while `a < b > (c)` is a comparison chain.
- A `{` in **statement position** always begins a block. An anonymous struct
  literal used as a statement must be parenthesised — `({ .x = 1 })`. In value
  position (after `=`, as an argument, as an operand) a `{` begins a struct
  literal.

**`is` capture versus bitwise OR.** A `|` token immediately following an `is`
test's type operand always opens a **capture clause**, never the bitwise-OR
operator — the capture binds tighter than any binary operator, so
`a is ::Some |b| && c` is `(a is ::Some |b|) && c`. A bitwise OR over an `is`
result must parenthesise the test: `(a is ::Some) | b`.

**Capture typing.** A capture (`|a|`) binds a **deep copy** of the captured
value by default. An explicit `: type` annotation overrides this, and the
annotation may also be a bare type modifier as shorthand — the captured value's
own type is then inferred, so `|a: &|` reads "whatever `a` is, I want a
reference to it":

- `|a|` — deep copy (default)
- `|a: T|` — deep copy, type stated explicitly
- `|a: &T|` / `|a: &|` — immutable reference to the value in place
- `|a: &var T|` / `|a: &var|` — mutable reference; the subject must be mutable
- `|a: *|` / `|a: *var|` — owning capture: takes ownership of the captured
  value, exactly like `move` (§5.2). The pointer transfers into `a`, and the
  source — the enum subject of an `is` test, or the captured variable of a
  lambda — is invalid (moved-from) after the construct. Valid only when the
  captured value's type is itself a pointer (`*T` / `*var T` / `*[T]`), and
  the subject must be mutable.

This is the only capture syntax, and it is identical at every capture site —
lambda capture lists, inline `is` tests (`x is ::Some |v|`, §4.2), `for`, and
`match` arms. A modifier written **before** the name (`|&var x|`) is not
valid Alloy.

Interface-object captures are the exception to the copy default: they always
bind by reference, mirroring the subject's indirection (§4.2).

### 3.2 Operator Precedence & Associativity

Higher number = tighter binding. All binary operators are **left-associative**.

| Precedence     | Operators         |
| -------------- | ----------------- |
| 100 (tightest) | `*` `/` `%`       |
| 90             | `+` `-`           |
| 80             | `<<` `>>`         |
| 70             | `<` `<=` `>` `>=` |
| 60             | `==` `!=`         |
| 50             | `&` (bitwise AND) |
| 40             | `^`               |
| 30             | `\|` (bitwise OR) |
| 20             | `&&`              |
| 10 (loosest)   | `\|\|`            |

The `expression` production above is written flat; the table alone fixes the grouping, and equal-precedence operators group left to right. Unary prefix operators (`-`, `~`, `!`, `&`, `new`, `move`) bind tighter than all binary operators and are **right-associative**. Postfix operators (call `()`, generic call `<>()`, member `.`, index `[]`) bind tighter than all unary prefix operators. The cast operators (`is`, `as`, `to`, §4.5) bind looser than unary prefix operators and tighter than all binary operators.

---

## 4. Type System & Static Semantics

### 4.1 Primitive Types

| Name   | Width   | Signed | Float |
| ------ | ------- | ------ | ----- |
| `u8`   | 1 byte  | No     | No    |
| `u16`  | 2 bytes | No     | No    |
| `u32`  | 4 bytes | No     | No    |
| `u64`  | 8 bytes | No     | No    |
| `i8`   | 1 byte  | Yes    | No    |
| `i16`  | 2 bytes | Yes    | No    |
| `i32`  | 4 bytes | Yes    | No    |
| `i64`  | 8 bytes | Yes    | No    |
| `f32`  | 4 bytes | —      | Yes   |
| `f64`  | 8 bytes | —      | Yes   |
| `bool` | 1 byte  | No     | No    |

Boolean literals are explicitly reserved via the language keywords `true` and `false`. Character literal primitive types scale automatically to fit their byte layout width (§2.6).

### 4.2 Composite & Derived Types

| Syntax                         | Kind                    | Notes                                                                                  |
| ------------------------------ | ----------------------- | -------------------------------------------------------------------------------------- |
| `struct { f: T, ... }`         | Struct                  | Declaration-order members                                                              |
| `enum { A, B: T, ... }`        | Enum (sum type)         | Variants with optional payload                                                         |
| `[T : N]` (N > 0)              | Fixed array             | Size is a compile-time-evaluatable expression; allocated on stack/inline               |
| `&[T]`                         | Slice                   | Unmanaged view (fat pointer containing a raw pointer + a `u64` length)                 |
| `*[T]`                         | Dynamically Sized Array | Managed heap pointer to runtime-sized memory layout handle                             |
| `(T1, T2) -> R`                | Function type           | First-class function value                                                             |
| `type X = BaseType`            | Named alias             | Nominally distinct; transparently assignable to its underlying type                    |
| `*T`                           | Immutable pointer       | Managed heap-allocated instance                                                        |
| `*var T`                       | Mutable pointer         | Managed mutable heap-allocated instance                                                |
| `&T`                           | Immutable reference     | Unmanaged borrowed reference                                                           |
| `&var T`                       | Mutable reference       | Unmanaged mutable borrowed reference                                                   |
| `&I` / `*I` (`I` an interface) | Interface object        | Dynamic-dispatch fat pointer: a data pointer plus the concrete type's identity (§6.2). |

An **interface used as a type** (only behind an indirection — `&I`, `&var I`, `*I`, `*var I`) is an _interface object_: a fat pointer carrying the address of a value together with the **identity of its concrete type**; calls through it dispatch at runtime to that type's implementations (§6.2). A value of concrete type `T` is implicitly convertible to an interface object of `I` if and only if `T` declares `I` among its interface markers (`type T : I = ...`). The reverse conversion (interface object down to a concrete type) is expressed through the same constructs used for enum discrimination:

- **By `match` (one arm per concrete type).** A `match` whose subject is an interface object accepts concrete-type names as arm patterns, with a payload capture that binds a reference to the concrete value:

```alloy
match (shape) {            // shape: &Shape
    Circle |c| { /* c: &Circle */ }
    Square |s| { /* s: &Square */ }
    else    { /* unhandled concrete type */ }
}
```

The capture's indirection mirrors the subject's: a `&Shape` subject yields `&Concrete` captures, a `&var Shape` subject yields `&var Concrete` captures, and similarly for `*` / `*var`.

- **By `is` (single-type test).** The `is` keyword tests whether an interface object's concrete type matches a target type, and — with an optional capture clause attached directly to the test — also produces the downcasted reference:

```alloy
if (shape is Circle |c|) { /* c: &Circle, only runs when shape's concrete is Circle */ }
```

`x is Type` is a boolean expression in its own right, and the `|c|` capture attaches **inline to the `is` test itself**. A capture is only permitted where its success dominates every use: on a direct `&&` conjunct of an `if` or `while` condition. Each conjunct may carry its own capture, and each binding is visible to the conjuncts after it and to the branch body — several variables can be captured in one condition:

```alloy
if (cursor.peek() is ::Some |first| && first == '/' && cursor.peek(1) is ::Some |second|) {
    // first and second both bound here
}
while (cursor.next() is ::Some |token|) { /* token re-binds each iteration */ }
```

A capture under `||`, under `!`, or outside an `if`/`while` condition is a compile-time error (a failed test could still reach the capture's uses). In a `while`, the captures re-bind on every iteration and scope over the body; in an `if` they scope over the then-branch only — never the `else`.

The `is` operator accepts two kinds of left operand:

1. **Interface object** — the right operand is a concrete type implementing the same interface, and the capture binds the downcasted value (above).
2. **Enum value** — the right operand is a variant of that enum's type, and `is` evaluates whether the enum currently holds that variant. The capture then binds the variant's **payload**, and is only permitted on variants that carry one:

```alloy
type SomeEnum = enum {
    ValueA: T,
    ValueB,
};

var val: SomeEnum = SomeEnum::ValueA(t);

if (val is SomeEnum::ValueA |a: &T|) { }   // a borrows the payload in place
if (val is SomeEnum::ValueA |a: &|) { }    // same, payload type inferred
if (val is SomeEnum::ValueA |a: T|) { }    // a is a copy, type stated
if (val is SomeEnum::ValueA |a|) { }       // a is a copy (default)
```

Capture binding follows the capture-typing rules (§3.1): a deep copy by default, a borrow when annotated with a reference modifier (`&` / `&var`). An owning capture (`*` / `*var`) takes the payload **out** of the enum:

```alloy
type Holder = enum {
    Boxed: *T,
    Empty,
};

var h: Holder = Holder::Boxed(new T {});

if (h is Holder::Boxed |p: *|) {
    // p owns the payload allocation
}
// h is invalid (moved-from) after the if, whether or not the branch ran
```

When the test matches, ownership of the payload transfers into the capture, the remainder of the enum value is dropped, and the subject binding is cleared — exactly like `move` (§5.2). When it does not, the subject value is reclaimed by its normal scope-end drop. In both cases the subject binding is treated as **moved-from after the construct**; using it again is a use-after-move error. Owning captures are only valid when the payload type is itself a pointer (`*T` / `*var T` / `*[T]`), since only pointers are movable (§5.2); the mutability requirements are those of §3.1.

#### Implied enum variants (`::Variant`)

A variant may be written without its enum name by prefixing it with `::`, which tells the compiler to infer the enum type. This works everywhere a variant is named: construction (`::Some(x)`, `::None`), `match` arm patterns, and `is` targets. A bare variant name without the `::` prefix is never valid.

The inference resolves in two steps, and is only valid when **exactly one candidate** matches in the given context:

1. When a contextual type is available (a declared variable type, an expected payload or return type, a `match` or `is` subject) and that type is an enum carrying the variant, it is the candidate. As a call argument, the context is the parameter type of each overload candidate in turn (§4.6): a candidate whose parameter cannot resolve the variant is simply not viable, so `::X` participates in overload selection like any other argument.
2. Otherwise every visible enum definition is searched for a variant of that name. Exactly one carrying it makes that enum the candidate; none or several is a compile-time error (the ambiguity is resolved by spelling the enum name).

```alloy
var maybe: Option<u32> = ::Some(7);   // context: the declared type
match (state) {
    ::Idle { }                        // context: the match subject
    ::Busy |load| { }
}
if (state is ::Busy |load|) { }       // context: the 'is' subject
```

Generic enums infer their type parameters through the same unification as named construction (§4.7).

### 4.3 Type Compatibility & Coercion Rules

1. **Identity** — identical types are always compatible.
2. **Untyped integer literal** — compatible with any numeric primitive type (not `bool`), or any named alias whose underlying chain reaches one. Regardless of radix format (`0x`, `0b`, `0o`, decimal), it resolves to `i32` when no contextual type is available. An array literal whose elements are untyped likewise adopts a contextual element type (`var a: [u8 : 3] = [1, 2, 3]`).
3. **Untyped float literal** — compatible with any float primitive (`f32`, `f64`). Resolves to `f32` when no contextual type is available.
4. **Named alias transparency (one-way)** — a named type is compatible with its own underlying type, and with anything that underlying type is compatible with **that is not itself a named type**: a primitive, an anonymous structural layout (rule 6), or an inline enum (rule 7). A named type is **never** implicitly compatible with a _different_ named type, even one of identical layout; that conversion is the reinterpretation cast's job (`x as T`, §4.5).
5. **Numeric widening** — a numeric primitive is implicitly compatible with a wider primitive of the **same sign class**: unsigned→unsigned, signed→signed, float→float (`f32`→`f64` is implicit; the reverse requires `to`). Cross-class conversions require an explicit conversion cast (`x to T`, §4.5).
6. **Structural struct compatibility** — Two named struct types are nominally distinct. A target expecting an **anonymous** structural layout (`struct { a: u8, b: f32 }`) accepts any value — named or anonymous — whose field list matches **exactly**: the same field names and types, in the same order (recursively, so nested structural layouts compare the same way). There is no width subtyping — extra or missing fields never coerce. Converting between distinct **named** types of equal layout is the reinterpretation cast's job (`x as T`, §4.5).
7. **Structural enum compatibility** — an inline `enum { ... }` type is compatible with any enum type — named or inline — whose **ordered variant list matches exactly**: same variant names in the same order, with identical payload types. Two distinct _named_ enums remain nominally distinct even when their shapes match; the structural rule applies only when at least one side is an inline enum type.

Inline `struct { ... }` and `enum { ... }` types are permitted **wherever a type is expected**: parameter and return types, variable annotations, struct fields, enum payloads, generic arguments, and array elements. Values of an inline enum type are constructed with the implied-variant syntax (`::Variant`, §4.2), since the type has no name to qualify with.

### 4.4 Compile-Time Special Types (`#Type`)

`#Type` is a dedicated system representation primitive available **exclusively during compile-time evaluation**. A `#Type` value is a first-class, mutable description of a type — a struct or enum layout, a primitive, or an interface — that compile-time code may inspect and reconstruct.

- `#Type` maps directly to abstract structures, built-in primitives, or structural layouts.
- It exposes programmable compile-time methods enabling reflection and mutation (see below).
- Any attempt to retain or use `#Type` inside a standard runtime declaration or variable state triggers an immediate compile-time error.

#### Obtaining a `#Type`

- **`#T`** — prefixing a type with the comptime token `#` yields the `#Type` reflecting it: a named type (`#u32`, `#Packet`) or an inline layout written in place (`#struct { id: u32 }`, `#enum { A, B: u8 }`). `#T.member_names()` reflects on `T` directly.
- **Built-in macros** — `#type_of(expr)`, `#struct_type()`, `#enum_type()`, and `#implementers_of(I)` all produce `#Type` values; their signatures and exact semantics are specified once, in §7.4.
- **`#void`** — the `#Type` denoting the absence of a payload. Passed as the member type to `add_member` on an enum `#Type`, it marks a payload-less variant; reflected enums encode their payload-less variants the same way in `member_types()`. `void` is not a value type: `#void` in any runtime position is a compile-time error.

#### `#Type` methods

All are evaluated at compile time and called with dot syntax on a `#Type` value.

| Method                 | Signature                      | Semantics                                                                                                                                                                                                                                                                                                                                                                                        |
| ---------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `is_struct`            | `() -> bool`                   | True iff the type is a struct.                                                                                                                                                                                                                                                                                                                                                                   |
| `is_enum`              | `() -> bool`                   | True iff the type is an enum.                                                                                                                                                                                                                                                                                                                                                                    |
| `is_primitive`         | `() -> bool`                   | True iff the type is a built-in primitive.                                                                                                                                                                                                                                                                                                                                                       |
| `is_interface`         | `() -> bool`                   | True iff the type is an interface.                                                                                                                                                                                                                                                                                                                                                               |
| `implements_interface` | `(other: #Type) -> bool`       | True iff this type implements `other` (an interface), by the same conformance rule as §6.2 — declared markers resolved by definition identity (same-named interfaces from different libraries never confuse it) plus `Number` lang-item conformance for the primitives. A generic interface matches by definition, ignoring its instantiation arguments. Synthesised `#Type`s implement nothing. |
| `name`                 | `() -> &[u8]`                  | The type's declared name.                                                                                                                                                                                                                                                                                                                                                                        |
| `equals`               | `(other: #Type) -> bool`       | True iff the two `#Type`s denote the same type.                                                                                                                                                                                                                                                                                                                                                  |
| `add_member`           | `(name: &[u8], member: #Type)` | Appends a member (struct field or enum variant) of the given name and type.                                                                                                                                                                                                                                                                                                                      |
| `remove_member`        | `(name: &[u8])`                | Removes the member with the given name.                                                                                                                                                                                                                                                                                                                                                          |
| `member_names`         | `() -> &[&[u8]]`               | Member names, in declaration order.                                                                                                                                                                                                                                                                                                                                                              |
| `member_types`         | `() -> &[#Type]`               | Member types, parallel to `member_names()`.                                                                                                                                                                                                                                                                                                                                                      |

A `#Type` is a **value**: `add_member` / `remove_member` mutate the `#Type` value in hand, not the original type it was reflected from. The final `#Type` value, assigned through `type T = <#Type-valued comptime expression>`, becomes the synthesised type `T`. A synthesised struct behaves as a structural layout (§4.3 rule 6); a synthesised enum behaves as an inline enum type (§4.3 rule 7), and its variants are constructed, matched, and tested through the alias name (`T::Variant`) like any declared enum.

### 4.5 Casting

Two explicit cast operators cover every cast the coercion rules (§4.3) do not perform implicitly. Both are binary keyword operators (`cast_expr`, §3.1) whose right operand is a type.

| Operator | Name                  | Semantics                                                                                                              |
| -------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `x as T` | Reinterpretation cast | Reinterprets the bytes of `x` as type `T` without changing them. No runtime cost.                                      |
| `x to T` | Conversion cast       | Converts the value of `x` to type `T`, producing a new value (e.g. numeric conversion between sign classes or widths). |

- `as` requires the source and target layouts to have the same byte width, computed by the §4.9 layout rules for every type. Applied to a reference (`&S`), it yields a reference (`&T`) viewing the same memory without copying; the width requirement then applies to the pointee layouts `S` and `T`, not to the references themselves.
- `to` is defined for `Number` types (§6.2) and follows standard numeric conversion semantics (truncation, sign conversion, float/integer rounding).
- `is` (§4.2) belongs to the same grammar family but is a runtime test — of an enum's current variant or an interface object's concrete type — yielding `bool` and optionally binding an inline capture.

```alloy
var raw: u32 = 0x3F800000;
var f = raw as f32;         // same bits, viewed as f32 (1.0)
var n: i64 = -5;
var u = n to u32;           // numeric conversion, value-changing
```

### 4.6 Function Overloading

Functions (free functions and extension functions alike) may **overload**: any
number of functions may share a name as long as their parameter type lists
differ. Declaring two functions with the same name and an identical parameter
type list is a redeclaration error, as is reusing a function's name for any
non-function definition (§6.4) — except a **macro**, which may share a name
with functions because the `#` invocation form disambiguates (§7.3).
`extern` declarations do not overload — a C symbol name is unique.

**Overload resolution.** At a call site the candidate set is every visible
function with the called name. A candidate is _viable_ when its arity matches
and every argument is compatible (§4.3) with the corresponding parameter type.

- Exactly one viable candidate: it is called.
- Several viable candidates: if exactly one matches every argument type
  **identically** (no coercion), it wins; otherwise the call is **ambiguous**
  and a compile-time error.
- No viable candidate: a compile-time error listing the candidates.

### 4.7 Generic Type Parameter Inference

A generic function's type parameters are bound at each call site:

1. **Explicit arguments** (`make<u64>(7)`) bind type parameters left-to-right.
2. Remaining parameters are **inferred by unification**: each declared
   parameter type is matched structurally against the corresponding argument
   type (the `self` receiver participates like any argument), and every
   occurrence of an unbound type parameter binds to the type found in its
   position. `vector.push(x)` with `fn push<T>(self v: &var Vector<T>, item: T)`
   binds `T` from the receiver and checks `x` against it.
3. When a contextual expected type is available, the declared return type
   unifies against it **before** the arguments, binding type parameters the
   same way generic variant construction does
   (`var v: Vector<u8> = Vector::empty();`); arguments then unify against
   (and may coerce to) those bindings.
4. A type parameter still unbound after unification is a compile-time error;
   conflicting bindings (`T` unified with both `u32` and `f32`) are too.

A bound type parameter must satisfy its constraint (`<T: Number>`, §6.2).
A constraint naming a **generic interface** (§6.2) supplies its type
arguments (`<T, It: Iterator<T>>`); a constraint's arguments may reference
the type parameters declared **to its left** in the same list, never later
ones. The bound type must conform to the constraint at exactly that
instantiation once the call's bindings substitute in (`It: Iterator<T>` with
`T = u64` accepts a `: Iterator<u64>` conformer, not a `: Iterator<u8>` one).
During overload resolution (§4.6) a candidate whose unification fails, leaves
a parameter unbound, or binds a type violating its constraint is simply not
viable.

Constructing a variant of a generic enum follows the same rules: in
`Option::Some(x)` the type parameters bind by unifying the variant's payload
type against the argument and against the contextual expected type, exactly
like a function call. A parameter left unbound by both sources is a
compile-time error (`cannot infer type parameter 'T'`). A generic variant
construction passed as a call argument takes its context from each overload
candidate's parameter type (§4.6), the same way implied variants do (§4.2).

A **struct literal** binds a generic type's parameters either from the
contextual expected type (`var v: Vector<u8> = Vector { ... }`) or
**explicitly, like a generic call**: `Vector<u8> { ... }` names the
instantiation before the braces (`named_struct_init`, §3.1). Explicit
arguments bind all parameters left-to-right and must match the type's
parameter count; they are the way to construct a generic value where no
context exists (`var v = Vector<T> { ... }` inside a generic body). A bare
generic literal with neither source is a compile-time error.

### 4.8 Mutability

Bindings are immutable by default and mutability is explicit:

- **`const`** declares an immutable binding; **`var`** declares a mutable one.
- **Parameters are immutable bindings.** A function mutates caller state only
  through `&var` / `*var` indirections passed to it.
- **Assignment** (including compound assignment) requires a **mutable target**:
  a `var` local, or a location reached through a `&var` / `*var` indirection.
- **`&var x`** (and `&var` / `*var` captures) require `x` to be mutable.
- Mutability pierces with pointee transparency (§5.2): a field accessed
  through a `&var T` value is mutable even when the reference binding itself
  is `const`; through `&T` it is immutable even on a `var` binding. For direct
  (non-indirected) access, fields and elements inherit the binding's
  mutability.

### 4.9 Data Layout

Struct layout is **C-compatible**: fields are laid out in declaration order, each aligned to its natural alignment, with padding inserted as in C; the struct's size is rounded up to a multiple of its largest field alignment. This makes `extern` FFI structs work without annotations and gives `as` reinterpretation (§4.5) well-defined widths. Enums are laid out as a tag (the smallest unsigned integer that fits the variant count) followed by the payload area sized and aligned for the largest payload, as C would lay out the corresponding tagged union.

---

## 5. Execution Semantics

### 5.1 Evaluation Order

**Eager (strict) evaluation.** All sub-expressions are fully evaluated before their result is used. Function arguments are evaluated **left-to-right** before the call.

**Short-circuit logical operators** are the one exception: `&&` evaluates its right operand only when the left evaluated to `true`, and `||` only when the left evaluated to `false`. Every other operator, the bitwise `&` and `|` included, always evaluates both operands. Short-circuiting is what makes an inline `is` capture safe to use in a later conjunct (§4.2).

### 5.2 Memory Model & Pointer Assignment Syntax

Alloy maps memory mechanics transparently using direct, predictable assignment rules:

#### Pointee Transparency

A pointer (`*T`, a managed heap-owning object) and a reference (`&T`, an unmanaged raw pointer in the C sense) are **always treated as their pointee** when used. There are no explicit dereference (`*ptr`) or member arrow (`->`) symbols: field access (`.field`), array indices (`[index]`), operators, function arguments, and plain reads all operate directly on the pointed-at value.

This includes ordinary assignment — reading a pointer or reference yields a **copy of the pointee**, never a copy of the address:

```alloy
var p: *T = new T {};
var x = p;              // x is a deep copy of the value p points at, type T
var r: &T = &p;         // r references p's pointee
var y = r;              // y is likewise a deep copy of the pointee
```

**Deep copies and pointer uniqueness.** Every assignment copies its right-hand side **deeply**: if the copied value owns heap — directly or through members, elements, or payloads — each owned allocation is duplicated into a fresh allocation owned by the copy. As a consequence, **two pointers can never point at the same object**: a `*T` is always the unique pointer to (and owner of) its allocation. The only way to hand an existing allocation to another binding is `move`, which transfers the pointer and clears the source, preserving uniqueness. References carry no ownership and **may alias freely** — any number of `&T` values can point at the same object.

Three operators step outside pointee transparency:

- **`move`** is the **only** operator that treats its operand as an address rather than a value: `var q: *T = move p` transfers `p`'s pointer into `q` (and clears `p`, see below). It is valid for any pointer-typed operand — `*T`, `*var T`, and `*[T]` alike — and always yields the operand's own pointer type.
- **`new <expression>`** evaluates any expression and deep-copies the resulting value into a fresh heap allocation, yielding a pointer: `new 5`, `new T {}`, `new [0 : n]`, `new some_local`. It is the pointer-producing allocator.
- **Unary `&`** yields a reference to any value — a stack local, a struct field, an array element, or a heap value behind a pointer (`&p` references the pointee of `p`). Applied to a heap array, `&` yields a **slice** (`&[T]`) viewing every element in place — the only non-owning view of a `*[T]`'s contents (`arr[start..end]` subslicing, §3.1, views a range the same way).

#### Explicit Assignment Restrictions

- **Assigning to a Reference (`&Type` / `&var Type`)**: The unary address-of operator `&` is **strictly required** on the right-hand side of the assignment (e.g., `var r: &i32 = &stack_var`). The empty array literal `[]` (§3.1) is the one exemption: it is already a slice value, with no place behind it to borrow.
- **Assigning to a Heap Pointer (`*Type` / `*var Type`) or Dynamically Sized Array (`*[T]`)**: The assignment expression **strictly requires** either the `new` allocation operator or the `move` ownership transfer keyword (e.g., `var p: *i32 = new 5`, `var p2: *i32 = move p`). A bare pointer on the right-hand side would copy the pointee (pointee transparency), never alias the pointer.
- **Reference results are borrowed explicitly.** A call whose result is reference-typed (`&T`, `&var T`, `&[T]`) does not flow bare into a **use site** — a binding, an argument, a member initializer, an assignment value, or a `return`/`break`/`yield`. Writing `&f()` keeps the borrow, visibly: unary `&` on an already-reference-typed call result passes the reference through verbatim and is the one exemption from `&`'s addressability requirement. A bare `&T` result at a use site **pierces** — the use consumes a deep copy of the pointee, consistent with pointee transparency on reads. A bare `&[T]` result is a compile-time error, since its pointee `[T]` is unsized: write `&f()` to keep the view or `new f()` to copy it into an owned `*[T]`. Positions that consume the temporary in place — a method receiver (`f().length()`), an index or subslice subject, an operand of `new`, a condition or match subject — take the result directly and need no marker.

- **Using a variable uses the value.** The same rule governs reference-typed **variables** (locals, parameters, fields, elements): a bare use at a use site means the _pointee value_, never the borrow. A bare `&T` variable consumes a **deep copy** of the pointee — so it no longer fits a `&T` parameter; passing the borrow is spelled `&x`. A bare `&[T]` variable is a **compile-time error** at a use site, because its value `[T]` is unsized and `[T]` is not a valid type: `&x` passes the view, `new x` copies the array into an owned `*[T]` (the only way to copy it). An interface-object variable likewise never flows bare — its erased value cannot be copied; write `&x`. Unary `&` on a place already holding a reference or slice yields _that_ borrow (`&view` on a `&[u8]` binding is the view itself, not a reference-to-reference). In-place consumers (receivers, index/subslice subjects, conditions, match and `for` subjects) read the variable directly, as always.

#### Slices (`&[T]`) versus Dynamically Sized Heap Arrays (`*[T]`)

- **Slices (`&[T]`)**: an unmanaged, non-owning view into a sequence whose bounds are unknown at compile time — the data-pointer + `u64`-length fat pointer of §4.2.
- **Dynamically Sized Heap Arrays (`*[T]`)**: Represent a completely managed heap instance block instantiated via a `new` allocation expression:

```alloy
var arr: *[u32] = new [0 : 120];    // 120 elements of u32, initialised to 0
var n: u64 = 120;
var dyn: *var [u32] = new [0 : n];  // count may be a runtime expression
```

The fill count in `new [value : count]` may be a compile-time integer literal (which also permits a fixed stack array `[T : N]`) **or a runtime expression** — a runtime count always allocates a dynamically sized heap array (`*[T]`) of `count` elements, each initialised to `value`.

- **Memory Layout & C-FFI Compatibility:** To retain total binary drop-in compatibility with legacy C ecosystems, a pointer to an Alloy dynamically sized array points directly to the memory address of the first active data element (`element[0]`).
- **Length Metadata Tracking:** The allocation's length value (returned via `arr.length()`) is stored automatically by the runtime in a dedicated metadata prefix block located **immediately before the array data pointer** (i.e., at a negative memory offset from the user-facing pointer address).

#### Ownership, `move`, and Structural Reclaim

Ownership is **structural and automatic**. A value _owns heap_ if it is a `*T` / `*var T` / `*[T]` pointer, a closure (which owns its captured environment), or a struct / array / enum that transitively contains an owning member, element, or active-variant payload. Plain references (`&T`, `&var T`) and slices (`&[T]`) are non-owning views and never own heap.

- **Scope-end drop.** When an owning local goes out of scope (at every `return` path and at the implicit fall-through), the runtime _drops_ it: it recursively frees the heap it owns. Dropping a pointer frees its allocation (the `*[T]` form releases the malloc base at `user_ptr - 8`); dropping a `*[T]` first drops every one of its elements (all elements of an array are initialised, so each is reclaimable); dropping a struct/array/enum drops each owning field/element/active payload; dropping a closure frees its environment. Recursive owning types (e.g. a `Node` struct holding a `*Node`) terminate at the first null pointer.
- **`move` transfers ownership.** `move` is the only operator that reads its operand as an address rather than a value (§5.2 Pointee Transparency). `var q = move p` copies the pointer into `q` and clears (zeroes) the source binding, so the source is no longer an owner. After `move p`, `p` is a null pointer and its scope-end drop is a no-op. `move` yields the operand's own pointer type (`*T`, `*var T`, or `*[T]`) — whole-struct transfer is therefore expressed by moving a `*Struct` pointer (or by borrowing through `&var`), not by copying the struct.
- **Returning owned values is explicit — no implicit move.** `return move v` transfers the local's allocation to the caller: the source is cleared and its scope-end drop is a no-op. A bare `return v` returns **by value** — like any read, it yields a deep copy of what `v` holds, and the local's own heap is reclaimed by its normal scope-end drop. The same by-value rule applies to `break v` and `yield v`. A value built in the `return` expression itself (a constructor, `new`, …) is owned by the caller directly.
- **Pointer parameters take ownership.** A parameter of type `*T` / `*var T` / `*[T]` declares "I am taking ownership of this allocation". The caller **must** either `move` an existing pointer in (`take(move p)`) or allocate inline (`take(new T {})`); a `*T` value is **never** borrowed. The callee becomes the owner, and the parameter is dropped at the function's scope end like any owning local (unless moved on or returned). To lend a value without transferring it, pass a reference (`&T` / `&var T`) instead.
- **By-value parameters follow assignment semantics.** Passing a non-pointer value by value deep-copies it into the parameter, exactly like assignment (§5.2 Pointee Transparency); references borrow without copying.
- **Free-on-reassign.** Assigning to an owning binding (`buf = new […]`, `obj.field = move p`) first drops whatever that binding currently owns, then stores the new value — so the previous allocation is reclaimed rather than leaked.
- **Integer overflow:** arithmetic that exceeds its type's range is a **runtime fault in checked builds** (like the null checks below) and **wraps two's-complement in release builds**. Compile-time evaluation and the interpreter always fault. Division by zero is a fault in every build mode.
- **Checked builds** insert a null check on every dereference of a `*T` binding. A use-after-move accesses the null slot and traps (`@llvm.trap`). **Release builds** skip the null checks, so a use-after-move dereferences null and the OS faults. The null-store on `move` itself is kept in every build mode: it is the moved-from mark the drop machinery reads, so scope-end drops and free-on-reassign stay single-free after a transfer.
- **Definite use-after-move is a compile-time error.** The compiler tracks moves of bare local variables flow-sensitively: after `move x` (or an owning lambda capture `|x: *|`), reading `x`, writing through `x.field`, moving it again, or capturing it is rejected at compile time — until a plain `=` rebinds it. The analysis merges branches conservatively: a move survives a merge only when every falling-through path performs it, so moves under a condition, inside one branch, inside a loop body, or of a field (`move x.inner`) remain checked **runtime** faults rather than compile errors.

**Growth is manual.** A `*[T]` array has a fixed length once allocated (its length lives in the metadata prefix described above); there is no in-place resize or `realloc` primitive. A growable collection (`Vector`/`String`) is built by hand: allocate a larger `*var [T]` buffer with a runtime-sized `new [value : count]`, copy the elements across, and reassign the owning field — free-on-reassign reclaims the old buffer automatically. The standard library's generic `Vector<T>` (`std/vector.alloy`) and owning `String` (`std/string.alloy`) are written exactly this way; their mutating operations (`push`, `append`, …) take a `&var self` receiver and are invoked as methods (`vector.push(x)`).

> **Manual-safety caveats.** Alloy does not run a borrow checker. References are unchecked raw pointers: a `&T` can outlive the value it points at (a dropped local, a moved-from or reassigned owner) and dangle. Double-frees are ruled out by construction — deep copying plus pointer uniqueness (§5.2) means no two owners ever share an allocation — at the cost of implicit allocation when an owning value is copied; use `move` to transfer or `&`/`&var` to borrow where a copy is not intended. Implicit allocation arises **only** from such value deep copies — a value that owns heap through members, elements, or payloads being assigned, passed, returned, or captured by copy. A pointer-typed position (a `*T` variable, parameter, or return) is always reached through an explicit `new` or `move`: pointee transparency makes a bare variable read as its value, so no bare operand can ever produce a pointer.

---

### 5.3 Control Flow Semantics

**`return [value]`**
Immediately exits the enclosing function, yielding `value` as its result.

**`break [value]`**
Exits the **innermost enclosing loop** — a `for` or a `while` — passing transparently through any `if` or `match` in between, so a conditional loop exit is written naturally:

```alloy
for (items) |item| {
    if (item.done()) { break; } // exits the 'for'
}
```

When a value is provided, the loop evaluates directly to that value — the same channel `yield` uses in a value-position loop (below). A `break` outside a loop is a compile-time error.

**`yield value`**
Produces the value of the **innermost enclosing value-position construct**: an `if`, a `match`, or a `for` / `while` that carries an `else` clause. A `yield` inside a loop that is not itself value-position passes transparently out through that loop to the nearest enclosing value-position `if` or `match` (the loop is exited on the way out, its owned locals dropping normally). A `yield` with no enclosing value-position construct is a compile-time error — a statement-position `if`, `match`, or loop has nothing to receive the value and is transparent to both `break` and `yield`.

#### `if` as a Value-Yielding Construct

An `if` used as a value yields via `yield value` in its branches; every branch must yield (or the construct is a compile-time error), and an `else` branch is required:

```alloy
var grade = if (score > 90) { yield "high"; } else { yield "low"; };
```

#### Path Termination

The compiler performs a conservative flow analysis over every function body (§ compile-time, no runtime cost). A statement **terminates** when control cannot fall out of it normally: `return`, `break`, and `yield` terminate; a block terminates when any of its statements does; an `if` with an `else` terminates when both branches do; a `match` terminates when every arm does; `while (true)` with no `break` reaching it never completes (divergence). Conditions are never assumed and ordinary loops always count as skippable.

- **Definite return:** a function or lambda that declares a return type must terminate on every path — control falling off the end is a compile-time error.
- **Definite yield:** every branch of a value-position `if` must terminate; every arm of a value-position `match` must terminate unless an external `else` supplies the fall-through value, in which case the external `else` must terminate; the `else` of a value-yielding loop must terminate.

#### Loop Semantics (`for` and `while`)

- A `for` subject is either **natively for-compatible** — the built-in array forms (fixed arrays `[T : N]`, slices `&[T]`, heap arrays `*[T]`) and range generators, which lower to counting loops below and take no interface marker — or a **custom iterable**: a type conforming to the `Iterable` interface (a lang item, §6.1a).
- **Custom iterables (`Iterable` / `Iterator`).** A user type becomes a `for` subject by declaring conformance to `Iterable`, whose `iterator` function yields a separate cursor value conforming to `Iterator` (both generic interfaces, §6.2):

  ```alloy
  // std::iterable (section 6.1a)
  pub interface Iterator<T> {
      fn next(self: &var) -> Option<&T>;
  }
  pub interface Iterable<T, It: Iterator<T>> {
      fn iterator(self: &) -> It;
  }

  // a conforming container binds 'It' to its concrete cursor type
  type Container<T> : Iterable<T, ContainerCursor<T>> = ...;
  type ContainerCursor<T> : Iterator<T> = ...;
  // the container yields a fresh cursor by value
  fn iterator<T>(self c: &Container<T>) -> ContainerCursor<T> { ... }
  // the cursor advances and yields a reference to the next element in
  // place, or None when exhausted
  fn next<T>(self it: &var ContainerCursor<T>) -> Option<&T> { ... }
  ```

  `for (c) |x| { ... }` lowers to: `it = c.iterator()` then repeatedly `match (it.next()) { ::Some |x| <body> ::None { break the loop } }`. The loop variable `x` follows the capture-typing rules like every other capture site (§3.1): a bare `|x|` deep-copies the element, `|x: &|` borrows it in place. The cursor hands the loop an `Option<&T>`, so a copying capture reads through that reference (§5.2) while a borrowing capture keeps it. A cursor-shaped type that does not declare the `Iterable` conformance is not a `for` subject — the marker is required, and the conformance check verifies `iterator`/`next` against the instantiated signatures (§6.2).

- **Counting-loop lowering.** Whenever the subject's iteration count and element access are directly available — every built-in array form (`[T : N]`, `&[T]`, `*[T]`) and every range generator (`[start..end]`, §3.1) — the compiler **must** lower the `for` to a plain counting loop (a C-style `for` over an index), never the cursor protocol. A range generator used as a `for` subject (`for ([..n]) |i| { ... }`) materializes no array at all: it lowers to a counter running from `start` to `end`, which also makes runtime bounds valid in this position without `new`. Element binding does not depend on the lowering: in either form a bare `|x|` copies the element and `|x: &|` borrows it in place (§3.1).

- **Multi-subject loops.** A `for` may take several comma-separated subjects, with one capture per subject in order: `for (a, b) |x, y| { ... }`. The subjects iterate in lockstep — each pass binds the next element of every subject — and every subject must itself be iterable. All subjects must produce the same number of elements; a length mismatch is a runtime fault in checked builds.

- **Expression-Only `else` Clause:** The trailing `else` block on a `for` or `while` loop is **only permitted when the entire loop construct is evaluated as an expression** (e.g., when assigning its value to a variable). Such a loop is a value-position construct, so a `yield value` in its body produces the loop's value directly (§5.3 `yield`); `break value` does the same. Either one ends the loop immediately. The `else` block runs **if and only if the loop finishes without producing a value** — the subjects are exhausted, or the `while` condition fails, and no iteration executed a `yield` or a valued `break` — and its own value then becomes the loop's value, which must match the body's type. Using an `else` arm on a loop that is executed purely as a statement is a compile-time error.

#### Match Expressions

```alloy
var x = match (subject) {
    Pattern1 |payload_capture| { yield 10 }
    Pattern2 { yield 20 }
} else {
    // External else block
    yield 30 // Required expression fallback
};
```

- **Subject Versatility:** The subject of a `match` statement can evaluate to an enum variant, a numeric primitive, a character literal, or a string literal (treated natively as an array of integral numbers). Enum arm patterns name variants either fully (`State::Idle`) or in the implied form (`::Idle`, §4.2); a bare variant name is invalid.
- **Pattern Captures:** The pattern capture clause (`|capture|`) is **exclusively valid** when matching enum variants containing attached data payloads. Utilizing a pattern capture when matching numbers, characters, or strings results in a compile-time error. Captures follow the capture-typing rules (§3.1): deep copy by default, optionally annotated (`|a: &|` borrows the payload in place; an owning capture `|a: *|` takes a pointer payload out, leaving the match subject moved-from after the `match`).
- **Exhaustiveness:** Only a `match` evaluated **as an expression** must cover all possible subject values — the construct must produce a value, so no subject may slip through. An enum subject is exhaustive when every variant appears as an arm pattern, or when an internal `else` arm is present. All other subjects — numbers, characters, strings, and interface objects — have open or unbounded domains and therefore always require an internal `else` arm in expression position. A non-exhaustive value-yielding `match` is a compile-time error. A `match` in **statement position** carries no such requirement: it may cover any subset of the subject's values, and when no arm matches, the statement simply does nothing. (The external `else` block below is not a coverage fallback: it handles an arm completing without `yield`, not an unmatched subject.)
- **Match Evaluation & Value Yielding:** Distinct match arms yield an evaluated value from the outer `match` expression block by terminating via a `yield value` statement. A `break` inside a match arm targets the enclosing loop (§5.3 `break`), never the match.
- **Expression-Only External Match `else` Block:** A `match` structure supports an optional **external `else` block** positioned after its closing bracket. This block is **only permitted when the match is evaluated as an expression**. It executes if and only if the selected match arm completes its execution path normally **without producing a value via a `yield` statement**. Because it is constrained to expression contexts, the external `else` block must also provide a final value matching the expression's expected return type. Appending an external `else` block to a `match` construct used purely as a statement is a compile-time error.

---

### 5.4 Lambda / Closure Semantics

```alloy
|x: &var, y| (param: T) -> R { body }
```

- The optional capture list (`|...|`) names variables from the enclosing scope. Each capture may carry an annotation after its name (`|x: &var|`) whose type modifier (`&`, `&var`, `*`, `*var`) controls how the outer variable is accessed within the lambda; the capture-typing rules of §3.1 apply unchanged.
- Capture lists are **value-only**. Type names — including the enclosing function's generic type parameters — remain visible inside the lambda's parameter types, return type, and body without being captured.
- A `*` / `*var` capture (`|x: *|`) is an **owning capture**: it takes ownership of the captured variable, moving its pointer into the closure environment. The outer binding is invalid (moved-from) after the lambda expression. Valid only for pointer-typed variables (§3.1 Capture typing).
- The parameter list and optional return type follow the same syntax as a regular function.
- The type of a lambda expression is the corresponding function type `(T) -> R`.
- A lambda with no captures omits the capture delimiters entirely: `(param: T) { ... }`. An empty capture list (`||`) is not valid Alloy — a capture list, when written, names at least one capture.
- A named, non-generic function used in value position becomes a function value of its signature's type. A generic function cannot become a function value (its type parameters are unbound), an overloaded name needs a unique function, and an extern function cannot be used as a function value.
- Function values have no defined identity or structural equality: comparing them with `==` / `!=` is a compile-time error.

### 5.5 Extension Functions

Any function whose first parameter is prefixed with `self` is an extension function:

```alloy
fn add(self v: &Vec3, other: &Vec3) -> Vec3 { ... }
```

- Called via dot notation: `v.add(other)`.
- The receiver is treated as an implicit first argument for the purpose of overload resolution.
- **Dot-call precedence:** extension resolution wins. When no extension (or interface function, §6.2) provides the name, a call `v.f(...)` where `f` is a function-typed field of `v` calls through the field's value (§5.4).
- `self` must appear only on the **first** parameter; any other position is an error.
- A **temporary receiver** (a call result, a literal construction) may invoke an extension whose `self` is an immutable reference (`&T`): the temporary materializes for the duration of the call. A `&var` receiver still requires a mutable place.
- A **pointer `self`** (`*T` / `*var T`) takes ownership of the receiver, exactly like a pointer parameter (§5.2): an owning place must transfer explicitly (`(move p).consume()`), and only a fresh value (`(new T {}).consume()`, a call result) passes bare.
- When the `self` receiver's type is an **interface** (e.g. `fn area(self s: &Shape) -> f32`), the extension is the interface function's **default implementation** — see §6.2.

---

## 6. Standard Library & Primitives

### 6.1 Arrays & Built-in Methods

All array forms — fixed arrays `[T : N]`, slices `&[T]`, and dynamically sized heap arrays `*[T]` — are **natively for-compatible** (§5.3): they lower to counting loops and implement **no** interface. There are no implicit interface implementations in the language; every conformance is declared (`type T : I = ...`, §6.2) — with one compiler-supplied exception, the primitive numeric types' conformance to `Number` (§6.1a), an interface that declares no functions of its own. A generic bound `<C: Iterable<...>>` therefore accepts only declared conformers, never a bare array or slice.

The compiler provides the arrays' `.length() -> u64` built-in method directly, because every array form already carries its size — each in a different place:

| Array form        | Where the length lives                                                   |
| ----------------- | ------------------------------------------------------------------------ |
| `[T : N]` fixed   | Known statically at compile time; `.length()` folds to the constant `N`. |
| `&[T]` slice      | The `u64` length half of the slice's fat pointer.                        |
| `*[T]` heap array | The metadata prefix stored immediately before the pointee (§5.2).        |

Custom types wanting a length supply a `length` extension themselves; it is not part of any interface contract (`Iterable` declares only `iterator`, §5.3).

Reinterpretation and conversion are performed by the `as` / `to` cast operators (§4.5), not by built-in methods.

### 6.1a Compiler-Recognized Declarations (Lang Items)

A small set of standard library declarations is **recognized by the compiler by canonical path** to power syntactic sugar. There is **no prelude**: lang items are ordinary Alloy source shipped with the standard library (§6.4), and nothing is in scope until imported. The compiler's own lowering references the canonical declaration directly — an import is only required to _name_ the item in source code.

| Lang item   | Canonical path            | Declaration                                                              | Compiler hook                                                                                                                                                                                                                                                                                                  |
| ----------- | ------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Option<T>` | `std::option::Option`     | `enum { Some: T, None }`                                                 | Cursor protocol: `for` over a custom iterable lowers to repeated `match` on `next()`'s `Option<&T>` result (§5.3).                                                                                                                                                                                             |
| `Iterable`  | `std::iterable::Iterable` | `interface Iterable<T, It: Iterator<T>> { fn iterator(self: &) -> It; }` | Gates `for` over custom types: the subject must declare the conformance (§5.3). Arrays take no marker — they are natively for-compatible (§6.1).                                                                                                                                                               |
| `Iterator`  | `std::iterable::Iterator` | `interface Iterator<T> { fn next(self: &var) -> Option<&T>; }`           | The cursor contract behind `for` lowering (§5.3); bound by `Iterable`'s own `It` constraint.                                                                                                                                                                                                                   |
| `Number`    | `std::number::Number`     | `interface Number { }` (no functions)                                    | Satisfied by the primitive numeric types (§4.1) without a declared marker — the compiler supplies this conformance, the language's only implicit one (§6.1). Bounds generic constraints and the `to` conversion cast (§4.5), and unlocks the arithmetic and comparison operators inside a generic body (§6.2). |
| `arguments` | `std::process::arguments` | `fn arguments() -> &[&[u8]]`                                             | The compiler supplies the command line (first element: the program's own path). Natively the entry wrapper captures argc/argv at startup; `alloyc run` serves the arguments after the program path. Unavailable at compile time (§7.2).                                                                        |

```alloy
import std::option;

var maybe: Option<u32> = Option::Some(42);
match (maybe) {
    Option::Some |v| { /* v : u32 */ }
    Option::None    { /* empty */ }
}
```

A user definition colliding with an imported lang-item name follows the normal redeclaration rules (§6.4); lang items carry no special naming privileges.

### 6.2 Standard & User-Defined Interfaces

The standard interfaces are lang items (§6.1a) defined in ordinary standard library source:

| Name       | Satisfied by                                                                                                                      |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `Number`   | `u8` `u16` `u32` `u64` `i8` `i16` `i32` `i64` `f32` `f64`                                                                         |
| `Iterable` | Custom types declaring the conformance and providing `iterator()` (§5.3). Arrays never — they are for-compatible natively (§6.1). |
| `Iterator` | Cursor types declaring the conformance and providing `next(self: &var) -> Option<&T>` (§5.3).                                     |

Used as a type-parameter constraint: `fn foo<T: Number>(...)`.

#### Generic Interfaces

An interface may declare its own type parameters (`interface Iterator<T> { fn next(self: &var) -> Option<&T>; }`), which scope over every declared signature. A parameter may itself be constrained, and a constraint's arguments may reference the parameters declared to its left (`interface Iterable<T, It: Iterator<T>>`, §4.7).

A conformance marker for a generic interface supplies **all** of its type arguments (`type Vector<T> : Iterable<T, VectorCursor<T>>`); the arguments may mention the conforming type's own parameters. Verification (below) substitutes the marker's arguments into the declared signatures — the satisfying extension must match the **instantiated** signature — and each argument must itself satisfy the interface's own constraints at that instantiation.

**Generic interface objects require full instantiation.** A generic interface forms an interface object only with **every** type parameter bound to a concrete type (`&Iterator<u64>`, `&var Iterator<u64>`): the instantiation pins each signature, so one dispatched call site has a fixed ABI even though the concrete implementer behind the pointer is unknown until runtime. There is **no partial erasure** — a bare `&Iterator` is a compile-time error, and a form like `&Iterable<u64>` (with `It` erased) is inexpressible by the arity rule; an erased implementer-chosen type would give each implementer a differently sized result, which the call site cannot receive without an implicit allocation (banned, §5.2). Dispatch identity is **per instantiation**: a generic implementer conforms once per binding of its own parameters (`VectorCursor<u64>` and `VectorCursor<u8>` carry distinct identities behind `Iterator` objects), derived by unifying its conformance marker against the object's arguments. Two consequences:

- **No downcasting.** `is` and `match` on a generic interface object are compile-time errors — runtime identity selects dispatch targets but is not a nameable type test. Dispatch through the interface's functions instead.
- Inside a generic body a constrained value (`cursor: It` with `It: Iterator<T>`) still resolves its calls **statically** at each instantiation, exactly like any constraint call (role 2 below); the interface object is only for call sites where the concrete type is genuinely unknown at compile time.

#### User-Defined Interfaces

Interfaces define traits or constraints as named contract blocks of function signatures. Every interface function declares its receiver indirection — `self: &`, `self: &var`, `self: *`, or `self: *var` — as its first parameter, with no name and no type: the type is whatever concrete type implements the interface. Parameters after it are ordinary:

```alloy
interface Serializable {
    fn serialize(self: &, format: u32) -> bool
}
```

A nominal type alias links itself explicitly to one or more user-defined interfaces using a C++-inspired mapping annotation during its declaration syntax:

```alloy
type Packet : Serializable, Iterable<u64, PacketCursor> = struct {
    id: u32,
    payload: *[u8],
};
```

#### The Two Roles of an Interface

An interface may be used in **two distinct ways**:

1. **Dynamic dispatch.** An interface used as a type — always behind an indirection (`&I`, `&var I`, `*I`, `*var I`), with a generic interface fully instantiated (`&Iterator<u64>`, above) — produces an _interface object_ (§4.2). A value of any concrete type that implements `I` (at that instantiation) is implicitly convertible to such an interface object. Calling an interface function through an interface object (`handle.do_something()` where `handle: &Shape`, `it.next()` where `it: &var Iterator<u64>`) is resolved at **runtime through the carried type identity**, matched against the closed world of implementers (§6.4), to the concrete type's implementation.

2. **Generic constraint.** An interface used as a type-parameter bound (`fn do<T: I>(...)`) restricts the generic to types that implement `I`. The call is resolved **statically** at each instantiation; no runtime dispatch is involved. Inside the generic body, a value of type `T` exposes the constraint's interface functions via dot notation; a `T: Number` value additionally supports the arithmetic and comparison operators (§4.1).

#### Default Implementations

An **extension function whose `self` receiver is an interface** is the **default implementation** of that interface function:

```alloy
interface Shape {
    fn area(self: &) -> f32;
    fn name(self: &) -> &[u8]
}

// default implementation of Shape::name, shared by every implementer
fn name(self s: &Shape) -> &[u8] { return "shape" }
```

- A default implementation makes the corresponding interface function **optional** for implementing types.
- An extension function written for a **concrete type** _overrides_ the default for that type. When resolving a call on a concrete value, a type-specific extension is always preferred over an interface default.
- A **generic** interface's functions cannot have default implementations: the default's receiver would be a generic interface object, and those require full instantiation (above) — there is no one receiver type covering every instantiation.

#### Compilation & Verification Mechanics

When a type `T` is flagged with interface markers (`type T : I1, I2 = ...`), the compiler performs a static verification pass over the **merged compilation unit** (§6.4). For every function declared inside each interface (`I1`, `I2`):

1. A satisfying **extension function** (§5.5) must exist in the merged unit — either an extension belonging to `T` itself, **or** a default implementation (an extension whose `self` receiver is the interface). Visibility does not enter into it: satisfaction is closed-world, so a library-internal extension satisfies a marker exactly as an exported one does (§6.4).
2. The satisfying extension must precisely match the method name, the parameter sequence, and the return type specified by the interface declaration — with a generic interface's marker arguments substituted into the declared signature first, and the candidate's own type parameters bound through its receiver (`fn next<E>(self c: &var Cursor<E>)` verifies against `Cursor<T>` with `E = T`). The parameters following `self` correspond positionally to the interface function's parameter list.
3. The receiver indirection is fixed by the **interface**, not by the implementer: a signature declares it as `self: &`, `self: &var`, `self: *`, or `self: *var`, and the satisfying extension's first parameter must carry exactly that indirection over its own type (`fn next(self: &var)` is satisfied by `fn next(self c: &var Cursor<T>)`, never by `fn next(self c: &Cursor<T>)`). Because every implementer agrees on the receiver form, an interface object's call sites have one well-defined ABI, and the object's own indirection (`&I`, `&var I`, …) must be at least as permissive as the function's receiver — calling a `self: &var` function through a `&I` object is a compile-time error.
4. If neither a type-specific extension nor a default implementation exists, verification fails with a compile-time error.

---

### 6.3 Extern FFI

External C functions are described with fixed, concrete signatures. The return arrow is **optional**: an omitted `-> type` declares a C `void` function.

```alloy
extern functionName(param: Type) -> ReturnType;
extern variadicFunc(...) -> &var u8;
extern releaseBuffer(buffer: &var u8);         // C void
```

- **Architecture Strategy:** The FFI layer is intentionally isolated. Raw `extern` declarations are designed strictly for low-level systems developers contextually porting legacy C libraries to Alloy ecosystems. Standard software applications are expected to consume safe, native Alloy modules that seamlessly wrap and encapsulate these unsafe FFI barriers.
- **No owning returns:** an `extern` may not return `*T`, `*var T`, or `*[T]`. Those types carry Alloy ownership, and the drop machinery (§5.2) would free memory Alloy never allocated. Model a C pointer result as a reference (`&T` / `&var T`) or as an opaque integer handle — `std::io` types C's `FILE*` as `i64` — and manage its lifetime by hand in the wrapping module.
- **Slice decay:** a slice (`&[T]`) crossing the extern boundary — as a parameter or a variadic argument — passes only its **data pointer**, matching the C convention; the length stays behind. String literal bytes are NUL-terminated in static memory, so a literal passed to C is a valid C string.
- **Variadic promotions:** arguments in a variadic tail follow the C default promotions (integers narrower than 32 bits widen to `i32` by their own signedness, `f32` widens to `f64`).

### 6.4 Module System

- **Module paths mirror the filesystem:** `import a::b::c` names the source file `a/b/c.alloy`. The directories searched for it, in order, are given under `alloyc build` below.
- **Standard library:** The standard library ships as ordinary Alloy source files alongside the compiler. It is **not** a prelude — nothing is in scope until imported. The modules: `std::option` (`Option<T>`, §6.1a), `std::number` and `std::iterable` (the constraint interfaces, §6.2), `std::vector` (the growable `Vector<T>`), `std::string` (the owned `String` builder), `std::io` (console and file input/output — the library's FFI barrier, §6.3: programs call these wrappers and never touch the C functions underneath), `std::process` (`arguments()`, §6.1a), and `std::macros` (the declaration-only built-in macros, §7.4).
- **`alloyc build` import resolution:** When compiling a single file to a native executable (`alloyc build file.alloy [-o out] [--release] [--emit-llvm]`), each `import a::b::c` is resolved to `a/b/c.alloy` searched under, in order: the **importing module's directory**, the **entry module's directory**, the compiler-executable's directory, and `$ALLOY_STDLIB`. The importing-module step means a module in a subdirectory reaches its siblings without spelling the directory (`import token_kind` inside `tokenizer/tokenizer.alloy` finds `tokenizer/token_kind.alloy` first); a module found there takes the directory-qualified canonical key (`tokenizer::token_kind`), while its unqualified alias stays the import path's last segment. `std::` and `pkg::` imports skip the importing-module step. Resolution never depends on the shell's working directory, so a build means the same thing from anywhere (`alloyc build src/main.alloy` resolves imports against `src/`), and matches how editor tooling resolves imports against the open file. A `pkg/` folder of `.alloylib` containers is likewise found next to the entry module. Every reachable module is **merged into one compilation unit**. The build is **checked** by default and `--release` selects the release semantics of §5.2; the backend emits LLVM IR and an external clang (located via `$ALLOY_CLANG`, then `PATH`) produces the executable.
- **Debug info:** checked builds embed DWARF debug metadata — file, function, and statement-level line/column locations — so native debuggers (LLDB, GDB) set source breakpoints, step by statement, and show Alloy names in call stacks. Release builds carry none. On Windows the debug link goes through lld (`/debug:dwarf`); if lld is unavailable the build falls back to linking without debug info and says so.
- **Program entry:** execution starts at a zero-parameter function named `main` in the entry module. An integer result becomes the process exit code (truncated to the platform's width); any other result type, or none, exits with 0.
- **Qualified vs. unqualified access:** an imported name may be written either unqualified (`Vector`) or module-qualified (`std::vector::Vector`). Every import also introduces an alias for qualified use — the explicit `as` name or the import path's last segment (`import pkg::mathx` allows `mathx::twice(...)`). Qualified access goes through the cross-module visibility check, so only `pub`/`exp` definitions are reachable that way. Unqualified access sees the requester's **own library** in full (the executable's own modules and `std::` count as one library), plus the `exp` definitions of each library the module imported **without an explicit `as`** — an unaliased library import _injects_ its exports into that module's unqualified namespace, while an aliased import (`import pkg::liba as la`) is reachable through the alias only (`la::Pair`, `la::Pair { ... }`).
- **Qualified functions (constructors):** `fn Vector::empty<T>() -> Vector<T> { ... }` defines a plain free function living in the **type's namespace** instead of the module's flat namespace, called as `Vector::empty()` (or `alias::Vector::empty()` across an aliased import). The qualifier must name a type visible to the defining module - like extension functions, any accessible type qualifies, not only locally declared ones. Qualified functions of one type overload among themselves; the same name may freely exist as a free function or under other types. On an enum, a qualified name colliding with a variant is a compile-time error, so `Type::Name(...)` stays unambiguous and variant construction is unchanged. A qualified function must not declare a `self` receiver - it is a plain free function, not an extension (a `self` parameter is a compile-time error); no dot-call, no dispatch - the association is purely a namespace.
- **Name collisions:** within one library, a name colliding with an existing definition is a redeclaration error, except that functions overload (§4.6) and a **macro may share a name with functions** — `#name(...)` always invokes the macro, a bare `name(...)` never does (§7.3, e.g. the `#read_file` macro besides `std::io`'s runtime `read_file`). Different libraries may reuse names internally. A name visible unqualified in one module from **two different libraries** (own declaration vs. an injected export, or two injected exports) is a **compile-time error at the import**, resolved by aliasing an import to take its exports out of the unqualified namespace. Nothing is resolved implicitly — no shadowing, no cross-library overload merging. Two imports whose aliases collide (implicit or explicit) are likewise an error.
- **Merge-then-codegen:** every reachable module — including every library module — always merges into ONE compilation unit before type checking and code generation. The whole-program stages depend on it (closed-world interface dispatch, monomorphization, §4.9 layouts); only the per-module front-end stages (tokenize, parse) run in parallel. Libraries therefore recompile with each consuming program; the `.alloylib` payload exists to make that cheap, not to skip it.

#### Import Namespaces

- `import a::b` — a **relative** import: the file `a/b.alloy` next to the importing code. Inside a library, relative imports resolve within that library's own namespace.
- `import std::x` — the **standard library**, shipped as Alloy source alongside the compiler.
- `import pkg::name[::module]` — a **package**: the compiler looks for `pkg/name.alloylib` in the project first, then (future) the trusted registry. `pkg::name` imports the package's entry module; `pkg::name::module` one of its members.

#### Libraries (`.alloylib`)

- `alloyc lib entry.alloy [-o name.alloylib]` fully checks the unit standalone (all §4/§5 rules, flow and move analysis included) and packs the entry module plus every module of its own into a container. `std::` and `pkg::` dependencies are not packed — they stay imports the consuming program resolves, so a library's package dependencies load transitively.
- The container embeds the **complete source** — the registry mandates open source, and the embedded source is authoritative: precompiled cache sections (future) are stamped with the producing compiler's version and silently ignored on mismatch, falling back to compiling the embedded source. A library therefore never breaks across compiler releases.
- **Export boundary:** `exp` marks a definition as exported from a library. Within one compilation unit `exp` behaves exactly like `pub`; across a library boundary, only `exp` definitions are visible to consumers — qualified (`mathx::twice(...)`) or unqualified via an unaliased import per the injection rules above — while `pub` covers a library's internal cross-module structure without leaking it.
- **Interface satisfaction stays closed-world:** whether a type satisfies an interface (§6.2) considers every extension in the merged unit, even library-internal ones — dynamic dispatch spans the whole program. Visibility governs who may _name_ an extension in a direct call, not whether it backs dynamic dispatch.
- **Comptime re-runs per program** (§7): library comptime and macros are re-evaluated in the consuming program's merged unit, so compile-time reflection can see the final closed world (every interface implementer, every type), not just the library's own.

---

## 7. Compile-Time Evaluation & Macros

### 7.1 The Comptime Modifier (`#`)

Any **value-yielding expression** prefixed by the token `#` — an `#if`, `#while`, `#match`, an arbitrary function call (`#compute(x)`), an identifier, or a parenthesised expression — is intercepted by the compiler and executed at compile time via an internal interpreter. A `#`-marked construct must produce a value; a bare statement block (`{ … }`) is not a value and cannot be marked.

#### Value-Substitution Model

Comptime expressions operate on a pure value-substitution model. Once a compile-time expression completes execution, its entire syntax node tree is stripped from the final runtime code layout and replaced with its final calculated literal value, struct initialization block, or nominal type signature.

A value-yielding construct yields its value via `yield` (§5.3), so an `#if` selecting between two values is written:

```alloy
const a = #if (cond) yield 50 else yield 100;
```

#### Implicit Comptime Inheritance

When a `#` modifier marks an outer expression, all nested child expressions, loop structures, and evaluation flows contained within that outer node scope implicitly inherit compile-time evaluation.

#### Visible Names

A comptime expression may reference literals, any function in the program (including ones defined later in the file), and enclosing `const` locals whose initializers are themselves compile-time evaluable (transitively: a `const` built from literals, other compile-time-known constants, and function calls over them). Runtime state — `var` bindings, function parameters, and `const` locals whose initializers depend on runtime state — is invisible to compile-time evaluation; referencing it is a compile-time error.

A fault during compile-time evaluation (integer overflow, division by zero, an exhausted evaluation budget) is a compile-time error.

### 7.2 The Compile-Time Pointer Barrier & Sandboxing

To eliminate cross-compilation target safety errors and prevent memory leakage from host architectures into generated binaries, compile-time blocks are bound to strict computational constraints:

- **The Pointer Barrier:** A compile-time evaluation block cannot yield an unmanaged reference (`&T`), a managed pointer (`*T`), a heap array (`*[T]`), or a closure (which owns captured references) that escapes into a runtime variable. Any data crossing the boundary from compile-time execution to a runtime variable state must be handled strictly as values. Breaking this constraint triggers a compile-time error. Slices (`&[T]`) are the exception: a slice result is **materialized** — deep-copied into static program data — so a comptime call yielding a string works. A comptime result is always a value or a slice, never an owning pointer: static data has no owner to free it.
- **Strict Sandboxing Boundaries:** compile-time evaluation reaches the filesystem **only** through the comptime read built-in (`#read_file`, §7.4), and only for paths inside the project's root directory — the entry module's directory tree. Every other host capability is unreachable.
- **Foreign Function Isolation:** Comptime blocks are strictly prohibited from invoking low-level `extern` C functions (§6.3). Compile-time evaluation can only run safe user-defined Alloy code or built-in system macros. The `arguments` lang item (§6.1a) is unreachable for the same reason: there is no command line during compilation.

### 7.3 Macros

Macros are specialized compile-time functions used for code reflection, source file introspection, and automated type generation.

```alloy
macro readTypeFromJson(path: &[u8]) {
    // Introspection and type mutation using compile-time features
}
```

- **Signature and Inferred Types:** Macros are defined using the `macro` keyword. While macro input parameters are strictly typed, **macro return types are completely inferred by the compiler** based on the generated AST layout or the underlying type node replacement it yields.
- **Declaration-only macros:** A macro may be declared without a body (`pub macro type_of(value);`), analogous to an interface's functions: the compiler supplies the implementation. A declaration-only macro's parameters need no type annotations; a macro **with** a body must type every parameter. Invoking a declared macro the compiler does not implement is a compile-time error. The built-in macros (§7.4) are declared this way in `std::macros`.
- **Invocation Syntax:** To explicitly distinguish macros from standard functions, all macro calls must be preceded by the `#` character token. Because the invocation syntax disambiguates, a macro **may share its name with functions** (§6.4): `#name(...)` always selects the macro, and a bare `name(...)` resolves the functions only.
- **Declaration Order:** Because a macro's result type is inferred from the value it produces, the compiler evaluates a macro the moment it checks the invocation site. Every definition a macro's body touches (functions, types, other macros) must therefore appear **earlier in program order** than the invocation. (`#` expressions calling only regular functions are exempt: their signatures are known statically, so evaluation waits until checking completes and forward references work.)
- **Value Position:** A macro invoked in value position (`const x = #m(1)`) takes the type of the value its body produced. Legal results are plain values: primitives, bools, strings and slices, fixed arrays, and named struct or enum values. A `#Type` result or a pointer in value position is a compile-time error (§4.4, §7.2).
- **Macro Bodies:** A macro body is not statically type-checked; it executes in the compile-time interpreter, where calls resolve by name and arity. Faults inside a macro body are compile-time errors at the invocation site.
- **Type Synthesis Examples:**

```alloy
type T = #readTypeFromJson("types/T.json");
type P = #if (DEVELOPMENT) yield #struct { id: u32 } else yield #readTypeFromJson("types/P.json");
```

### 7.4 Built-in Macros

The compiler provides a small set of built-in macros, declared — but not implemented — in the standard library module `std::macros` (§7.3 declaration-only macros), so they are discoverable like any other definition. Using one requires `import std::macros;`. Like all macros they are invoked with a leading `#`.

| Macro             | Signature                | Semantics                                                                                                                                                                                                                                                                                                                        |
| ----------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type_of`         | `(value) -> #Type`       | The `#Type` of the argument expression's type (§4.4).                                                                                                                                                                                                                                                                            |
| `struct_type`     | `() -> #Type`            | A fresh, empty struct `#Type`, for synthesising a type.                                                                                                                                                                                                                                                                          |
| `enum_type`       | `() -> #Type`            | A fresh, empty enum `#Type`.                                                                                                                                                                                                                                                                                                     |
| `implementers_of` | `(target) -> &[#Type]`   | Every type in the merged unit implementing the interface `target`.                                                                                                                                                                                                                                                               |
| `name_of`         | `(value) -> &[u8]`       | The variant name of an enum value, as a string.                                                                                                                                                                                                                                                                                  |
| `read_file`       | `(path: &[u8]) -> &[u8]` | The bytes of a project file, read at compile time as static data. The path resolves against the entry module's directory and must not escape it (§7.2); a missing, absolute, or escaping path is a compile-time error. Distinct from `std::io`'s runtime `read_file` — macro invocation (`#read_file`) always selects the macro. |

A type may also be reflected directly by prefixing its name with `#` (`#u32`, `#Packet`) — see §4.4.

**`implementers_of` is whole-world** (§6.4): because every module — including every library module — merges before checking, the returned array covers the entire program regardless of declaration order or library visibility: a library-internal type implementing an exported interface is included. Elements arrive in module order, then declaration order, so results are deterministic. Generic types are excluded (they reflect only as instances, §4.4). The array is a compile-time value: consume it inside the `#` expression (`#implementers_of(Shape).length()`, `#implementers_of(Shape)[0].name()` — the `#` binds the whole postfix chain, §3.1) — a `#Type` itself cannot be retained into runtime (§7.2). Types synthesised by comptime (`type T : I = #...`) are enumerated like any other declaration, but their member descriptions are only complete once their own `#` initialiser has evaluated (declaration order, §7.3).

---

_End of specification._
