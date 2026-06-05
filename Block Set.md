
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

// nospace is an assertion: if a space is matched at this point, the match fails.

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

// Greedy matching: repetitions default to matching as many times as possible.
// A lookahead checks the next TOKEN — if the next TOKEN can start the
// continuation of the current block, greed stops and matching proceeds.
```

---

```Block Set Def
def DIGIT     { "0"; / "1"; / "2"; / "3"; / "4"; / "5"; / "6"; / "7"; / "8"; / "9"; };

def NUMBER    { DIGIT; (nospace; DIGIT;)(N >= 0); };

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
    KEYWORD_SET;
};

def BODY_BLOCK {
    "{"; STMT (N >= 0); "}";
};

def PAREN_BLOCK {
    "("; STMT (N >= 1); ")";
};

def END {
    ";"; / "end";
};

def COMMENT {
    "//"; ANY_STR; "\n";
    / "/*"; ANY_STR; "*/";
};
```

---

# Block Set

> 结果正确
> *Correct result*
>
> 无恶意程序
> *No malicious code.*

## Block

```Block Set Def
def BLOCK_STMT {
    "block"; IDENTIFIER; BRACKET_PARAMS (N = [1,0]); STMT; END;
};

def BRACKET_PARAMS {
    "["; DECL; (","; DECL;)(N >= 0); "]";
};

def DECL {
    IDENTIFIER; TYPE (N = [1,0]); (":"; (","; TYPE;)(N >= 0);)(N = [1,0]);
};

def VAR_STMT {
    "var"; ("|"; DECL;)(N >= 0); END;
};
```

A block carries both a **value** and a **body** — which one is used depends on context. `STMT` is the body of the block.

---

`var` 语句必须出现在 `block` 语句的 `STMT` 内，用于声明可写入的变量。
*The `var` statement must appear within the `STMT` of a `block` statement, and is used to declare writable variables.*

只有在 `BRACKET_PARAMS` 处声明的变量，才是函数式调用时的位置传参。
*Only variables declared at `BRACKET_PARAMS` serve as positional parameters during functional invocation.*

---

一个 block 同时是一个命名空间。若 `block` 语句嵌套在另一个 `block` 语句内部，则自动构成路径关系。
*A block is also a namespace. If a `block` statement is nested inside another `block` statement, a path relationship is automatically formed.*

```Block Set Def
def PATH_STMT {
    NAME; STMT; END;
};

def AT_PATH {
    "@"; PATH_SEGMENT (N = [1,0]);
};

def PATH_SEGMENT {
    IDENTIFIER; ("."; IDENTIFIER;)(N >= 0);
};

def NAME {
    AT_PATH; / PATH_SEGMENT; / IDENTIFIER;
};
```

---

在 `PATH_STMT` 内部，`@` 表示携带该语句 `NAME` 已穿透的路径。
*Inside a `PATH_STMT`, `@` represents the path that the statement's `NAME` has already traversed.*

整个文件被视为一个 `block`。
*The entire file is treated as a single `block`.*

约定，名为 `main` 的 `block` 是主程序。
*By convention, the `block` named `main` is the main program.*

`@` 访问可视为语法糖。
*The `@` access can be regarded as syntactic sugar.*

---

## 调用
## *Invocation*

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
```

---

`PAREN_CALL` 与 `BRACKET_CALL` 都会即时创建一个 block 副本，执行结束后返回其值（若位置允许），随后清除副本。
*Both `PAREN_CALL` and `BRACKET_CALL` immediately create a copy of the block, return its value after execution (if the position permits), and then discard the copy.*

返回的值就是 block 本身。
*The returned value is the block itself.*

---

## 控制流与原语
## *Control Flow and Primitives*

```Block Set Def
def ASSIGN_STMT {
    NAME; "<-"; VALUE; END;
};

def COPY_STMT {
    NAME; "#"; NAME; END;
};
```

---

`#` 深拷贝源的全部 `var` 与值槽，覆盖铺入目标路径。覆盖是完全的，包括从属的 block。
*`#` deep-copies all `var` and value slots from the source, overwriting them into the destination path. The overwrite is complete, including subordinate blocks.*

```Block Set Def
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

---

`LOOP_LABEL` 仅在 loop 语句内有效，不与 block 共享命名空间。
*`LOOP_LABEL` is only valid within a `loop` statement and does not share a namespace with `block`.*

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
```

---

## STATEMENT 聚合
## *STATEMENT Aggregation*

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
    / BACK_STMT;
    / PATH_STMT;
    / DISCARD_CALL;
    / COMMENT;
};

/* ---- 独立调用并丢弃返回值 ---- */
/* ---- Standalone call that discards the return value ---- */
def DISCARD_CALL {
    (PAREN_CALL; / BRACKET_CALL;); END;
};
```

---

## VALUE — 表达式
## *VALUE — Expressions*

优先级从低到高：
*Precedence from low to high:*

> **OR  →  AND  →  比较 (comparison)  →  +/-  →  ×÷*/  →  NOT/-  →  原子 (atom)**

```Block Set Def
def VALUE {
    LOGIC_OR;
};

/* ---- 逻辑或 ---- */
/* ---- Logical OR ---- */
def LOGIC_OR {
    LOGIC_AND;
    ("OR"; LOGIC_AND;)(N >= 0);
};

/* ---- 逻辑与 ---- */
/* ---- Logical AND ---- */
def LOGIC_AND {
    COMPARE;
    ("AND"; COMPARE;)(N >= 0);
};

/* ---- 比较 ---- */
/* ---- Comparison ---- */
def COMPARE {
    SUM;
    (COMPARE_OP; SUM;)(N = [0,1]);
};

def COMPARE_OP {
    ">=?"; / "=?"; / "<=?"; / ">?"; / "<?"; / "!=?";
};

/* ---- 加减 ---- */
/* ---- Addition / Subtraction ---- */
def SUM {
    TERM;
    (ADD_OP; TERM;)(N >= 0);
};

def ADD_OP {
    "+"; / "-";
};

/* ---- 乘除 ---- */
/* ---- Multiplication / Division ---- */
def TERM {
    UNARY;
    (MUL_OP; UNARY;)(N >= 0);
};

def MUL_OP {
    "×"; / "÷"; / "*"; / "/";
};

/* ---- 一元运算 ---- */
/* ---- Unary operations ---- */
def UNARY {
    UNARY_PREFIX; PRIMARY;
};

def UNARY_PREFIX {
    ("NOT"; / "-";)(N = [0,1]);
};

/* ---- 原子 ---- */
/* ---- Atom ---- */
def PRIMARY {
    NUMBER;
    / STRING;
    / PAREN_CALL;
    / BRACKET_CALL;
    / NAME;
    / ("("; VALUE; ")";);
};

/* ---- 字面量 ---- */
/* ---- Literals ---- */
def STRING {
    "\""; (any_char;)(N >= 0); "\"";
};
```

---

详细语义会在其他地方给出。
*The detailed semantics will be provided elsewhere.*

The detailed semantics will be provided elsewhere.

I don't have much time, sorry.
