
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

def BODY_BLOCK_STMT {
    BODY_BLOCK; END;
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
    / BODY_BLOCK_STMT;
};
```

---

## Block 与变量语句

```Block Set Def
def BLOCK_STMT {
    "block"; IDENTIFIER; BRACKET_PARAMS (N = [0,1]);
    (":"; TYPE;)(N = [0,1]);
    BODY_BLOCK; END;
};

def BACK_STMT {
    "back"; END;
};

def BRACKET_PARAMS {
    "["; DECL; (","; DECL;)(N >= 0); "]";
};

def DECL {
    IDENTIFIER; (":"; TYPE(N >= 1);)(N = [1,0]);
};

def TYPE {
    CHAIN;
};

/*
  TYPE 解析为 CHAIN 返回的 block 的 var 与方括号声明的数据结构。
  一些基本类型（如 Int、String）由系统提供。
  在此之外，每个 block 的结构本身就是一个类型 —
  不存在单独的类型定义语法。
  它决定什么通过<-传递。
*/

def VAR_STMT {
    "var"; ("|"; DECL;)(N >= 0); END;
};
```

---

## 赋值与拷贝

```Block Set Def
def ASSIGN_STMT {
    CHAIN; "<-"; BLOCK; END;
};

def COPY_STMT {
    CHAIN; "#"; BLOCK; END;
};
```

---

## 调用

```Block Set Def
def CALL_STMT {
    "call"; BLOCK; END;
};

def ARG_LIST {
    "["; (BLOCK; (","; BLOCK;)(N >= 0);)(N = [1,0]); "]";
};

/* ---- 丢弃返回值的独立调用 ---- */
def CHAIN_STMT {
    CHAIN; END;
};

def CHAIN {
    (IDENTIFIER; / "$"; / "@";); (PAREN_BLOCK; / ARG_LIST)(N = [1,0]);
    (("."; / "..";) IDENTIFIER; (PAREN_BLOCK; / ARG_LIST)(N = [1,0]);)(N >= 0);
};

def BLOCK {
      STMT;
    / VALUE;
    / CHAIN;
    / BODY_BLOCK;
}

def PATH_STMT {
    CHAIN; BODY_BLOCK; END;
};
```

---

## 控制流

```Block Set Def
def IF_STMT {
    "if"; VALUE;
        BLOCK;
    ("elseif"; VALUE;
        BLOCK;)(N >= 0);
    ("else";
        BLOCK;)(N = [0,1]);
    END;
};

def LOOP_STMT {
    "loop"; LOOP_LABEL (N = [0,1]); "|"; BLOCK; END;
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
    "def"; ident; "{"; alt (N = [0,1]); "}";
    ("not"; "{"; alt; "}";) (N = [0,1]);
    ";";
};

// not 块：前一个匹配完成后，同一个 TOKEN 流
// 会与 not 块的 alt 进行匹配。
// 若 not 块完全匹配，则前一个匹配被视为失败。

def alt {
    seq (N >= 1);
    ("/"; seq (N >= 1);) (N >= 0);
};

// / 表示有序选择

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
            // 此处方括号表示集合，而非区间。
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
/* newline：--- 未定义 --- 内置 TOKEN；匹配换行符。 */
```

---

## 未定义 TOKEN 汇总

| TOKEN | 所在位置 | 说明 |
|---|---|---|
| `KEYWORD_SET` | IDENTIFIER 的 `not` 块 | 由实现提供的一组保留语言关键字 |
| `ANY_STR` | COMMENT | 匹配除终止符外的任意字符序列；由词法层实现 |
| `any_char` | STRING、comment_rule | 内置 — 匹配任意单个字符（EOF 除外） |
| `alpha` | 元语法 `ident` | 内置 — 匹配单个 ASCII 字母 |
| `cjk_char` | 元语法 `ident` | 内置 — 匹配单个 CJK 字符 |
| `digit_seq` | 元语法 `quant_term`、`ident` | 内置 — 匹配连续数字 |
| `newline` | 元语法 `comment_rule` | 内置 — 匹配换行符 |

---

这并非全部。

我回复可能不够及时 — 上网时间有限。
