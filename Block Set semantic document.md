# Block Set Semantic Interpretation

## 1. Program Model

- A `.bs` file as a whole is treated as an implicit top-level `block`.
- The `block` named `main` is the entry point.
- `compiler "…";` is a file-level compiler directive (e.g. `compiler "import <studio>";`), whose semantics are implementation-defined.

---

## 2. Blocks and Scope

### 2.1 Block Declaration

```
block name [parameter-list?] { statement* };
```

- Variables declared within a `block` are **globally visible**.
- The parameter list `[x: Type, y]` is optional (0 or 1 occurrence); parameters may carry type annotations or not.

### 2.2 Variable Exposure

Variables are exposed to the outside only through:

| Method | Syntax | Description |
|--------|--------|-------------|
| `var` declaration | `var \| x: Int, y;` | Explicit exposure |
| Bracket parameters | `block Foo[x: Int] { }` | Parameters are automatically exposed |

Variables that have neither a `var` declaration nor serve as block parameters are **anonymous**. An anonymous variable is equivalent to `block anonymous { anonymous <- null; }` from the outside; accessing it via an external path yields `null`.

### 2.3 Shadowing Rules

> Externally exposed variables shadow internal unexposed ones; internally exposed variables shadow external ones.

Exposure status determines priority in name conflict resolution.

### 2.4 Naming Conflict Rules

A block may not share its name with its **parent, sibling, or child** blocks.

---

## 3. Path System

### 3.1 Root Reference `$`

`$` always refers to the root block (the file's top level).

### 3.2 Child Navigation `.`

`a.b`: looks up `b` among `a`'s child blocks.
If not found, attempts the parent block.
If still not found, treats it as an empty child block (value `null`).

### 3.3 Sibling Navigation `..`

`a..b`: looks up `b` among `a`'s sibling blocks.
If not found, attempts the parent block.
If still not found, treats it as a sibling block (value `null`).

### 3.4 Implicit Paths

Within a block body, the implicit prefix for variable references is the block itself. A variable name declared by the block points to itself when used inside the body.

---

## 4. `@` Syntactic Sugar

```
path { … };
```

Every `@` appearing in the body refers to that path. Equivalent to `PATH_STMT` (`path { body };`), expanded at compile time into ordinary path access.

---

## 5. Type System

### 5.1 Types Are Block Structures

There is no separate type definition syntax. **The structure of every block is itself a type.**

```
block Point[x: Int, y: Int] { };
block Vector[x: Int, y: Int] {
    block add[q: Point]: Point {
        add.x <- x + q.x;
        add.y <- y + q.y;
    };
};
```

The colon `:` causes the block after the colon to be run at compile time, then compile-time `#`-copied onto the block before the colon, with checking applied. It determines which fields that block **"carries as a value."**

Primitive types (`Int`, `String`, `Bool`, etc.) are provided by the system.

### 5.2 Semantics of `<-` and `#`

| Operation | Semantics |
|-----------|-----------|
| `target <- source` | **Type-qualified assignment**: the target's type (the set of fields exposed via `var` and `[]`) determines which fields to extract from the source. Extra fields on the source are discarded; missing required fields cause an error. |
| `target # source` | **Full copy**: ignoring type declarations, all fields from the source are copied to the target, including bodies. |

- As long as a variable is undeclared, it is allowed to change type, but **there is no implicit type conversion**. Type incompatibility (`<-` where the target requires fields the source lacks) is a compile-time error.

---

## 6. Statement Overview

| Statement | Syntax | Semantics |
|-----------|--------|-----------|
| `BLOCK_STMT` | `block name [params] body;` | Declares a block |
| `VAR_STMT` | `var \| x: Type, …;` | Exposes variables |
| `ASSIGN_STMT` | `path <- value;` | Type-qualified assignment (see §5.2) |
| `COPY_STMT` | `path # value;` | Full copy (see §5.2) |
| `CALL_STMT` | `call path;` | Invokes the block body and discards the return value; **always valid** |
| `CHAIN_STMT` | `path;` | Bare path invocation; **valid only if the target block has no `[]` parameter declaration**, equivalent to a parameterless call |
| `PATH_STMT` | `path { body };` | Expanded form of `@` syntactic sugar |
| `BACK_STMT` | `back;` | Exits the current block |
| `IF_STMT` | `if value body elseif … else …;` | Conditional branch; condition must be `Bool` |
| `LOOP_STMT` | `loop label? \| body;` | Loop; `\|` is a pure separator with no special semantics |
| `BREAK_STMT` | `break label\|number?;` | Exits a loop |
| `CONTINUE_STMT` | `continue label\|number?;` | Continues to the next loop iteration |

### 6.1 Return Values

`back;` exits the current block. The return value of a block is the block's own value upon exit — assignments to the block itself within the body ultimately become the block's value exposed to the caller.

### 6.2 `call` vs Bare `CHAIN`

- `call a.b.c;` is always legal.
- Bare `a.b.c;` is legal only when `c` has no `[]` parameter declaration. If `c` has parameters, it is illegal.

### 6.3 `break` / `continue` with Numbers

- With a number: exits / continues N levels of loops.
- If the number exceeds the actual nesting depth: **undefined behavior**; the compiler may report an error.
- Labels do not share a namespace with blocks.

---

## 7. Expressions (VALUE)

Precedence, lowest to highest:

> **OR → AND → Comparison → +/- → ×÷\* / → NOT / - → Primary**

| Level | Operators | Associativity |
|-------|-----------|---------------|
| Logical OR | `OR` | Left |
| Logical AND | `AND` | Left |
| Comparison | `>=?` `=?` `<=?` `>?` `<?` `!=?` | At most once |
| Addition/Subtraction | `+` `-` | Left |
| Multiplication/Division | `×` `÷` `*` `/` | Left |
| Unary | `NOT` `-` | Prefix |
| Primary | Number, string, `CHAIN`, `( VALUE )` | — |

### 7.1 Boolean Type

- `true` is true, `false` is false.
- `if` / `loop` conditions **must** be `Bool`; non-boolean values are not accepted; there is no implicit conversion.
- Comparison operators return `Bool`.

### 7.2 Parentheses

`( VALUE )` is an expression parenthesis; inside is a VALUE, not a statement. This is distinct from `PAREN_BLOCK` appearing in CHAIN paths (inside which are statements, used for argument passing).

---

## 8. Control Flow

### 8.1 `if`

```
if condition body
elseif condition body
...
else body
;
```

The condition must be `Bool`. `elseif` may appear multiple times; `else` is optional. Terminator is `;` or `end`.

### 8.2 `loop`

```
loop label? | body;
```

- Label is optional and does not share a namespace with blocks.
- `|` is a pure separator.

---

## 9. Miscellaneous

### 9.1 `END`

`;` and `end` are equivalent.

### 9.2 Comments

- Line comment: `// …` to end of line
- Block comment: `/* … */`

---

### 9.3 Primitive Types

`String` is a string type; no character length is presupposed.
`String[]` is a string type carrying an initial value.
For a variable carrying a string type, `[n]` queries the n-th string (0-indexed).

`Int` is an integer type; no bit width is presupposed.
`Int[]` is an integer type carrying an initial value.

`Bool` is a boolean type; no implementation is presupposed.
`Bool[]` is a boolean type carrying an initial value.

`List` is a list type; no implementation is presupposed.
`List[]` is a list type of a specified element type; no implementation is presupposed.
For a variable carrying a list type, `[n]` returns the n-th item (0-indexed).

`Map` is a dictionary; no implementation is presupposed.

`Number` is an arbitrary floating-point number; no implementation is presupposed.
`Number[].NS[]` specifies the concrete base (radix) and initial value on which it is based.
