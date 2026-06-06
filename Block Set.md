Okay, then the metagrammar is actually provided at the end of the document.

You can skip to the end and come back after you finish watching.

# Block Set

## Lexical / Token Definitions

```Block Set Def
def DIGIT     { "0"; / "1"; / "2"; / "3"; / "4"; / "5"; / "6"; / "7"; / "8"; / "9"; };

def NUMBER    { DIGIT; (nospace; DIGIT;)(N >= 0); };
/* nospace: built-in assertion — matching fails if whitespace is encountered here */

def LETTER    {
      "A"; / "B"; / "C"; / "D"; / "E"; / "F"; / "G"; / "H"; / "I"; / "J";
    / "K"; / "L"; / "M"; / "N"; / "O"; / "P"; / "Q"; / "R"; / "S"; / "T";
    / "U"; / "V"; / "W"; / "X"; / "Y"; / "Z";
    / "a"; / "b"; / "c"; / "d"; / "e"; / "f"; / "g"; / "h"; / "i"; / "j";
    / "k"; / "l"; / "m"; / "n"; / "o"; / "p"; / "q"; / "r"; / "s"; / "t";
    / "u"; / "v"; / "w"; / "x"; / "y"; / "z";
};

def IDENTIFIER {
    (LETTER; / "_";) (N = 1);
    (nospace; (LETTER; / DIGIT; / "_";);) (N >= 0);
} not {
    KEYWORD_SET;  /* --- undefined --- A set of reserved keywords provided by the language implementation, such as "block", "var", "if", "loop", etc. */
};

def COMMENT {
    "//"; ANY_STR; "\n";
    / "/*"; ANY_STR; "*/";
};
/* ANY_STR: --- undefined --- Matches any sequence of characters excluding the terminator. A placeholder; actual behavior is implemented by the lexical layer per line/block comment rules. */

def STRING {
    "\""; (any_char;)(N >= 0); "\"";
};
/* any_char: --- undefined --- Built-in TOKEN; matches any single character (excluding EOF). */
```

---

## Structural Blocks

```Block Set Def
def BODY_BLOCK {
    "{"; STMT (N >= 0); "}";
};

def PAREN_BLOCK {
    "("; STMT (N >= 1); ")";
};

def END {
    ";"; / "end";
};
```

---

## Statement Aggregation

*The entire file is treated as a single `block`.*

*By convention, the `block` named `main` is the main program.*

```Block Set Def
def STMT {
      BLOCK_STMT;
    / BODY_BLOCK;
    / VAR_STMT;
    / ASSIGN_STMT;
    / COPY_STMT;
    / LOOP_STMT;
    / IF_STMT;
    / BREAK_STMT;
    / CONTINUE_STMT;
    / CALL_STMT;
    / CHAIN_STMT;
    / BACK_STMT;
    / PATH_STMT;
    / COMMENT;
};
```

---

## Block & Variable Statements

```Block Set Def
def BLOCK_STMT {
    "block"; IDENTIFIER; BRACKET_PARAMS (N = [1,0]); STMT; END;
};

def BRACKET_PARAMS {
    "["; DECL; (","; DECL;)(N >= 0); "]";
};

def DECL {
    IDENTIFIER; (":"; TYPE(N >= 1);)(N = [1,0]);
};

def TYPE {
    NAME;
};

/*
  TYPE resolves to NAME.
  Some base types (e.g., Int, String) are provided by the system.
  Beyond those, every block's structure is itself a type —
  there is no separate type definition syntax.
*/

def VAR_STMT {
    "var"; ("|"; DECL;)(N >= 0); END;
};
```

A block carries both a **value** and a **body** — which one is used depends on context. `STMT` is the body of the block.

*The `var` statement must appear within the `STMT` of a `block` statement, and is used to declare writable variables.*

*Only variables declared at `BRACKET_PARAMS` serve as positional parameters during functional invocation.*

*A block is also a namespace. If a `block` statement is nested inside another `block` statement, a path relationship is automatically formed.*

---

## Path / Namespace

```Block Set Def
def PATH_STMT {
    NAME; STMT; END;
};

def AT_PATH {
      "@"; PATH_SEGMENT (N = [1,0]);
    / "$"; PATH_SEGMENT (N = [1,0]);
};

def PATH_SEGMENT {
    IDENTIFIER; ("."; IDENTIFIER;)(N >= 0);
};

def NAME {
    AT_PATH; / PATH_SEGMENT; / IDENTIFIER;
};
```

*Inside a `PATH_STMT` or `BLOCK_STMT`, `@` represents the path that the statement's `NAME` has already traversed.*

* `@` refers to the current block (the block in which the statement appears).
* `$` refers to the root block (the implicit outermost block of the file).
  `$` is unaffected by nesting depth or name shadowing.
* `.` also resolves against sibling and parent blocks, climbing upward
  until the name is found. `$` overrides this and starts from the root.
---

## Assignment & Copy

```Block Set Def
def ASSIGN_STMT {
    NAME; "<-"; VALUE; END;
};

def COPY_STMT {
    NAME; "#"; VALUE; END;
};
```

*`#` deep-copies all `var` and value slots from the source, overwriting them into the destination path. The overwrite is complete, including subordinate blocks.*

---

## Invocation

```Block Set Def
def CALL_STMT {
    "call"; NAME; END;
};

def PAREN_CALL {
    NAME; PAREN_BLOCK;
};

def BRACKET_CALL {
    NAME; ARG_LIST;
};

def ARG_LIST {
    "["; (VALUE; (","; VALUE;)(N >= 0);)(N = [1,0]); "]";
};

def BACK_STMT {
    "back"; END;
};

/* ---- Standalone call that discards the return value ---- */
def CHAIN_STMT {
    CHAIN; END;
};

def CHAIN {
    CHAIN_ATOM;
    ("."; IDENTIFIER; (ARG_LIST; / PAREN_BLOCK;)(N = [0,1]);)(N >= 0);
};

def CHAIN_ATOM {
    PAREN_CALL;
    / BRACKET_CALL;
    / NAME;
};
```

*Both `PAREN_CALL` and `BRACKET_CALL` immediately create a copy of the block, return its value after execution (if the position permits), and then discard the copy.*

*The returned is the block itself.*

---

## Control Flow

```Block Set Def
def IF_STMT {
    "if"; VALUE;
        STMT;
    ("elseif"; VALUE;
        STMT;)(N >= 0);
    ("else";
        STMT;)(N = [0,1]);
    END;
};

def LOOP_STMT {
    "loop"; LOOP_LABEL (N = [0,1]); "|"; STMT; END;
};

def BREAK_STMT {
    "break"; (LOOP_LABEL; / NUMBER;)(N = [0,1]);
};

def CONTINUE_STMT {
    "continue"; (LOOP_LABEL; / NUMBER;)(N = [0,1]);
};

def LOOP_LABEL {
    IDENTIFIER;
};
```

*`LOOP_LABEL` is only valid within a `loop` statement and does not share a namespace with `block`.*

---

## VALUE — Expressions

*Precedence from low to high:*

> **OR  →  AND  →  comparison  →  +/-  →  ×÷*/  →  NOT/-  →  atom**

```Block Set Def
def VALUE {
    LOGIC_OR;
};

/* ---- Logical OR ---- */
def LOGIC_OR {
    LOGIC_AND;
    ("OR"; LOGIC_AND;)(N >= 0);
};

/* ---- Logical AND ---- */
def LOGIC_AND {
    COMPARE;
    ("AND"; COMPARE;)(N >= 0);
};

/* ---- Comparison ---- */
def COMPARE {
    SUM;
    (COMPARE_OP; SUM;)(N = [0,1]);
};

def COMPARE_OP {
    ">=?"; / "=?"; / "<=?"; / ">?"; / "<?"; / "!=?";
};

/* ---- Addition / Subtraction ---- */
def SUM {
    TERM;
    (ADD_OP; TERM;)(N >= 0);
};

def ADD_OP {
    "+"; / "-";
};

/* ---- Multiplication / Division ---- */
def TERM {
    UNARY;
    (MUL_OP; UNARY;)(N >= 0);
};

def MUL_OP {
    "×"; / "÷"; / "*"; / "/";
};

/* ---- Unary operations ---- */
def UNARY {
    UNARY_PREFIX; PRIMARY;
};

def UNARY_PREFIX {
    ("NOT"; / "-";)(N = [0,1]);
};

/* ---- Atom ---- */
def PRIMARY {
    NUMBER;
    / STRING;
    / CHAIN;
    / ("("; VALUE; ")";);
};
```

---

## Meta-Grammar (Notation)

*Greedy matching: repetitions default to matching as many times as possible. A lookahead checks the next TOKEN — if the next TOKEN can start the continuation of the current block, greed stops and matching proceeds.*

*The name is Block Set Def.*

```Block Set Def
def grammar {
    def_rule (N >= 0);
};

def ident {
    (alpha; / cjk_char;) (N = 1);
    (alpha; / cjk_char; / digit_seq; / "_";) (N >= 0);
} not {
    "def"; / "not"; / "N"; / "nospace";
};
/*
  alpha:     --- undefined --- Built-in TOKEN; matches a single ASCII letter [A-Za-z].
  cjk_char:  --- undefined --- Built-in TOKEN; matches a single CJK unified ideograph.
  digit_seq: --- undefined --- Built-in TOKEN; matches consecutive digits [0-9]+.
*/

def def_rule {
    "def"; ident; "{"; alt (N = [0,1]); "}";
    ("not"; "{"; alt; "}";) (N = [0,1]);
    ";";
};

// not block: after the preceding match completes, the same TOKEN stream
// is matched against the not block's alt.
// If the not block matches exhaustively, the preceding match is deemed a failure.

def alt {
    seq (N >= 1);
    ("/"; seq (N >= 1);) (N >= 0);
};

// / denotes ordered choice

def seq {
    (
        ident;
        / "\""; any_char; "\"";
        / paren;
    );
    quant (N = [0,1]);
    / "nospace";
    ";";
};

def paren {
    "("; alt (N >= 1); ")";
};

def quant_term {
    "N";
    (
        (">"; / "<"; / ">="; / "<=";); digit_seq;
        / "="; (
            digit_seq;
            / "["; digit_seq (N >= 1); (","; digit_seq;) (N >= 0); "]";
            // brackets here denote a set, not an interval.
        );
    );
};

def quant {
    "("; quant_term; (","; quant_term;) (N >= 0); ")";
};

def comment_rule {
    "/*"; any_char (N >= 0); "*/";
    / "//"; any_char (N >= 0); newline;
};
/* newline: --- undefined --- Built-in TOKEN; matches a newline character. */
```

---

## Undefined TOKEN Summary

| TOKEN | Location | Description |
|---|---|---|
| `KEYWORD_SET` | IDENTIFIER's `not` block | Set of reserved language keywords, provided by the implementation |
| `ANY_STR` | COMMENT | Matches any character sequence excluding the terminator; implemented by the lexical layer |
| `any_char` | STRING, comment_rule | Built-in — matches any single character (excluding EOF) |
| `alpha` | Meta-grammar `ident` | Built-in — matches a single ASCII letter |
| `cjk_char` | Meta-grammar `ident` | Built-in — matches a single CJK character |
| `digit_seq` | Meta-grammar `quant_term`, `ident` | Built-in — matches consecutive digits |
| `newline` | Meta-grammar `comment_rule` | Built-in — matches a newline character |

---

I am not a native English speaker, so some of my expressions may seem strange.

This is not all.

I may not be able to reply quickly — my time online is limited.
