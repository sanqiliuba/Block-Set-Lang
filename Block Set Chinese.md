好的，那么元语法实际上在文档末尾提供了。

你可以跳到末尾，看完再回来。

# Block Set

## 词法 / Token 定义

```Block Set Def
def DIGIT     { "0"; / "1"; / "2"; / "3"; / "4"; / "5"; / "6"; / "7"; / "8"; / "9"; };

def NUMBER    { DIGIT; (nospace; DIGIT;)(N >= 0); };
/* nospace：内置断言 — 若此处遇到空白则匹配失败 */

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
    KEYWORD_SET;  /* --- 未定义 --- 由语言实现提供的一组保留关键字，如 "block"、"var"、"if"、"loop" 等 */
};

def COMMENT {
    "//"; ANY_STR; "\n";
    / "/*"; ANY_STR; "*/";
};
/* ANY_STR：--- 未定义 --- 匹配除终止符外的任意字符序列。占位符；实际行为由词法层按行注释/块注释规则实现。 */

def STRING {
    "\""; (any_char;)(N >= 0); "\"";
};
/* any_char：--- 未定义 --- 内置 TOKEN；匹配任意单个字符（EOF 除外）。 */
```

---

## 结构块

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

## 语句聚合

*整个文件被视为一个单一的 `block`。*

*按惯例，名为 `main` 的 `block` 是主程序。*

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

## Block 与变量语句

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
  TYPE 解析为 NAME。
  一些基本类型（如 Int、String）由系统提供。
  在此之外，每个 block 的结构本身就是一个类型 —
  不存在单独的类型定义语法。
*/

def VAR_STMT {
    "var"; ("|"; DECL;)(N >= 0); END;
};
```

一个 block 同时携带**值**和**体**——使用哪一个取决于上下文。`STMT` 是 block 的体。

*`var` 语句必须出现在 `block` 语句的 `STMT` 内，用于声明可写变量。*

*只有在 `BRACKET_PARAMS` 处声明的变量才可作为函数式调用时的位置参数。*

*block 也是一个命名空间。如果一个 `block` 语句嵌套在另一个 `block` 语句内部，路径关系会自动形成。*

---

## 路径 / 命名空间

```Block Set Def
def PATH_STMT {
    NAME; STMT; END;
};

def AT_PATH {
      "@"; ("."; / "..";)(N = [1,0]); PATH_SEGMENT (N = [1,0]);
    / "$"; ("."; / "..";)(N = [1,0]); PATH_SEGMENT (N = [1,0]);
};

def PATH_SEGMENT {
    IDENTIFIER; (("."; / "..";); IDENTIFIER;)(N >= 0);
};

def NAME {
    AT_PATH; / PATH_SEGMENT; / IDENTIFIER;
};
```

*在 `PATH_STMT` 或 `BLOCK_STMT` 内部，`@` 表示该语句的 `NAME` 已遍历的路径。*

* `@` 指向当前 block（该语句所在的 block）。
* `$` 指向根 block（文件的隐式最外层 block）。
  `$` 不受嵌套深度或名称遮蔽的影响。

### `.` / `..` 解析规则

> **`.` 往下找，找不到就往上找同名父。**
> **`..` 往旁边找，找不到就往上找同名父。**
> **父名不与子、兄弟、自己重名，保证兜底时无歧义。**

---

**`.` 在当前节点的可见范围（子节点 + 父节点名）内匹配：**

| 命中 | 行为 |
|---|---|
| 子节点 | 进入该子节点 |
| 父节点名 | 进入父节点 |
| 均未命中 | 进入空块（值 `null`，体 `{}`）。空块**不是**父节点 |

**`..` 在当前节点的可见范围（兄弟节点 + 父节点名）内匹配：**

| 命中 | 行为 |
|---|---|
| 兄弟节点 | 进入该兄弟节点 |
| 父节点名 | 进入父节点 |
| 均未命中 | 进入空块（值 `null`，体 `{}`）。空块**不是**父节点 |

> **约束：** 节点自身、子节点、兄弟节点均不与父节点重名，保证父节点匹配始终无歧义。
---

## 赋值与拷贝

```Block Set Def
def ASSIGN_STMT {
    NAME; "<-"; VALUE; END;
};

def COPY_STMT {
    NAME; "#"; VALUE; END;
};
```

*`#` 从源深度拷贝所有 `var` 和值槽，将它们覆盖写入目标路径。覆盖是完全的，包括所有下级 block。*

---

## 调用

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

/* ---- 丢弃返回值的独立调用 ---- */
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

*`PAREN_CALL` 和 `BRACKET_CALL` 会立即创建 block 的副本，执行后返回其值（若位置允许），然后丢弃副本。*

*返回值是 block 本身。*

---

## 控制流

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

*`LOOP_LABEL` 仅在 `loop` 语句内有效，不与 `block` 共享命名空间。*

---

## VALUE — 表达式

*优先级从低到高：*

> **OR → AND → 比较 → +/- → ×÷*/ → NOT/- → 原子**

```Block Set Def
def VALUE {
    LOGIC_OR;
};

/* ---- 逻辑或 ---- */
def LOGIC_OR {
    LOGIC_AND;
    ("OR"; LOGIC_AND;)(N >= 0);
};

/* ---- 逻辑与 ---- */
def LOGIC_AND {
    COMPARE;
    ("AND"; COMPARE;)(N >= 0);
};

/* ---- 比较 ---- */
def COMPARE {
    SUM;
    (COMPARE_OP; SUM;)(N = [0,1]);
};

def COMPARE_OP {
    ">=?"; / "=?"; / "<=?"; / ">?"; / "<?"; / "!=?";
};

/* ---- 加减 ---- */
def SUM {
    TERM;
    (ADD_OP; TERM;)(N >= 0);
};

def ADD_OP {
    "+"; / "-";
};

/* ---- 乘除 ---- */
def TERM {
    UNARY;
    (MUL_OP; UNARY;)(N >= 0);
};

def MUL_OP {
    "×"; / "÷"; / "*"; / "/";
};

/* ---- 一元运算 ---- */
def UNARY {
    UNARY_PREFIX; PRIMARY;
};

def UNARY_PREFIX {
    ("NOT"; / "-";)(N = [0,1]);
};

/* ---- 原子 ---- */
def PRIMARY {
    NUMBER;
    / STRING;
    / CHAIN;
    / ("("; VALUE; ")";);
};
```

---

## 元语法（记法）

*贪婪匹配：重复默认尽可能多地匹配。向前查看检查下一个 TOKEN——如果下一个 TOKEN 可以开始当前块的延续，则停止贪婪匹配并继续解析。*

*其名称为 Block Set Def。*

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
  alpha：     --- 未定义 --- 内置 TOKEN；匹配单个 ASCII 字母 [A-Za-z]。
  cjk_char：  --- 未定义 --- 内置 TOKEN；匹配单个中日韩统一表意文字。
  digit_seq： --- 未定义 --- 内置 TOKEN；匹配连续数字 [0-9]+。
*/

def def_rule {
    "def"; ident; "{"; alt (N = [0
