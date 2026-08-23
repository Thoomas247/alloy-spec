# Alloy Language — Formal Language Specification

**Status:** draft · **Revision:** 2026-08-23 · **Source of truth for:** the `alloyc` compiler implementations that consume this repository as a submodule.

Code samples and asides explain; they do not define. Where an example and a rule disagree, the rule wins.

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

One program using the pieces the rest of this document defines one at a time: imports (§6.4), an interface and a type that conforms to it (§6.2), an extension function (§5.5), a standard library container (§6.1a), a `for` loop over a custom iterable (§5.3), and explicit borrowing of reference results (§5.2).

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
    out.append(&p.name());        // reference result: borrowed explicitly
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

    for (points) |&p| {
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

Each `String` built in the loop owns its buffer and is freed at the end of the iteration that built it (§5.2). Nothing here frees memory by hand.

---

## 2. Lexical Grammar

### 2.1 Character Set

Source files are UTF-8. Identifiers and string literals may contain any Unicode character.

### 2.2 Whitespace & Comments

```
whitespace     ::= ' ' | '\t' | '\n' | '\r'
line_comment   ::= '//' <any char except '\n'>*
block_comment  ::= '/*' ( block_comment | <any sequence not containing '/*' or '*/'> )* '*/'
```

Whitespace and comments separate tokens and are otherwise ignored. Newlines are plain whitespace; statements end at `;` (§3.1). A line comment ends at the newline.

Block comments nest. A block comment ends only when every block opened inside it is closed. An unclosed block comment is a compile-time error.

### 2.3 Identifiers

```
identifier     ::= ( letter | '_' ) ( letter | digit | '_' )*
letter         ::= [a-zA-Z] | <any non-ASCII UTF-8 sequence>
digit          ::= [0-9]
```

An identifier that spells a keyword (§2.4) is that keyword and cannot name anything.

Any non-ASCII UTF-8 sequence is allowed in an identifier, at the start and after it. Narrowing this to the UAX #31 character categories is planned.

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

Every symbolic token:

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

When several tokens share a prefix, the longest one wins.

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

Integer literals come in decimal, hex (`0x`), binary (`0b`), and octal (`0o`).

A float literal needs at least one digit after the dot, so `1.` is not a literal and `1.foo()` is member access on an integer literal. Scientific notation uses `e` / `E` (`6.02e23`, `1e-9`, `2.5E+3`); the exponent is decimal and may carry a sign.

String and character literals stay on one line. A raw newline inside one is a compile-time error (reported as an unterminated literal); write `\n` instead. An escape outside the list below is a compile-time error.

#### Escape Sequences

- `\n` (line feed), `\r` (carriage return), `\t` (tab), `\0` (null byte)
- `\\` (backslash), `\"` (double quote), `\'` (single quote)
- `\xHH` (one byte, hex)
- `\u{HHHH}` (Unicode scalar value)

#### String Literal Typing

A string literal has type `&[u8]`: a slice viewing the literal's bytes in static program memory, valid for the whole program run. A string literal never allocates. An owned heap copy is written out (`new "text"`, type `*[u8]`).

#### Character Literal Typing

A character literal's type is the unsigned integer that fits its bytes:

- Single quotes hold one character, or a run of characters up to 8 bytes.
- A 1-byte ASCII character (`'a'`) is a `u8`.
- An 8-byte packed sequence (`'abcdefgh'`) is a `u64`.
- A multi-byte Unicode character takes the smallest unsigned integer wide enough to hold its bytes (usually `u32`).

---

## 3. Syntactic Grammar

### 3.1 Full EBNF

**Notation.** `=` defines a rule and `;` ends it, `|` separates alternatives, `{ x }` is zero or more `x`, `[ x ]` is optional, `"..."` is a literal token, `(* ... *)` is a comment, and `<...>` states in words what the grammar cannot.

**Trailing commas.** Every comma-separated list below may end with a comma before its closing delimiter. The rules leave it out to stay readable.

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
                | expr_stmt ;

var_def         = ( "var" | "const" ) identifier [ ":" type ] "=" expression terminator ;
stmt_block      = "{" { statement } "}" ;
break_stmt      = "break" [ expression ] terminator ;
yield_stmt      = "yield" expression terminator ;
return_stmt     = "return" [ expression ] terminator ;
expr_stmt       = expression ( assign_op expression terminator | terminator ) ;

terminator      = ";" ;   (* the block statements above carry none *)

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
anon_struct_init  = "{" member_init { "," member_init } "}" ;   (* at least one member *)
member_init       = "." identifier "=" expression ;

lambda_expr       = [ "|" capture { "," capture } "|" ] function ;
capture           = [ "&" [ "var" ] | "move" ] identifier ;

if_expr     = "if" "(" expression ")" if_branch [ "else" if_branch ] ;
if_branch   = stmt_block | expression ;   (* bare expression only in value position *)

for_expr    = "for" "(" expression { "," expression } ")"
              [ "|" capture { "," capture } "|" ]
              stmt_block [ "else" stmt_block ] ;

while_expr  = "while" "(" expression ")" stmt_block [ "else" stmt_block ] ;

match_expr  = "match" "(" expression ")" "{"
              { ( expression | "else" ) [ "|" capture "|" ] match_arm }
              "}" [ "else" stmt_block ] ;
match_arm   = stmt_block | expression terminator ;   (* bare expression only in value position *)

comptime_expr   = "#" ( postfix_expr | struct_type | enum_type ) ;
```

**Comptime prefix.** `#` marks an expression for compile-time evaluation (§7). It binds like a postfix expression: `#f(x)` is `#(f(x))`, while `#a + b` is `(#a) + b` — parenthesise to mark the whole thing. `#` also prefixes an inline `struct { … }` or `enum { … }` to give that layout's `#Type` (§4.4).

**Braces are required.** The body of an `if`, an `else`, a `while`, a `for`, and every `match` arm is a block. There is no brace-less body: `if (c) doA();` is a syntax error, written `if (c) { doA(); }` instead. The one place a body may be something else is the value form of `if` below, whose branches may be bare expressions.

**Statement termination.** Every statement and declaration ends with `;`. The exception is the **block statements** — a `{ … }` block, and an `if`, `while`, `for`, or `match` in statement position — which end at their closing `}` and take no `;`:

```alloy
if (x) { doA(); } else { doB(); }        // statement: no ';'
var v = if (x) { yield a; } else { yield b; };   // value: the 'var' still ends with ';'
var w = if (x) a else b;                 // value form with bare branches
```

Using one of those constructs as a **value** changes nothing about the enclosing statement: the `var`, `return`, argument, or assignment around it ends with its own `;` as always. Newlines are whitespace, so a statement may span lines and several may share one. There is no empty statement, so a `;` that terminates nothing is a compile-time error. A statement starting with `if`, `while`, `for`, `match`, or `{` always parses as the statement form, never as an expression statement.

**`if` as a value.** An `if` in value position has two branch forms, and both branches must be present:

- **Bare expressions** — `if (c) a else b` yields the chosen branch's value implicitly. No `yield`, and this form is valid in value position only.
- **Blocks** — `if (c) { yield a; } else { yield b; }` yields through `yield` (§5.3).

The two mix freely (`if (c) a else { yield b; }`), and an `else` may carry another `if` for a chain (`if (a) x else if (b) y else z`).

**`match` arms take the same two forms.** An arm body is a block, or — in value position — a bare expression that yields implicitly. A bare-expression arm ends with `;`, which is what stops the next pattern from being read as more of the arm's expression: without it, `::A x ::B` would parse as the path `x::B` (§5.3).

**Array types.** `[T : N]` is the fixed array, and the count is any expression the compiler can evaluate at compile time (§4.2). `[T]` is not a type on its own: it appears only under `&` or `*` (`&[T]`, `*[T]`).

**Array-fill count.** The count in `[value : count]` is any expression, but a stack array `[T : N]` needs a compile-time one — checked after parsing, not by the grammar. A runtime count is only valid for `new [value : count]` (§5.2).

**Empty array literal.** `[]` is the empty array: null data pointer, zero length, safe because it is never read. It has no elements to infer from, so it is valid only where a slice `&[T]` is expected, taking that element type. Anywhere else it is a compile-time error, and fixed arrays still need at least one element.

**Range generators.** `[start..end]` is the array of integers from `start` (inclusive) to `end` (exclusive), so `[0..5]` is `[0, 1, 2, 3, 4]`; `[..end]` starts at `0`. Both bounds are integers of one type (§4.3; untyped literals default to `i32`), and a literal `end` below a literal `start` is a compile-time error. Literal bounds give a stack array `[T : end - start]`, runtime bounds need `new` (giving `*[T]`) — except as a `for` subject, where no array is built and runtime bounds are always fine (§5.3).

**Subslicing.** `arr[start..end]` borrows a slice (`&[T]`) over elements `start` to `end` in place, with no copy; `arr[..end]` starts at `0`. The subject may be any array form, and the view is mutable when the subject is. Bounds must satisfy `start <= end <= length`; breaking that is a runtime fault in checked builds. Like any reference, the view dangles once the subject is dropped, moved, or reallocated.

**Disambiguation.** Two forms need lookahead, and an implementation **must** resolve them this way:

- A `<` after a name opens a **generic argument list** when the tokens up to the matching `>` read as a comma-separated list of types and that `>` is immediately followed by `(` or `{`. So `f<a, b>(c)` is a generic call and `Vector<u8> { … }` a generic struct literal. Anywhere else `<` is the less-than operator: in `f(a < b, c > d)` the `>` is followed by `d`, so both are comparisons. Where the two readings are both possible the generic one **wins**, and a comparison written that way needs parentheses: `a < b > (c)` is the generic call `a<b>(c)`, and the comparison is `(a < b) > (c)`.
- A `{` in **statement position** starts an anonymous struct literal when the next token is `.`, and a block otherwise — one token of lookahead, since every member initializer begins with `.` (`{ .x = 1 };` is a literal, `{ x = 1; }` a block). There is no empty anonymous struct literal, so `{}` is always an empty block. The same lookahead settles an `if` branch, where a block is also possible: `{` there starts a block unless the next token is `.`. Everywhere else in value position `{` starts a struct literal.

**`is` capture versus bitwise OR.** A `|` right after an `is` test's type always opens a capture, never bitwise OR, and the capture binds tighter than any binary operator: `a is ::Some |b| && c` is `(a is ::Some |b|) && c`. Bitwise OR over an `is` result needs parentheses: `(a is ::Some) | b`.

**Capture typing.** A capture writes its modifier **before** the name, and there are only four forms:

- `|x|` — a deep copy of the value (the default)
- `|&x|` — an immutable reference to the value in place
- `|&var x|` — a mutable reference; the subject must be mutable
- `|move x|` — an owning capture: it moves the pointer into `x` exactly like `move` (§5.2), leaving the source — the enum subject of an `is` test or `match`, or the captured variable of a lambda — moved-from. Valid only on a pointer value (`*T` / `*var T` / `*[T]`), since only pointers move, and the subject must be mutable.

A capture carries no type annotation: the type always comes from the subject. This is the only capture syntax, and it is identical at every capture site: lambda capture lists, `is` tests (§4.2), `for`, and `match` arms. An interface-object capture must state its form — `|&x|`, `|&var x|`, or `|move x|` — because an erased value cannot be copied; a plain `|x|` there is a compile-time error (§4.2).

### 3.2 Operator Precedence & Associativity

Higher number binds tighter. All binary operators are **left-associative**.

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

The `expression` rule is flat; this table alone fixes the grouping. Unary prefixes (`-`, `~`, `!`, `&`, `new`, `move`) bind tighter than every binary operator and are right-associative. Postfix operators (call `()`, generic call `<>()`, member `.`, index `[]`) bind tighter still. The casts (`is`, `as`, `to`, §4.5) sit between: looser than the unary prefixes, tighter than every binary operator.

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

`true` and `false` are keywords. A character literal takes the primitive that fits its bytes (§2.6).

### 4.2 Composite & Derived Types

| Syntax                         | Kind             | Notes                                                             |
| ------------------------------ | ---------------- | ----------------------------------------------------------------- |
| `struct { f: T, ... }`         | Struct           | Members in declaration order                                      |
| `enum { A, B: T, ... }`        | Enum (sum type)  | Variants, each with an optional payload                           |
| `[T : N]` (N > 0)              | Fixed array      | Compile-time size; stack or inline                                |
| `&[T]`                         | Slice            | Non-owning view: raw pointer + `u64` length                       |
| `*[T]`                         | Heap array       | Owned heap block, length fixed at allocation                      |
| `(T1, T2) -> R`                | Function type    | First-class function value                                        |
| `type X = BaseType`            | Named type       | Distinct by name; assignable down its own chain (§4.3 rule 4)     |
| `*T` / `*var T`                | Pointer          | Owned heap instance, immutable / mutable                          |
| `&T` / `&var T`                | Reference        | Non-owning borrow, immutable / mutable                            |
| `&I` / `*I` (`I` an interface) | Interface object | Data pointer + the concrete type's identity (§6.2)                |

An **interface used as a type** — only behind an indirection (`&I`, `&var I`, `*I`, `*var I`) — is an _interface object_: the address of a value plus the identity of its concrete type, which calls dispatch through at runtime (§6.2). A value of type `T` converts to an interface object of `I` exactly when `T` declares `I` as a marker (`type T : I = ...`). Going back down to the concrete type uses the same two constructs as enum discrimination, `match` and `is`.

**By `match`, one arm per concrete type.** The arms are concrete type names, and each capture binds the concrete value. The capture states its own form (§3.1), and a copying `|c|` is a compile-time error on an interface object: `|&c|` and `|&var c|` borrow the concrete value in place, giving `&Circle` / `&var Circle`, and `|move c|` takes ownership of it. The subject has to allow the form asked for — `|&var c|` needs a `&var I` or `*var I` subject, and `|move c|` an owning `*I` / `*var I` one, which is then moved-from after the `match` (§3.1):

```alloy
match (shape) {            // shape: &Shape
    Circle |&c| { /* c: &Circle */ }
    Square |&s| { /* s: &Square */ }
    else     { /* unhandled concrete type */ }
}
```

**By `is`, a single test.** `x is Type` is a boolean expression, and an attached capture also produces the downcast value, in the form the capture asks for:

```alloy
if (shape is Circle |&c|) { /* c: &Circle, only when shape's concrete is Circle */ }
```

A capture is allowed only where a successful test guards every use: on a direct `&&` conjunct of an `if` or `while` condition. Each conjunct may carry one, and each binding is visible to the conjuncts after it and to the branch body:

```alloy
if (cursor.peek() is ::Some |first| && first == '/' && cursor.peek(1) is ::Some |second|) {
    // first and second both bound here
}
while (cursor.next() is ::Some |token|) { /* token re-binds each iteration */ }
```

A capture under `||`, under `!`, or outside an `if` / `while` condition is a compile-time error, since a failed test could still reach its uses. In a `while` the captures re-bind every iteration and scope over the body; in an `if` they scope over the then-branch only.

`is` takes two kinds of left operand. On an **interface object** the right operand is a concrete type and the capture binds the downcast value, as above, always with an explicit `&` / `&var` / `move`. On an **enum value** the right operand is a variant of that enum, `is` tests whether the value holds it, and the capture binds that variant's **payload** — so it is only allowed on variants that carry one:

```alloy
type SomeEnum = enum { ValueA: T, ValueB };
var val: SomeEnum = SomeEnum::ValueA(t);

if (val is SomeEnum::ValueA |&a|) { }      // a borrows the payload in place
if (val is SomeEnum::ValueA |a|) { }       // a is a copy (default)
```

An owning capture (`|move a|`, §3.1) takes the payload **out** of the enum. On a match the payload moves into the capture, the rest of the enum value is dropped, and the subject is cleared, exactly like `move` (§5.2); on a miss the subject is freed by its normal scope-end drop. Either way the subject is **moved-from after the construct**, and using it again is a use-after-move error. The payload must itself be a pointer, since only pointers move (§5.2):

```alloy
type Holder = enum { Boxed: *T, Empty };
var h: Holder = Holder::Boxed(new T {});

if (h is Holder::Boxed |move p|) { /* p owns the payload allocation */ }
// h is moved-from here, whether or not the branch ran
```

#### Implied enum variants (`::Variant`)

A variant may drop its enum name and be written `::Variant`, letting the compiler infer the enum. This works wherever a variant is named: construction (`::Some(x)`, `::None`), `match` arms, and `is` targets. A bare variant name without `::` is never valid.

Inference needs **exactly one candidate**, found in two steps:

1. A contextual type — a declared variable type, an expected payload or return type, a `match` or `is` subject — that is an enum with that variant. As a call argument the context is each overload candidate's parameter type in turn, so a candidate that cannot resolve the variant is simply not viable and `::X` takes part in overload selection like any argument (§4.6).
2. Otherwise every visible enum is searched. Exactly one with that variant wins; none or several is a compile-time error, fixed by writing the enum name.

```alloy
var maybe: Option<u32> = ::Some(7);   // context: the declared type
match (state) {
    ::Idle { }                        // context: the match subject
    ::Busy |load| { }
}
if (state is ::Busy |load|) { }       // context: the 'is' subject
```

Generic enums infer their type parameters by the same unification as named construction (§4.7).

### 4.3 Type Compatibility & Coercion Rules

1. **Identity** — identical types are compatible.
2. **Untyped integer literal** — compatible with any numeric primitive except `bool`, and with any named type whose underlying chain reaches one. Radix makes no difference, and with no context the literal becomes `i32`. An array literal of untyped elements takes a contextual element type (`var a: [u8 : 3] = [1, 2, 3]`).
3. **Untyped float literal** — compatible with `f32` and `f64`; `f32` with no context.
4. **Named type transparency (one way)** — a named type is compatible with everything **below it in its own chain**: its underlying type, that type's underlying type, and so on down to the base, whether the links along the way are named or not. So with `type Meters = f32;` and `type Distance = Meters;`, a `Distance` is compatible with `Meters` and with `f32`. The reverse never holds: nothing converts implicitly **up** a chain, so an `f32` is not a `Meters`. Two named types that merely share a base are therefore incompatible in both directions — each converts down to the base, neither converts up into the other — as are two named types of identical layout on separate chains. Converting up a chain, or across to another, is `x as T` (§4.5).
5. **Numeric widening** — a numeric primitive is compatible with a wider one of the same class: unsigned→unsigned, signed→signed, float→float (`f32`→`f64`, the reverse needs `to`). Crossing classes needs `x to T` (§4.5).
6. **Structural struct compatibility** — a target expecting an **anonymous** layout (`struct { a: u8, b: f32 }`) accepts any value, named or anonymous, whose fields match exactly: same names and types in the same order, nested layouts compared the same way. Extra or missing fields never coerce, and named struct types are otherwise distinct.
7. **Structural enum compatibility** — an inline `enum { ... }` is compatible with any enum whose variant list matches exactly: same names in the same order with the same payload types. Two named enums stay distinct even with matching shapes; this rule needs at least one inline side.

Inline `struct { ... }` and `enum { ... }` types are allowed **wherever a type is expected**: parameters, return types, variable annotations, struct fields, enum payloads, generic arguments, array elements. Values of an inline enum are built with `::Variant` (§4.2), since the type has no name to qualify with.

### 4.4 Compile-Time Special Types (`#Type`)

A `#Type` is a first-class, mutable description of a type — a struct or enum layout, a primitive, or an interface — that compile-time code can read and rebuild. It exists **only during compile-time evaluation**; keeping one in a runtime declaration or variable is a compile-time error.

A `#Type` comes from one of three places:

- **`#T`** — a type prefixed with `#`, either named (`#u32`, `#Packet`) or an inline layout written in place (`#struct { id: u32 }`, `#enum { A, B: u8 }`). `#T.member_names()` reflects on `T` directly.
- **Built-in macros** — `#type_of(expr)`, `#struct_type()`, `#enum_type()`, and `#implementers_of(I)`, specified in §7.4.
- **`#void`** — "no payload". Passed as the member type to `add_member` on an enum `#Type` it makes a payload-less variant, and reflected enums report theirs the same way. `void` is not a value type: `#void` in a runtime position is an error.

All `#Type` methods run at compile time and are called with dot syntax.

| Method                 | Signature                      | Semantics                                                                                                                                                                                                                                              |
| ---------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `is_struct`            | `() -> bool`                   | True iff the type is a struct.                                                                                                                                                                                                                         |
| `is_enum`              | `() -> bool`                   | True iff the type is an enum.                                                                                                                                                                                                                          |
| `is_primitive`         | `() -> bool`                   | True iff the type is a built-in primitive.                                                                                                                                                                                                             |
| `is_interface`         | `() -> bool`                   | True iff the type is an interface.                                                                                                                                                                                                                     |
| `implements_interface` | `(other: #Type) -> bool`       | True iff this type implements `other`, by the §6.2 rule: declared markers matched by definition identity, plus `Number` for the primitives. A generic interface matches by definition, whatever its arguments. Synthesised `#Type`s implement nothing. |
| `name`                 | `() -> &[u8]`                  | The type's declared name.                                                                                                                                                                                                                              |
| `equals`               | `(other: #Type) -> bool`       | True iff both denote the same type.                                                                                                                                                                                                                    |
| `add_member`           | `(name: &[u8], member: #Type)` | Appends a struct field or enum variant.                                                                                                                                                                                                                |
| `remove_member`        | `(name: &[u8])`                | Removes the named member.                                                                                                                                                                                                                              |
| `member_names`         | `() -> &[&[u8]]`               | Member names, in declaration order.                                                                                                                                                                                                                    |
| `member_types`         | `() -> &[#Type]`               | Member types, in the same order.                                                                                                                                                                                                                       |

A `#Type` is a **value**: `add_member` and `remove_member` change the `#Type` in hand, not the type it came from. Assigned through `type T = <#Type-valued comptime expression>`, it becomes the new type `T`. A synthesised struct behaves as a structural layout (§4.3 rule 6) and a synthesised enum as an inline enum (§4.3 rule 7), with variants built, matched, and tested through the name (`T::Variant`) like any declared enum.

### 4.5 Casting

Two operators cover what §4.3 does not do implicitly. Both take a type on the right.

| Operator | Name                  | Semantics                                                               |
| -------- | --------------------- | ----------------------------------------------------------------------- |
| `x as T` | Reinterpretation cast | Reads the bytes of `x` as a `T` without changing them. No runtime cost. |
| `x to T` | Conversion cast       | Converts the value to a `T`, producing a new value.                     |

- `as` needs both types to have the same byte width, by the §4.9 layout rules. On a reference (`&S`) it gives a `&T` over the same memory with no copy, and the width rule then applies to the pointees.
- `to` is defined for `Number` types (§6.2) and does the usual numeric conversions: truncation, sign change, float/integer rounding.
- `is` (§4.2) is in the same grammar family but is a runtime test, giving a `bool` and an optional capture.

```alloy
var raw: u32 = 0x3F800000;
var f = raw as f32;         // same bits, read as f32 (1.0)
var n: i64 = -5;
var u = n to u32;           // numeric conversion, changes the value
```

### 4.6 Function Overloading

Functions, free and extension alike, may share a name as long as their parameter type lists differ. Same name and same parameter types is a redeclaration error, as is reusing a function name for a non-function definition (§6.4) — except a **macro**, which may share a name because `#` tells them apart (§7.3). `extern` declarations do not overload: a C symbol name is unique.

At a call site the candidates are every visible function with that name. A candidate is _viable_ when its arity matches and every argument is compatible (§4.3) with the matching parameter.

- One viable candidate: it is called.
- Several: the one matching every argument type **identically**, with no coercion, wins; if there is no such candidate the call is ambiguous, a compile-time error.
- None: a compile-time error listing the candidates.

### 4.7 Generic Type Parameter Inference

A generic function's type parameters bind at each call site:

1. **Explicit arguments** (`make<u64>(7)`) bind left to right.
2. The rest are **inferred by unification**: each declared parameter type is matched against the argument's type, the `self` receiver included, and every unbound parameter binds to the type in its position. With `fn push<T>(self v: &var Vector<T>, item: T)`, `vector.push(x)` binds `T` from the receiver and checks `x` against it.
3. With a contextual expected type, the return type unifies against it **before** the arguments (`var v: Vector<u8> = Vector::empty();`); the arguments then unify against those bindings and may coerce to them.
4. A parameter still unbound is a compile-time error, and so are conflicting bindings (`T` unified with both `u32` and `f32`).

A bound parameter must satisfy its constraint (`<T: Number>`, §6.2). A constraint naming a **generic interface** supplies its arguments (`<T, It: Iterator<T>>`), and those arguments may mention parameters declared to the **left** in the same list, never later ones. The bound type must conform at exactly that instantiation once the call's bindings substitute in: `It: Iterator<T>` with `T = u64` accepts an `: Iterator<u64>` conformer, not an `: Iterator<u8>` one. During overload resolution a candidate whose unification fails, leaves a parameter unbound, or breaks a constraint is not viable.

Building a variant of a generic enum works the same way: in `Option::Some(x)` the parameters bind by unifying the payload type against the argument and against the contextual expected type. A parameter neither source binds is a compile-time error (`cannot infer type parameter 'T'`). As a call argument, the context is each overload candidate's parameter type, like implied variants (§4.2).

A **struct literal** binds a generic type's parameters from the contextual expected type (`var v: Vector<u8> = Vector { ... }`) or **explicitly**, naming the instantiation before the braces (`Vector<u8> { ... }`, §3.1). Explicit arguments bind all parameters left to right and must match the parameter count; they are how to build a generic value where no context exists (`var v = Vector<T> { ... }` inside a generic body). A bare generic literal with neither source is a compile-time error.

### 4.8 Mutability

Bindings are immutable by default:

- **`const`** declares an immutable binding, **`var`** a mutable one.
- **Parameters are immutable.** A function changes caller state only through `&var` / `*var` indirections passed to it.
- **Assignment**, compound included, needs a mutable target: a `var` local, or a location reached through a `&var` / `*var`.
- **`&var x`**, and `|&var x|` / `|move x|` captures, need `x` to be mutable.
- Mutability travels with pointee transparency (§5.2): a field reached through a `&var T` is mutable even when the reference binding is `const`, and through a `&T` it is immutable even on a `var` binding. Direct fields and elements take the binding's mutability.

### 4.9 Data Layout

Struct layout is **C-compatible**: fields in declaration order, each at its natural alignment, padding as in C, total size rounded up to the largest field alignment. This makes `extern` structs work without annotations and gives `as` (§4.5) well-defined widths. An enum is a tag — the smallest unsigned integer that fits the variant count — followed by a payload area sized and aligned for the largest payload, as C would lay out the matching tagged union.

---

## 5. Execution Semantics

### 5.1 Evaluation Order

**Eager evaluation.** Every sub-expression is fully evaluated before its result is used, and call arguments are evaluated **left to right** before the call.

**Short-circuit operators** are the one exception: `&&` evaluates its right operand only when the left was `true`, `||` only when the left was `false`. Every other operator, bitwise `&` and `|` included, evaluates both. Short-circuiting is what makes an inline `is` capture safe to use in a later conjunct (§4.2).

### 5.2 Memory Model & Pointer Assignment

#### Pointee Transparency

A pointer (`*T`, an owned heap object) and a reference (`&T`, a raw pointer in the C sense) are **always used as their pointee**. There is no dereference (`*ptr`) or arrow (`->`) syntax: field access, indexing, operators, arguments, and plain reads all act on the pointed-at value. Reading a pointer or reference therefore gives a **copy of the pointee**, never the address:

```alloy
var p: *T = new T {};
var x = p;              // x is a deep copy of p's pointee, type T
var r: &T = &p;         // r references p's pointee
var y = r;              // y is likewise a deep copy of the pointee
```

**Deep copies and pointer uniqueness.** Every assignment copies its right-hand side **deeply**: if the value owns heap, directly or through members, elements, or payloads, each allocation is duplicated into a fresh one owned by the copy. So **two pointers can never point at the same object** — a `*T` is the unique pointer to, and owner of, its allocation. Only `move` hands an allocation to another binding, and it clears the source. References own nothing and **may alias freely**.

Three operators step outside pointee transparency:

- **`move`** is the **only** operator that reads its operand as an address: `var q: *T = move p` transfers `p`'s pointer into `q` and clears `p` (below). It works on any pointer operand — `*T`, `*var T`, `*[T]` — and gives back that same pointer type.
- **`new <expression>`** evaluates an expression, deep-copies the result into a fresh heap allocation, and gives a pointer: `new 5`, `new T {}`, `new [0 : n]`, `new some_local`.
- **Unary `&`** gives a reference to any value: a local, a field, an element, or a heap value behind a pointer (`&p` references `p`'s pointee). On a heap array it gives a **slice** (`&[T]`) over every element in place — the only non-owning view of a `*[T]`, as subslicing (§3.1) is of a range.

#### Explicit Assignment Rules

- **To a reference (`&T` / `&var T`)** the right-hand side **must** use `&` (`var r: &i32 = &stack_var`). The empty array literal `[]` (§3.1) is the one exception: it is already a slice, with nothing behind it to borrow.
- **To a pointer (`*T` / `*var T` / `*[T]`)** the right-hand side **must** use `new` or `move` (`var p: *i32 = new 5`, `var p2: *i32 = move p`). A bare pointer would copy the pointee, never alias it.
- **A reference is borrowed explicitly.** A reference-typed value — from a call or from a variable, local, parameter, field, or element alike — never flows bare into a **use site**: a binding, an argument, a member initializer, an assignment value, or a `return` / `break` / `yield`. Writing `&x` or `&f()` keeps the borrow visibly; unary `&` on something already reference-typed passes that same borrow through, and is the one case where `&` needs no addressable operand. A bare use means the value instead:
  - a bare `&T` **pierces** to a deep copy of the pointee, matching pointee transparency on reads — so it no longer fits a `&T` parameter;
  - a bare `&[T]` is a compile-time error, since `[T]` has no size: write `&x` / `&f()` to keep the view, or `new x` / `new f()` to copy the array into an owned `*[T]` (the only way to copy it);
  - a bare interface object is a compile-time error too, since an erased value cannot be copied; write `&x`.

  Positions that consume the value in place need no marker and read it directly: a method receiver (`f().length()`), an index or subslice subject, an operand of `new`, a condition, a match or `for` subject.

#### Slices (`&[T]`) versus Heap Arrays (`*[T]`)

A **slice** is a non-owning view over a sequence whose length is not known at compile time — the pointer + `u64` length pair of §4.2. A **heap array** is an owned heap block made by `new`:

```alloy
var arr: *[u32] = new [0 : 120];    // 120 u32 elements, all 0
var n: u64 = 120;
var dyn: *var [u32] = new [0 : n];  // the count may be a runtime expression
```

The count in `new [value : count]` may be a compile-time literal — which also allows a fixed stack array `[T : N]` — or a runtime expression, which always allocates a `*[T]` of `count` elements, each set to `value`.

A pointer to a heap array points straight at the first element, so it drops into C code unchanged. Its length (`arr.length()`) lives in a metadata prefix immediately **before** the data, at a negative offset from the pointer.

#### Ownership, `move`, and Reclaim

Ownership is **structural and automatic**. A value _owns heap_ if it is a `*T` / `*var T` / `*[T]` pointer, a closure (which owns its captured environment), or a struct / array / enum containing an owning member, element, or active payload. References and slices own nothing.

- **Scope-end drop.** When an owning local goes out of scope — at every `return` path and at the fall-through — the runtime drops it, freeing the heap it owns. Dropping a pointer frees its allocation (a `*[T]` releases the malloc base at `user_ptr - 8`); dropping a `*[T]` first drops every element, all of which are initialised; dropping a struct / array / enum drops each owning field, element, or active payload; dropping a closure frees its environment. Recursive owning types, like a `Node` holding a `*Node`, stop at the first null pointer.
- **`move` transfers ownership.** `var q = move p` copies the pointer into `q` and zeroes the source, so after `move p` the binding `p` is null and its scope-end drop does nothing. `move` gives back the operand's own pointer type, so a whole struct is transferred by moving a `*Struct`, or borrowed through `&var`, never by copying the struct.
- **Returning an owned value is explicit.** `return move v` hands the local's allocation to the caller and clears the source. A bare `return v` returns **by value**: like any read, a deep copy of what `v` holds, with the local's own heap freed by its scope-end drop. `break v` and `yield v` work the same way. A value built inside the `return` expression is owned by the caller directly.
- **Pointer parameters take ownership.** A `*T` / `*var T` / `*[T]` parameter says "I take this allocation", so the caller must `move` a pointer in (`take(move p)`) or allocate inline (`take(new T {})`); a pointer is **never** borrowed. The callee owns it, and the parameter drops at the function's scope end like any owning local, unless moved on or returned. To lend a value, pass `&T` / `&var T`.
- **By-value parameters follow assignment.** A non-pointer value is deep-copied into the parameter; references borrow without copying.
- **Free-on-reassign.** Assigning to an owning binding (`buf = new […]`, `obj.field = move p`) drops what it currently owns first, so the old allocation is freed rather than leaked.
- **Integer overflow** is a runtime fault in checked builds and wraps two's-complement in release builds; compile-time evaluation always faults. Division by zero faults in every build.
- **Checked builds** null-check every dereference of a `*T`, so a use-after-move traps (`@llvm.trap`). **Release builds** skip the checks, so a use-after-move dereferences null and the OS faults. The null store in `move` stays in every build: it is the moved-from mark the drop machinery reads, which keeps drops and free-on-reassign single-free after a transfer.
- **Definite use-after-move is a compile-time error.** The compiler tracks moves of bare locals flow-sensitively: after `move x`, or an owning capture `|move x|`, reading `x`, writing `x.field`, moving it again, or capturing it is rejected until a plain `=` rebinds it. Branches merge conservatively — a move survives a merge only when every falling-through path performs it — so a move under a condition, in one branch, in a loop body, or of a field (`move x.inner`) stays a runtime check.

**Growth is manual.** A `*[T]` has a fixed length once allocated; there is no in-place resize or `realloc`. A growable collection is built by hand: allocate a larger `*var [T]` with a runtime-sized `new [value : count]`, copy the elements across, and reassign the owning field, which frees the old buffer. The standard library's `Vector<T>` (`std/vector.alloy`) and `String` (`std/string.alloy`) are written exactly this way; their mutating functions (`push`, `append`, …) take a `&var self` receiver and are called as methods (`vector.push(x)`).

> **Manual-safety caveats.** Alloy has no borrow checker. References are unchecked raw pointers: a `&T` can outlive what it points at — a dropped local, a moved-from or reassigned owner — and dangle. Double frees are impossible by construction, since deep copying plus pointer uniqueness means no two owners share an allocation, at the price of an implicit allocation whenever an owning value is copied; use `move` to transfer or `&` to borrow when a copy is not what you want. Implicit allocation comes **only** from those value copies. A pointer position is always reached through an explicit `new` or `move`, since pointee transparency makes a bare variable read as its value.

---

### 5.3 Control Flow Semantics

**`return [value]`** exits the enclosing function with `value` as its result.

**`break [value]`** exits the **innermost enclosing loop**, passing through any `if` or `match` in between:

```alloy
for (items) |item| {
    if (item.done()) { break; } // exits the 'for'
}
```

With a value, the loop evaluates to that value — the same channel `yield` uses in a value-position loop (below). A `break` outside a loop is a compile-time error.

**`yield value`** produces the value of the **innermost enclosing value-position construct**: an `if`, a `match`, or a `for` / `while` with an `else` clause. A `yield` inside a loop that is not itself value-position passes out through that loop to the nearest value-position `if` or `match`, exiting the loop on the way and dropping its owned locals normally. A `yield` with no value-position construct around it is a compile-time error: a statement-position `if`, `match`, or loop has nothing to receive the value and is transparent to both `break` and `yield`.

#### `if` as a Value

An `if` in value position needs both branches (§3.1). Bare-expression branches yield their value implicitly; block branches yield with `yield`, and every path through such a block must yield:

```alloy
var grade = if (score > 90) "high" else "low";
var level = if (score > 90) { yield "high"; } else { yield "low"; };
```

#### Path Termination

A conservative flow analysis runs over every function body at compile time. A statement **terminates** when control cannot fall out of it: `return`, `break`, and `yield` terminate; a block terminates when any statement in it does; an `if` with an `else` terminates when both branches do; a `match` terminates when every arm does; `while (true)` with no `break` reaching it never completes. Conditions are never assumed, and ordinary loops always count as skippable.

- **Definite return:** a function or lambda with a return type must terminate on every path; falling off the end is a compile-time error.
- **Definite yield:** a bare-expression branch of a value `if` yields by itself and always counts as terminating; every block branch of a value `if` must terminate; a bare-expression arm of a value `match` likewise yields by itself; every block arm of a value `match` must terminate unless an external `else` supplies the fall-through value, in which case that `else` must terminate; the `else` of a value-yielding loop must terminate.

#### Loops (`for` and `while`)

A `for` subject is either **natively for-compatible** — the array forms (`[T : N]`, `&[T]`, `*[T]`) and range generators, which need no interface marker — or a **custom iterable**: a type declaring `Iterable` (a lang item, §6.1a), whose `iterator` function gives a cursor conforming to `Iterator`:

```alloy
// std::iterable (section 6.1a)
pub interface Iterator<T> {
    fn next(self: &var) -> Option<&T>;
}
pub interface Iterable<T, It: Iterator<T>> {
    fn iterator(self: &) -> It;
}

// a conforming container binds 'It' to its own cursor type
type Container<T> : Iterable<T, ContainerCursor<T>> = ...;
type ContainerCursor<T> : Iterator<T> = ...;
fn iterator<T>(self c: &Container<T>) -> ContainerCursor<T> { ... }
fn next<T>(self it: &var ContainerCursor<T>) -> Option<&T> { ... }
```

- **Cursor lowering.** `for (c) |x| { ... }` becomes `it = c.iterator()` then repeated `match (it.next()) { ::Some |x| <body> ::None { break the loop } }`. A cursor-shaped type without the `Iterable` marker is not a `for` subject: the marker is required, and the check verifies `iterator` / `next` against the instantiated signatures (§6.2).
- **Counting-loop lowering.** When the count and element access are directly available — every array form and every range generator — the compiler **must** lower the `for` to a counting loop over an index, never the cursor protocol. A range generator as a subject (`for ([..n]) |i|`) builds no array at all, which is why runtime bounds are valid here without `new`.
- **Element binding** is the same in both lowerings and follows the capture rules (§3.1): a bare `|x|` deep-copies the element, `|&x|` borrows it in place. The cursor hands the loop an `Option<&T>`, so a copying capture reads through that reference (§5.2) and a borrowing capture keeps it.
- **Multi-subject loops.** A `for` may take several subjects with one capture each, in order: `for (a, b) |x, y| { ... }`. They iterate in lockstep, every subject must be iterable, and all must have the same length; a mismatch is a runtime fault in checked builds.
- **`else` needs expression position.** A trailing `else` on a loop is only allowed when the whole loop is used as an expression. Such a loop is value-position, so `yield value` in the body produces the loop's value and `break value` does the same; either ends the loop at once. The `else` runs if and only if the loop finishes without producing a value — the subjects ran out, or the condition failed — and its own value becomes the loop's value, which must match the body's type. An `else` on a statement loop is a compile-time error.

#### Match

```alloy
var x = match (subject) {
    Pattern1 |payload_capture| { yield 10; }
    Pattern2 { yield 20; }
} else {
    yield 30; // external else: the fallback value
};

var y = match (state) {   // bare-expression arms, each ended with ';'
    ::Idle 0;
    ::Busy |load| load;
    else -1;
};
```

- **Subjects:** an enum value, a numeric primitive, a character literal, or a string literal (matched as an array of integers). Enum arms name variants fully (`State::Idle`) or implied (`::Idle`, §4.2).
- **Captures** are valid **only** on enum variants that carry a payload; using one on numbers, characters, or strings is a compile-time error. They follow §3.1: `|a|` deep-copies the payload, `|&a|` and `|&var a|` borrow it in place, and `|move a|` takes a pointer payload out, leaving the subject moved-from after the `match`.
- **Exhaustiveness:** only a `match` used **as an expression** must cover every subject value, since it has to produce one. An enum subject is exhaustive when every variant has an arm, or when an internal `else` arm is present. Numbers, characters, strings, and interface objects have open domains, so in expression position they always need an internal `else`. A non-exhaustive value-yielding `match` is a compile-time error. A statement `match` has no such rule: it may cover any subset and does nothing when no arm matches.
- **Arm bodies:** an arm body is a block, or — in value position only — a bare expression ended with `;` (`::Idle 0;`), which yields that value implicitly. The `;` keeps the next pattern from being read as a continuation of the expression, since `::A x ::B` would otherwise parse as the path `x::B`. A block arm produces the value with `yield value`. A `break` inside an arm targets the enclosing loop, never the match.
- **External `else` block:** the optional `else` after the closing brace is only allowed when the `match` is an expression. It runs if and only if the selected arm finished **without** a `yield` — it is not a fallback for an unmatched subject — and must supply a value of the expected type. An external `else` on a statement `match` is a compile-time error.

---

### 5.4 Lambdas / Closures

```alloy
|&var x, y| (param: T) -> R { body }
```

- The optional capture list names variables from the enclosing scope, each with an optional modifier (`|&var x|`) saying how the outer variable is reached inside the lambda; §3.1's capture rules apply unchanged. A `|move x|` capture moves the variable's pointer into the closure environment, leaving the outer binding moved-from after the lambda expression.
- Capture lists are **value-only**. Type names, including the enclosing function's generic parameters, stay visible in the lambda without being captured.
- The parameter list and optional return type use the same syntax as a function, and the lambda's type is the matching function type `(T) -> R`.
- A lambda with no captures leaves the delimiters out: `(param: T) { ... }`. An empty capture list (`||`) is not valid — a capture list, when written, names at least one capture.
- A named, non-generic function used in value position becomes a function value of its signature's type. A generic function cannot (its parameters are unbound), an overloaded name must resolve to one function, and an `extern` function cannot be a function value.
- Function values have no identity: comparing them with `==` / `!=` is a compile-time error.

### 5.5 Extension Functions

A function whose first parameter is prefixed with `self` is an extension function, called with dot notation:

```alloy
fn add(self v: &Vec3, other: &Vec3) -> Vec3 { ... }   // v.add(other)
```

- The receiver counts as the first argument for overload resolution.
- **Dot-call precedence:** extensions win. When no extension or interface function (§6.2) has the name, `v.f(...)` calls through a function-typed field `f` of `v`.
- `self` may appear only on the **first** parameter.
- A **temporary receiver** — a call result, a fresh construction — may call an extension whose `self` is `&T`, and the temporary lives for the call. A `&var` receiver still needs a mutable place.
- A **pointer `self`** (`*T` / `*var T`) takes ownership like any pointer parameter (§5.2): an owning place must transfer explicitly (`(move p).consume()`), and only a fresh value (`(new T {}).consume()`, a call result) passes bare.
- When `self`'s type is an **interface** (`fn area(self s: &Shape) -> f32`), the extension is that interface function's default implementation (§6.2).

---

## 6. Standard Library & Primitives

### 6.1 Arrays & Built-in Methods

Every array form — `[T : N]`, `&[T]`, `*[T]` — is **natively for-compatible** (§5.3) and implements **no** interface. Nothing in the language implements an interface implicitly; every conformance is declared (`type T : I = ...`, §6.2), with one exception the compiler supplies: the numeric primitives conform to `Number` (§6.1a), an interface with no functions. So a bound like `<C: Iterable<...>>` accepts only declared conformers, never a bare array.

The compiler supplies `.length() -> u64` on arrays directly, since every form already carries its size:

| Array form        | Where the length lives                                        |
| ----------------- | ------------------------------------------------------------- |
| `[T : N]` fixed   | Known at compile time; `.length()` folds to the constant `N`. |
| `&[T]` slice      | The `u64` half of the fat pointer.                            |
| `*[T]` heap array | The metadata prefix before the pointee (§5.2).                |

A custom type that wants a length supplies its own `length` extension; it is part of no interface contract. Reinterpretation and conversion are the `as` / `to` operators (§4.5), not methods.

### 6.1a Compiler-Recognized Declarations (Lang Items)

A few standard library declarations are **recognized by canonical path** to drive syntax sugar. There is **no prelude**: lang items are ordinary Alloy source shipped with the standard library, and nothing is in scope until imported. The compiler's lowering refers to the declaration directly, so an import is needed only to _name_ the item in source.

| Lang item   | Canonical path            | Declaration                                                              | Compiler hook                                                                                                                                                                                                                           |
| ----------- | ------------------------- | ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Option<T>` | `std::option::Option`     | `enum { Some: T, None }`                                                 | `for` over a custom iterable matches on `next()`'s `Option<&T>` (§5.3).                                                                                                                                                                 |
| `Iterable`  | `std::iterable::Iterable` | `interface Iterable<T, It: Iterator<T>> { fn iterator(self: &) -> It; }` | Gates `for` over custom types (§5.3). Arrays need no marker (§6.1).                                                                                                                                                                     |
| `Iterator`  | `std::iterable::Iterator` | `interface Iterator<T> { fn next(self: &var) -> Option<&T>; }`           | The cursor contract behind `for` (§5.3).                                                                                                                                                                                                |
| `Number`    | `std::number::Number`     | `interface Number { }` (no functions)                                    | Satisfied by the numeric primitives with no marker — the only implicit conformance. Bounds generics, backs `to` (§4.5), and unlocks arithmetic and comparison inside a generic body (§6.2).                                             |
| `arguments` | `std::process::arguments` | `fn arguments() -> &[&[u8]]`                                             | The compiler supplies the command line, first element the program's own path. Natively the entry wrapper captures argc/argv at startup; `alloyc run` serves the arguments after the program path. Not available at compile time (§7.2). |

```alloy
import std::option;

var maybe: Option<u32> = Option::Some(42);
match (maybe) {
    Option::Some |v| { /* v : u32 */ }
    Option::None    { /* empty */ }
}
```

A user definition colliding with an imported lang-item name follows the normal redeclaration rules (§6.4); lang items get no naming privileges.

### 6.2 Standard & User-Defined Interfaces

An interface is a named block of function signatures. Every interface function declares its receiver indirection — `self: &`, `self: &var`, `self: *`, or `self: *var` — as its first parameter, with no name and no type: the type is whatever concrete type implements the interface. Later parameters are ordinary. A named type links itself to interfaces in its declaration:

```alloy
interface Serializable {
    fn serialize(self: &, format: u32) -> bool;
}

type Packet : Serializable, Iterable<u64, PacketCursor> = struct {
    id: u32,
    payload: *[u8],
};
```

The standard interfaces are lang items (§6.1a) written in ordinary library source:

| Name       | Satisfied by                                                                                  |
| ---------- | --------------------------------------------------------------------------------------------- |
| `Number`   | `u8` `u16` `u32` `u64` `i8` `i16` `i32` `i64` `f32` `f64`                                     |
| `Iterable` | Types declaring the conformance and providing `iterator()` (§5.3). Never arrays (§6.1).       |
| `Iterator` | Cursor types declaring the conformance and providing `next(self: &var) -> Option<&T>` (§5.3). |

#### The Two Roles of an Interface

1. **Dynamic dispatch.** An interface used as a type, always behind an indirection, is an _interface object_ (§4.2). Any concrete type implementing it converts implicitly, and a call through the object (`handle.do_something()` where `handle: &Shape`) resolves at **runtime** through the carried type identity, matched against the closed world of implementers (§6.4).
2. **Generic constraint.** An interface used as a bound (`fn do<T: I>(...)`) limits the generic to types implementing `I`, and calls resolve **statically** at each instantiation with no runtime dispatch. Inside the body a `T` value exposes the constraint's functions by dot notation; a `T: Number` value also supports the arithmetic and comparison operators (§4.1).

#### Generic Interfaces

An interface may declare type parameters (`interface Iterator<T> { fn next(self: &var) -> Option<&T>; }`), which scope over every signature in it. A parameter may be constrained, and a constraint's arguments may mention parameters declared to its left (`interface Iterable<T, It: Iterator<T>>`, §4.7).

A conformance marker supplies **all** of a generic interface's arguments (`type Vector<T> : Iterable<T, VectorCursor<T>>`), and those arguments may mention the conforming type's own parameters. Verification substitutes them into the declared signatures, so the satisfying extension must match the **instantiated** signature, and each argument must satisfy the interface's own constraints there.

**A generic interface object needs full instantiation.** It forms only with every parameter bound to a concrete type (`&Iterator<u64>`), which pins each signature so one call site has a fixed ABI while the implementer stays unknown until runtime. There is **no partial erasure**: a bare `&Iterator` is a compile-time error, and `&Iterable<u64>` with `It` erased cannot be written at all by the arity rule — an erased implementer-chosen type would give each implementer a differently sized result, which the call site cannot receive without an implicit allocation (banned, §5.2). Dispatch identity is **per instantiation**: a generic implementer conforms once per binding of its own parameters, so `VectorCursor<u64>` and `VectorCursor<u8>` carry distinct identities, derived by unifying the marker against the object's arguments. Hence **no downcasting**: `is` and `match` on a generic interface object are compile-time errors, because runtime identity picks dispatch targets but is not a nameable type test. Inside a generic body a constrained value still resolves statically (role 2); interface objects are only for call sites where the concrete type is genuinely unknown.

#### Default Implementations

An **extension function whose `self` receiver is an interface** is that interface function's default implementation:

```alloy
interface Shape {
    fn area(self: &) -> f32;
    fn name(self: &) -> &[u8];
}

// default implementation of Shape::name, shared by every implementer
fn name(self s: &Shape) -> &[u8] { return "shape"; }
```

- A default makes that interface function optional for implementing types.
- An extension written for a **concrete type** overrides the default for that type; on a concrete value the type-specific extension always wins.
- A **generic** interface's functions cannot have defaults: the receiver would be a generic interface object, and those need full instantiation, so no single receiver type covers every instantiation.

#### Verification

When a type `T` carries markers (`type T : I1, I2 = ...`), the compiler checks the **merged compilation unit** (§6.4). For every function in each interface:

1. A satisfying **extension function** (§5.5) must exist in the merged unit: one belonging to `T`, or a default implementation. Visibility does not matter — satisfaction is closed-world, so a library-internal extension counts like an exported one.
2. It must match the name, the parameter sequence, and the return type, with a generic interface's marker arguments substituted in first and the candidate's own type parameters bound through its receiver (`fn next<E>(self c: &var Cursor<E>)` checks against `Cursor<T>` with `E = T`). Parameters after `self` line up positionally.
3. The receiver indirection is fixed by the **interface**, not the implementer: `fn next(self: &var)` is satisfied by `fn next(self c: &var Cursor<T>)`, never by `fn next(self c: &Cursor<T>)`. Because every implementer agrees on it, an interface object's call sites have one ABI, and the object's own indirection must be at least as permissive as the function's receiver — calling a `self: &var` function through a `&I` object is a compile-time error.
4. With neither a type-specific extension nor a default, verification fails.

---

### 6.3 Extern FFI

External C functions are declared with fixed, concrete signatures. The return arrow is optional; leaving it out declares a C `void` function. `extern` is for porting C libraries — ordinary programs use native Alloy modules that wrap those declarations.

```alloy
extern functionName(param: Type) -> ReturnType;
extern variadicFunc(...) -> &var u8;
extern releaseBuffer(buffer: &var u8);         // C void
```

- **No owning returns:** an `extern` may not return `*T`, `*var T`, or `*[T]`. Those carry Alloy ownership, and the drop machinery (§5.2) would free memory Alloy never allocated. Model a C pointer result as `&T` / `&var T` or as an opaque integer handle — `std::io` types C's `FILE*` as `i64` — and manage its lifetime by hand in the wrapper.
- **Slice decay:** a slice crossing the boundary passes only its **data pointer**, as C expects; the length stays behind. String literal bytes are NUL-terminated in static memory, so a literal passed to C is a valid C string.
- **Variadic promotions:** arguments in a variadic tail follow the C defaults — integers narrower than 32 bits widen to `i32` by their own signedness, `f32` widens to `f64`.

### 6.4 Module System

- **Module paths mirror the filesystem:** `import a::b::c` names `a/b/c.alloy`.
- **Standard library:** ordinary Alloy source shipped with the compiler, not a prelude. The modules are `std::option` (`Option<T>`), `std::number` and `std::iterable` (the constraint interfaces), `std::vector` (`Vector<T>`), `std::string` (`String`), `std::io` (console and file I/O, the library's FFI barrier so programs never touch C directly), `std::process` (`arguments()`), and `std::macros` (the built-in macro declarations, §7.4).
- **`alloyc build` import resolution:** compiling one file to an executable (`alloyc build file.alloy [-o out] [--release] [--emit-llvm]`) resolves each `import a::b::c` to `a/b/c.alloy`, searched in this order: the **importing module's directory**, the **entry module's directory**, the compiler executable's directory, then `$ALLOY_STDLIB`. The first step lets a module reach its siblings without naming the directory — `import token_kind` inside `tokenizer/tokenizer.alloy` finds `tokenizer/token_kind.alloy` — and such a module takes the directory-qualified key (`tokenizer::token_kind`) while its unqualified alias stays the path's last segment. `std::` and `pkg::` imports skip that first step. Resolution never depends on the shell's working directory, so a build means the same thing from anywhere and matches how editor tooling resolves against the open file. A `pkg/` folder of `.alloylib` containers is found next to the entry module. Builds are **checked** by default; `--release` selects the release semantics of §5.2. The backend emits LLVM IR and an external clang (found via `$ALLOY_CLANG`, then `PATH`) links the executable.
- **Debug info:** checked builds embed DWARF — file, function, and statement line/column — so LLDB and GDB set breakpoints, step by statement, and show Alloy names in call stacks. Release builds carry none. On Windows the debug link goes through lld (`/debug:dwarf`); without lld the build links without debug info and says so.
- **Program entry:** execution starts at a zero-parameter `main` in the entry module. An integer result becomes the process exit code, truncated to the platform's width; any other result type, or none, exits with 0.
- **Qualified vs. unqualified access:** an imported name may be written plain (`Vector`) or qualified (`std::vector::Vector`). Every import introduces an alias for qualified use — the explicit `as` name, or the path's last segment (`import pkg::mathx` allows `mathx::twice(...)`). Qualified access goes through the visibility check, so only `pub` / `exp` definitions are reachable that way. Unqualified access sees the requester's **own library** in full — the executable's own modules and `std::` count as one library — plus the `exp` definitions of each library imported **without** an `as` alias. An unaliased library import injects its exports into the unqualified namespace; an aliased one (`import pkg::liba as la`) is reachable only through the alias (`la::Pair`).
- **Qualified functions (constructors):** `fn Vector::empty<T>() -> Vector<T> { ... }` defines a plain free function in the **type's namespace** instead of the module's, called as `Vector::empty()` (or `alias::Vector::empty()`). The qualifier must name a type visible to the defining module, not necessarily a local one. Qualified functions of one type overload among themselves, and the same name may exist as a free function or under other types. On an enum, a qualified name colliding with a variant is a compile-time error, so `Type::Name(...)` stays unambiguous. A qualified function must not declare a `self` receiver: no dot-call, no dispatch, the type is only a namespace.
- **Name collisions:** within one library, a name colliding with an existing definition is a redeclaration error, except that functions overload (§4.6) and a **macro may share a name with functions** — `#name(...)` always calls the macro, a bare `name(...)` never does (§7.3). Different libraries may reuse names internally. A name visible unqualified from **two libraries** — own declaration vs. an injected export, or two injected exports — is a compile-time error at the import, fixed by aliasing one import. Nothing resolves implicitly: no shadowing, no cross-library overload merging. Two imports whose aliases collide are also an error.
- **Merge, then codegen:** every reachable module, library modules included, merges into ONE compilation unit before type checking and code generation. The whole-program stages need it — closed-world dispatch, monomorphization, §4.9 layouts — and only the per-module front-end stages (tokenize, parse) run in parallel. Libraries therefore recompile with each consuming program; the `.alloylib` payload makes that cheap, not skippable.

#### Import Namespaces

- `import a::b` — a **relative** import: the file `a/b.alloy` next to the importing code. Inside a library, relative imports stay in that library's namespace.
- `import std::x` — the **standard library**.
- `import pkg::name[::module]` — a **package**: the compiler looks for `pkg/name.alloylib` in the project first, then (future) the trusted registry. `pkg::name` imports the package's entry module, `pkg::name::module` one of its members.

#### Libraries (`.alloylib`)

- `alloyc lib entry.alloy [-o name.alloylib]` checks the unit standalone — all §4 and §5 rules, flow and move analysis included — and packs the entry module plus every module of its own into a container. `std::` and `pkg::` dependencies are not packed; they stay imports the consuming program resolves, so package dependencies load transitively.
- The container embeds the **complete source**, which is authoritative: the registry requires open source, and precompiled cache sections (future) are stamped with the producing compiler's version and ignored on mismatch. A library therefore never breaks across compiler releases.
- **Export boundary:** `exp` marks a definition as exported. Inside one compilation unit `exp` behaves like `pub`; across a library boundary only `exp` definitions are visible to consumers, while `pub` covers a library's internal structure without leaking it.
- **Interface satisfaction stays closed-world:** whether a type satisfies an interface (§6.2) considers every extension in the merged unit, internal ones included, so dispatch spans the whole program. Visibility governs who may _name_ an extension in a direct call, not what backs dispatch.
- **Comptime re-runs per program** (§7): library comptime and macros are re-evaluated in the consuming program's merged unit, so compile-time reflection sees the final closed world, not just the library's own.

---

## 7. Compile-Time Evaluation & Macros

### 7.1 The Comptime Modifier (`#`)

Any **value-yielding expression** prefixed with `#` — an `#if`, `#while`, `#match`, a call (`#compute(x)`), an identifier, a parenthesised expression — runs at compile time in the compiler's interpreter. It must produce a value; a bare block (`{ … }`) is not a value and cannot be marked. When `#` marks an outer expression, everything nested inside it runs at compile time too.

A comptime expression is then **replaced by its result**: its whole syntax tree is stripped from the runtime program and replaced with the value it produced — a literal, a struct initializer, or a type. Values come out through `yield` (§5.3), or implicitly from a bare-expression `if` (§3.1), so an `#if` picking between two values reads:

```alloy
const a = #if (cond) 50 else 100;
```

**Visible names.** A comptime expression may use literals, any function in the program (including ones defined later in the file), and enclosing `const` locals whose initializers are themselves compile-time evaluable — transitively, so a `const` built from literals, other compile-time constants, and calls over them counts. Runtime state is invisible: referencing a `var` binding, a parameter, or a `const` that depends on runtime state is a compile-time error.

A fault during compile-time evaluation — integer overflow, division by zero, an exhausted evaluation budget — is a compile-time error.

### 7.2 The Pointer Barrier & Sandboxing

Compile-time code runs under strict limits, so host memory and host capabilities cannot leak into the generated binary:

- **Pointer barrier:** a comptime block cannot produce a reference (`&T`), a pointer (`*T`), a heap array (`*[T]`), or a closure that escapes into a runtime variable. Anything crossing from compile time to runtime must be a value, and breaking this is a compile-time error. Slices are the exception: a slice result is **materialized**, deep copied into static program data, so a comptime call producing a string works. A comptime result is always a value or a slice, never an owning pointer — static data has no owner to free it.
- **Sandboxing:** compile-time evaluation reaches the filesystem **only** through `#read_file` (§7.4), and only inside the project root, which is the entry module's directory tree. Every other host capability is unreachable.
- **No FFI:** comptime code cannot call `extern` C functions (§6.3); it runs only safe Alloy code and built-in macros. The `arguments` lang item (§6.1a) is unreachable for the same reason — there is no command line during compilation.

### 7.3 Macros

Macros are compile-time functions for reflection, source introspection, and type generation.

```alloy
macro readTypeFromJson(path: &[u8]) {
    // introspection and type mutation using compile-time features
}
```

- **Signature:** macro parameters are typed, but **the return type is inferred** from the AST or type node the macro produces.
- **Declaration-only macros:** a macro may be declared without a body (`pub macro type_of(value);`), like an interface function, and the compiler supplies the implementation. Its parameters need no type annotations, while a macro **with** a body must type every parameter. Calling a declared macro the compiler does not implement is a compile-time error. The built-in macros (§7.4) are declared this way in `std::macros`.
- **Invocation:** every macro call is prefixed with `#`. Because that distinguishes them, a macro **may share its name with functions** (§6.4): `#name(...)` always picks the macro, a bare `name(...)` only the functions.
- **Declaration order:** since a macro's result type comes from the value it produces, the compiler evaluates a macro as soon as it checks the call, so every definition the body touches must appear **earlier in program order** than the call. `#` expressions that only call regular functions are exempt: their signatures are known statically, so evaluation waits until checking finishes and forward references work.
- **Value position:** a macro called in value position (`const x = #m(1)`) takes the type of the value its body produced. Legal results are plain values: primitives, bools, strings and slices, fixed arrays, and named struct or enum values. A `#Type` or a pointer there is a compile-time error (§4.4, §7.2).
- **Macro bodies** are not statically type-checked. They run in the compile-time interpreter, where calls resolve by name and arity, and a fault inside one is a compile-time error at the call site.

```alloy
type T = #readTypeFromJson("types/T.json");
type P = #if (DEVELOPMENT) #struct { id: u32 } else #readTypeFromJson("types/P.json");
```

### 7.4 Built-in Macros

The compiler provides a few built-in macros, declared but not implemented in `std::macros` (§7.3), so they are discoverable like any other definition. Using one needs `import std::macros;`, and like all macros they are called with `#`.

| Macro             | Signature                | Semantics                                                                                                                                                                                                                                                                                                  |
| ----------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type_of`         | `(value) -> #Type`       | The `#Type` of the argument expression's type (§4.4).                                                                                                                                                                                                                                                      |
| `struct_type`     | `() -> #Type`            | A fresh, empty struct `#Type` to build on.                                                                                                                                                                                                                                                                 |
| `enum_type`       | `() -> #Type`            | A fresh, empty enum `#Type`.                                                                                                                                                                                                                                                                               |
| `implementers_of` | `(target) -> &[#Type]`   | Every type in the merged unit implementing the interface `target`.                                                                                                                                                                                                                                         |
| `name_of`         | `(value) -> &[u8]`       | The variant name of an enum value, as a string.                                                                                                                                                                                                                                                            |
| `read_file`       | `(path: &[u8]) -> &[u8]` | The bytes of a project file, read at compile time as static data. The path resolves against the entry module's directory and may not escape it (§7.2); a missing, absolute, or escaping path is a compile-time error. Distinct from `std::io`'s runtime `read_file` — `#read_file` always picks the macro. |

A type can also be reflected by prefixing its name with `#` (`#u32`, `#Packet`) — see §4.4.

**`implementers_of` is whole-world** (§6.4): because every module merges before checking, the result covers the whole program regardless of declaration order or library visibility, so a library-internal type implementing an exported interface is in it. Elements arrive in module order, then declaration order, so results are deterministic. Generic types are excluded — they reflect only as instances (§4.4). The array is a compile-time value, so use it inside the `#` expression (`#implementers_of(Shape).length()`, `#implementers_of(Shape)[0].name()` — `#` binds the whole postfix chain, §3.1); a `#Type` cannot be kept into runtime (§7.2). Types synthesised by comptime (`type T : I = #...`) are listed like any declaration, but their members are only complete once their own `#` initialiser has run (§7.3).

---

_End of specification._
