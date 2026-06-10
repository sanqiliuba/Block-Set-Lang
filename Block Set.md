# Block Set

## 语法定义

本语言使用 `Block Set Def` 元语法，是一种 `C风格` 的，类似于 `BNF` 的上下文无关文法。

 `Block Set Def` 的自举，以及规范，可以跳转到文档的查看附录。

本语言使用一个反斜杠 `\` 转义。

```Block Set Def
def end {
    ",";
};

def digital {
    "0"; / "1"; / "2"; / "3"; / "4"; / "5"; / "6"; / "7"; / "8"; / "9";
};

def letter {
      "a"; / "b"; / "c"; / "d"; / "e"; / "f"; / "g";
    / "h"; / "i"; / "j"; / "k"; / "l"; / "m"; / "n";
    / "o"; / "p"; / "q"; / "r"; / "s"; / "t";
    / "u"; / "v"; / "w"; / "x"; / "y"; / "z";
    / "A"; / "B"; / "C"; / "D"; / "E"; / "F"; / "G";
    / "H"; / "I"; / "J"; / "K"; / "L"; / "M"; / "N";
    / "O"; / "P"; / "Q"; / "R"; / "S"; / "T";
    / "U"; / "V"; / "W"; / "X"; / "Y"; / "Z";
};

def number {
    digital(N >= 1);
};

def ui_cjk {
    // --- 即中日韩统一文字。取决于语言实现平台，此处不做该级别定义 ---
};

def any_char {
    // --- 即任意单个字符。取决于语言实现平台，此处不做该级别定义 ---
};

def escape {
    "\\"; any_char;
};

def string {
    "\""; (escape; / any_char;)(N >= 0); "\"";
};

def identifier_base {
    (letter; / ui_cjk; / "_"; );
    (nospace; (letter; / ui_cjk; / digital; / "_"; );)(N >= 0);
} not {
      key_set;
};

def identifier {
      identifier_base;
    / string;
};

def block_block {
    "{"; blockset(N >= 0); ("---"; statement(N >= 0);)(N = [0,1]); "}";
};

def statement_block {
    "{"; statement(N >= 0); "}";
};

def blockset {
    (identifier; / "@";); ":"; (para_list_Statement(N = [0,1]); block;)(N = [0,1]); end;
};

def para_list_Statement {
    "("; ((identifier_base; ":"; block;)(N = [0,1]); end;)(N >= 0); ")";
};

def block {
      block_block;
    / value;
    / array;
};

def statement {
      if_statement;
    / loop_statement;
    / break_statement;
    / continue_statement;
    / blockset_statement;
    / blockset_type_statement;
    / blockset_same_statement;
};

def if_statement {
     "if"; value;
        statement_block;
    ("elseif"; value;
        statement_block;)(N >= 0);
    ("else";
        statement_block;)(N = [0,1]);
    end;
};

def loop_statement {
    "loop"; identifier; statement_block; end;
};

def break_statement {
    "break"; (identifier; / digital;)(N = [0,1]); end;
};

def continue_statement {
    "continue"; (identifier; / digital;)(N = [0,1]); end;
};

def chain {
    ("$"; / "@"; / "fn";); (("~"; / "~";); identifier; para_list(N = [0,1]);)(N >= 0);
};

def para_list {
    "("; (block; (","; block;)(N >= 0);)(N = [0,1]); ")";
};

def blockset_statement {
    (chain; / identifier_base;) "#"; block; end;
};

def blockset_type_statement {
    (chain; / identifier_base;) "<-"; block; end;
};

def blockset_same_statement {
    (chain; / identifier_base;) "="; block; end;
};

def value {
    value_not;
};

def value_not {
    "NOT"(N = [0,1]); value_and_or;
};

def value_and_or {
    value_compare; (("AND"; / "OR";); value_compare;)(N = [0,1]);
};

def value_compare {
    value_equal; ((">?"; / "<?"; / ">=?"; / "<=?";); value_equal;)(N = [0,1]);
};

def value_equal {
    value_add; (("=?"; / "=??"; / "!=?"; / "!=??";); value_add;)(N = [0,1]);
};

def value_add {
    value_mul; (("+"; / "-";); value_mul;)(N >= 0);
};

def value_mul {
    value_base; (("*"; / "×"; / "/"; / "÷";); value_base;)(N >= 0);
};

def value_base {
      chain;
    / "-"(N = [0,1]); number; (nospace; "."; nospace; number;)(N = [0,1]);
    / string;
    / "("; value; ")";
    / identifier_base;
};

def array {
    "["; (block; ("," block;)(N >= 0);)(N = [0,1]); "]";
}

def annotation {
      "//"; any_char(N >= 0); newline;
    / "/*"; any_char(N >= 0); "*/";
};
```

## 语义说明

 `Block Set` 是 `JSON` 的超集，且独立于 `JavaScript` 。

该语言的数据结构就是 `JSON` 对象。

### 可见性

在一个 `block` 内，所有链式访问都是有具体指向的。

如果是 `identifier_base` 则表示局部变量。

其中，在圆括号内进行声明的，视为虚拟变量。

非虚拟变量的局部变量，每次调用开始时都视为 `null` 值的空 `block` 。

 `Block Set` 的数据结构与 `JSON` 是同像的。

 `Block Set` 运行结束后，所有数据就是最终的 `JSON` 输出。

---
未完工
---
