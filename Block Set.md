# Block Set 语言规范

**版本** 0.2.0 | **面向** 编译器/解释器实现者

---

## 目录

1. [词法规范](#1-词法规范)
2. [语法规范](#2-语法规范)
3. [静态语义](#3-静态语义)
4. [运行时语义](#4-运行时语义)
5. [类型系统](#5-类型系统)
6. [内置特殊值](#6-内置特殊值)
7. [JSON 互操作](#7-json-互操作)
8. [标准库](#8-标准库)
9. [附录：Block Set Def](#9-附录block-set-def)

---

## 1. 词法规范

### 1.1 字符集

源码为 Unicode 文本。实现应至少支持 UTF-8 编码。

- `any_char` — 任意单一 Unicode 码点（包括换行符 U+000A、回车 U+000D 等）。
- `newline` — 换行符。接受 `\n`（U+000A）、`\r\n`（U+000D U+000A）、`\r`（U+000D）三种形式，均归一化为一个换行。

### 1.2 空白字符

| 字符 | 码点 | 名称 |
|------|------|------|
| ` ` | U+0020 | 空格 |
| `\t` | U+0009 | 制表 |
| `\r` | U+000D | 回车 |
| `\n` | U+000A | 换行 |

空白字符在 token 之间一般被跳过（不发射给 parser）。但：

- `//` 注释以换行终结，换行在此处有意义。
- 词法分析器在识别数字字面量时，内部不允许出现空白（这是 `nospace` 约束的实现方式——将含小数点和指数的数字作为一个整体 token 读取）。
- 实现可选地保留换行信息以辅助错误报告。

### 1.3 注释

两种形式，均视为空白：

```
// 单行注释：到行尾（换行符）结束
/* 多行注释：到 */ 结束，不支持嵌套 */
```

### 1.4 Token 全集

#### 1.4.1 关键字（保留字，不可作为标识符）

```
if  elseif  else  loop  break  continue
NOT  AND  OR  fn
true  false  null
placeholder  nonExist  derefer
```

#### 1.4.2 标识符

**基本形式** — 以字母、CJK 统一表意文字或下划线开头，后续可含字母、CJK、数字、下划线。内部无空白。

正则近似（具体范围由实现平台的 Unicode 支持决定）：

```
[_a-zA-Z\u4E00-\u9FFF\u3400-\u4DBF][_a-zA-Z0-9\u4E00-\u9FFF\u3400-\u4DBF]*
```

**字符串形式** — 双引号字符串在某些上下文（`blockset` 的键位置、`chain` 的 `~` 之后）也被识别为标识符。这是为了与 JSON 键兼容。

#### 1.4.3 数字字面量

完整格式（一条 token，内部无空白）：

```
数字字面量 ::= '-'? 数字序列 ('.' 数字序列)? (('e' | 'E') ('+' | '-')? 数字序列)?
数字序列   ::= 数字 (数字)*
数字       ::= '0'..'9'
```

**约束**：
- 允许前导零（`007` 合法）。
- 小数点前后至少各有一位数字（`3.` 和 `.14` 非法，`3.14` 合法）。
- 指数部分至少一位数字（`1e` 非法，`1e0` 合法）。
- 整个数字字面量内部无空白（`nospace` 约束）。

**示例**：`0` `123` `-45` `3.14` `-0.5` `1e10` `1.5e-3` `2E+5` `007`

#### 1.4.4 字符串字面量

双引号包裹。内部可含任意字符（含字面换行）。

**转义序列**：

| 序列 | 产出 | 说明 |
|------|------|------|
| `\\` | `\` | 反斜杠自身 |
| `\"` | `"` | 双引号 |
| `\/` | `/` | 正斜杠（JSON 兼容） |
| `\n` | U+000A | 换行 |
| `\t` | U+0009 | 制表 |
| `\r` | U+000D | 回车 |
| `\uXXXX` | 对应码点 | 4 位十六进制 Unicode（大小写均可） |
| `\` + 其他任意字符 c | c | 反斜杠被移除，c 保留 |

注意：`\` 后接非上述特殊字符时，反斜杠静默移除。例如 `\a` → `a`，`\0` → `0`。

#### 1.4.5 标点与运算符

| Token | 名称 |
|-------|------|
| `{` `}` | 花括号 |
| `[` `]` | 方括号 |
| `(` `)` | 圆括号 |
| `,` | 逗号（即 `end`） |
| `:` | 冒号 |
| `---` | 数据区/脚本区分隔符 |
| `.` | 点号（小数点或成员访问） |
| `~` | 链式导航符 |
| `$` | 根引用 |
| `@` | 本块引用 / 隐含键 |
| `#` | 覆盖赋值 |
| `<-` | 追加赋值 |
| `=` | @覆盖赋值 |
| `+` `-` | 加减 |
| `*` `×` `/` `÷` | 乘除 |
| `=?` `=??` | 等于（值/严格） |
| `!=?` `!=??` | 不等于（值/严格） |
| `>?` `<?` `>=?` `<=?` | 大小比较 |

---

## 2. 语法规范

以下使用 **Block Set Def** 元语法（见附录）。终结符以双引号括起。`N` 为重复次数约束。

### 2.1 终结符定义

```block_set_def
def end { ","; };

def digital {
    "0"; / "1"; / "2"; / "3"; / "4";
    / "5"; / "6"; / "7"; / "8"; / "9";
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

def ui_cjk {
    // --- 中日韩统一表意文字。具体范围由实现定义 ---
};

def any_char {
    // --- 任意单一 Unicode 码点 ---
};

def newline {
    // --- 换行符：\n | \r\n | \r ---
};
```

### 2.2 数字与字符串

```block_set_def
def number {
    digital(N >= 1);
    (
        nospace; "."; nospace; digital(N >= 1);
    )(N = [0,1]);
    (
        nospace; ("e"; / "E";);
        nospace; ("+"; / "-";)(N = [0,1]);
        digital(N >= 1);
    )(N = [0,1]);
};

def escape {
    "\\";
    (
          "\\"; / "\""; / "/";
        / "n";  / "t";  / "r";
        / "u"; digital; digital; digital; digital;
        / any_char;
    );
};

def string {
    "\""; (escape; / any_char;)(N >= 0); "\"";
};
```

### 2.3 标识符

```block_set_def
def identifier_base {
    (letter; / ui_cjk; / "_";);
    (nospace; (letter; / ui_cjk; / digital; / "_";);)(N >= 0);
} not {
    key_set;
};

def identifier {
      identifier_base;
    / string;
};
```

`key_set` = `{ if, elseif, else, loop, break, continue, NOT, AND, OR, fn, true, false, null, placeholder, nonExist, derefer }`

### 2.4 块与数组

```block_set_def
def block_block {
    "{"; blockset(N >= 0); ("---"; statement(N >= 0);)(N = [0,1]); "}";
};

def statement_block {
    "{"; statement(N >= 0); "}";
};

def array {
    "["; (block; (","; block;)(N >= 0);)(N = [0,1]); ","(N = [0,1]); "]";
};

def block {
      block_block;
    / value;
    / array;
};
```

**约束**：
- 无 `---` 的 `block_block` 不包含脚本区，仅含数据。
- 尾随逗号在数组和 blockset 列表中均允许。

### 2.5 数据区条目

```block_set_def
def blockset {
    (identifier; / "@";); ":";
    (para_list_Statement(N = [0,1]); block;)(N = [0,1]);
    end;
};
```

**含义**：
- 冒号后若 `para_list_Statement` 和 `block` 均不出现，该键的值为 `null`。
- `para_list_Statement` 和 `block` 作为一个整体可选：要么一起出现，要么都不出现。
- 若仅出现 `block` 而无 `para_list_Statement`，则为无参声明。

```block_set_def
def para_list_Statement {
    "(";
    ((identifier_base; ":"; block;)(N = [0,1]); end;)(N >= 0);
    ")";
};
```

此处 `block` 作为**类型标注**出现。参数可省略类型标注（`(a: number, b)`，`b` 无类型约束）。

### 2.6 链式访问

```block_set_def
def chain {
    ("$"; / "@"; / "fn";);
    ("~"; identifier; para_list(N = [0,1]);)(N >= 0);
};

def para_list {
    "("; (block; (","; block;)(N >= 0);)(N = [0,1]); ")";
};
```

| 前缀 | 含义 |
|------|------|
| `$` | 根 block（最外层 `{}`） |
| `@` | 当前所在 block |
| `fn` | 外部库命名空间 |

`~` 为导航/调用操作符。`$~a~b~c` 表示 `$.a.b.c` 的链式访问。`fn~math~sin(x)` 调用外部库函数。

### 2.7 值表达式

```block_set_def
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
    value_equal;
    ((">?"; / "<?"; / ">=?"; / "<=?";); value_equal;)(N = [0,1]);
};

def value_equal {
    value_add;
    (("=?"; / "=??"; / "!=?"; / "!=??";); value_add;)(N = [0,1]);
};

def value_add {
    value_mul; (("+"; / "-";); value_mul;)(N >= 0);
};

def value_mul {
    value_base; (("*"; / "×"; / "/"; / "÷";); value_base;)(N >= 0);
};

def value_base {
      chain;
    / "-"(N = [0,1]); number;
    / string;
    / "true";  / "false";  / "null";
    / "placeholder"; / "nonExist"; / "derefer";
    / "("; value; ")";
    / identifier_base;
};
```

**运算符优先级（从低到高）**：

| 优先级 | 运算符 | 结合性 |
|--------|--------|--------|
| 1 | `NOT` | 右一元 |
| 2 | `AND` `OR` | 左 |
| 3 | `>?` `<?` `>=?` `<=?` | 左 |
| 4 | `=?` `=??` `!=?` `!=??` | 左 |
| 5 | `+` `-` | 左 |
| 6 | `*` `×` `/` `÷` | 左 |

**比较运算符说明**：

| 运算符 | 含义 |
|--------|------|
| `=?` | 值相等（递归比较内容） |
| `=??` | 严格相等（同一引用/同一类型+同值） |
| `!=?` | 值不等 |
| `!=??` | 严格不等 |

### 2.8 语句

```block_set_def
def statement {
      if_statement;
    / loop_statement;
    / break_statement;
    / continue_statement;
    / blockset_statement;
    / blockset_type_statement;
    / blockset_same_statement;
};
```

#### 2.8.1 条件语句

```block_set_def
def if_statement {
    "if"; value; statement_block;
    ("elseif"; value; statement_block;)(N >= 0);
    ("else"; statement_block;)(N = [0,1]);
    end;
};
```

- `value` 被求值为布尔值。
- 至多一个分支被执行。
- 每个 `statement_block` 内的语句均以 `end`（`,`）分隔。

#### 2.8.2 循环语句

```block_set_def
def loop_statement {
    "loop"; identifier; statement_block; end;
};
```

- `identifier` 为循环标签（必选）。该标签仅在循环体内对 `break`/`continue` 可见。
- 循环为死循环，仅由 `break`/`continue` 控制。

#### 2.8.3 跳转语句

```block_set_def
def break_statement {
    "break"; (identifier; / digital;)(N = [0,1]); end;
};

def continue_statement {
    "continue"; (identifier; / digital;)(N = [0,1]); end;
};
```

- 无操作数：跳出/继续最内层循环。
- `identifier`：跳出/继续指定标签的循环（编译时静态绑定）。
- `digital`（`0`-`9`）：跳出/继续从内向外第 N 层循环。`1` 为最内层。`0` 无意义（实现可报错或忽略）。超过实际嵌套层数时可任意时刻报错。
- 标签查找失败时可任意时刻报错。

#### 2.8.4 赋值语句

```block_set_def
def blockset_statement {
    (chain; / identifier_base;); "#"; block; end;
};

def blockset_type_statement {
    (chain; / identifier_base;); "<-"; block; end;
};

def blockset_same_statement {
    (chain; / identifier_base;); "="; block; end;
};
```

| 运算符 | 语义 |
|--------|------|
| `#` | **覆盖**：用右值整体替换左值 |
| `<-` | **追加**：将右值追加到左值（数组追加元素，对象合并键） |
| `=` | **@覆盖**：仅将右值的 `@` 键值覆盖到左值的 `@` 键，其余键不变 |

### 2.9 注解（annotation）

```block_set_def
def annotation {
      "//"; any_char(N >= 0); newline;
    / "/*"; any_char(N >= 0); "*/";
};
```

注解在词法分析阶段视为空白，可在任何 token 之间出现。

---

## 3. 静态语义

### 3.1 作用域与可见性

```
┌─────────────────────────────────────────────┐
│  根 block ($)                                │
│  ┌──────────────────────────────────────┐   │
│  │  子 block A                            │   │
│  │  ┌───────────────────────────────┐    │   │
│  │  │  子 block B                    │    │   │
│  │  │  @ → B                         │    │   │
│  │  │  $ → 根                        │    │   │
│  │  └───────────────────────────────┘    │   │
│  │  @ → A     $ → 根                     │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

- **`$`**：始终指向最外层根 block。
- **`@`**：指向当前代码所在的 block。
- **`identifier_base`**（非链式）：局部变量，作用域为当前 block 的脚本区。每次调用开始时局部变量值为 `null`。
- **`~`** 链式导航：`$~a~b` 从根 block 开始沿数据键路径访问。`@~x~y` 从当前 block 开始。

### 3.2 虚拟变量

在 `para_list_Statement` 中声明的参数为**虚拟变量**。虚拟变量仅在当前 block 的脚本区内可见，不会被序列化到 JSON 输出中。

```
{
    greet(name: string, times: number): {
        message: "hello",
    },
    ---
    // name 和 times 在此脚本区可用
}
```

非虚拟的局部变量（在脚本区中直接使用的 `identifier_base`）每次调用开始时初始化为 `null`。

### 3.3 名称解析

1. 遇到 `identifier_base`（非链式），先在当前作用域的局部变量中查找。
2. 若未找到，在数据区的键中查找（`@~key`）。
3. 若仍未找到，可任意时刻报错（编译时或运行时）。

### 3.4 `key_set` 完整枚举

以下标识符被保留，不可作为 `identifier_base` 使用：

```
if  elseif  else  loop  break  continue
NOT  AND  OR  fn
true  false  null
placeholder  nonExist  derefer
```

---

## 4. 运行时语义

### 4.1 执行模型

1. 解析整个源文件，构建数据结构的内部表示。
2. 若存在脚本区（`---` 之后），按**从上到下**的顺序执行语句。
3. 脚本区语句可以读取、修改数据区的值，可以追加子键，但**不能删除或重命名**已有的键名。
4. 删除通过将键的值设为 `nonExist` 实现。
5. 所有语句执行完毕后，数据区即为最终结果。

### 4.2 `@` 隐含键与数组排序

每个 block 都有一个隐含的 `@` 键。

#### 数组排序规则

在数组中，元素按其 `@` 键值自动排序：

1. **无 `@` 键的元素**：保持插入顺序，位于所有有序元素之前。
2. **`@` 值为非数字的元素**：保持插入顺序，位于无 `@` 元素之后、数字 `@` 元素之前。
3. **`@` 值为数字的元素**：按数值升序排列。负数正常参与比较。

排序在**每次插入时**即时发生。排序是**每个数组独立**进行的。

`@` 键在最终 JSON 输出中**不被保留**。

**示例**：

```
arr: [
    {name: "c"},           // 无 @
    {@: "abc", name: "b"}, // 非数字 @
    {@: 3, name: "e"},     // 数字 @
    {@: 1, name: "d"},     // 数字 @
    {name: "a"},           // 无 @
]
```

排序结果（`@` 仅用于排序，不输出）：

```json
[
    {"name": "c"},
    {"name": "a"},
    {"name": "b"},
    {"name": "d"},
    {"name": "e"}
]
```

### 4.3 赋值语义详解

#### `#` 覆盖赋值

```
target # value
```

用 `value` 整体替换 `target`。`target` 原有内容被丢弃。

#### `<-` 追加赋值

```
target <- value
```

- 若 `target` 为数组：`value` 追加为数组的最后一个元素。
- 若 `target` 为对象：`value` 的键合并入 `target`。**键冲突为未定义行为**，实现可覆盖、报错或采取任意策略。
- 若 `target` 为 `null` 或 `placeholder`：`target` 变为 `value`。

#### `=` @覆盖赋值

```
target = value
```

仅将 `value` 的 `@` 键值写入 `target` 的 `@` 键。`target` 的其余键不受影响。

**示例**：

```
// 执行前: a = {@: 0, x: 1, y: 2}
//         b = {@: 99, z: 3}
a = b,
// 执行后: a = {@: 99, x: 1, y: 2}
```

### 4.4 `loop` / `break` / `continue`

```
loop outer {
    loop inner {
        if cond =? true {
            break outer,    // 跳出 outer 循环
        },
        if cond2 =? true {
            continue,       // 继续 inner 循环的下一次迭代
        },
    },
},
```

- `break` 和 `continue` 的标签查找是**编译时静态绑定**。
- `break 1` 跳出最内层（第 1 层），`break 2` 跳出第 2 层，以此类推至 `break 9`。
- `break 0` 无意义，实现可报错或静默忽略。
- 标签找不到或数字超出嵌套层数时，实现可在任意时刻（编译时或运行时）报错。

### 4.5 函数调用（`para_list`）

```
$~add(a, b)
```

- 参数传递为**引用传递**。
- 返回值也是引用。
- 被调用的 block 若声明了 `para_list_Statement`，参数与声明的形参绑定。
- 调用时执行该 block 的脚本区（若有），执行完毕后该 block 的数据区作为返回值。

---

## 5. 类型系统

### 5.1 基础类型

| 类型 | 说明 | 字面量示例 |
|------|------|-----------|
| `number` | 双精度浮点数 | `0` `-3.14` `1e10` |
| `string` | Unicode 字符串 | `"hello"` |
| `bool` | 布尔值 | `true` `false` |
| `null` | 空值 | `null` |
| `array` | 有序列表 | `[1, 2, 3]` |
| `block` | 键值对集合（即对象） | `{a: 1, b: 2}` |
| `placeholder` | 待填入占位符 | `placeholder` |
| `nonExist` | 不存在标记 | `nonExist` |
| `derefer` | 解引用提升标记 | `derefer` |

### 5.2 类型标注

类型标注出现在 `para_list_Statement` 中，`identifier_base: block` 的 `block` 部分作为类型引用：

```
key(a: number, b: string, c: bool): { ... }
```

- 类型检查可以在编译时或运行时进行，具体时机由实现决定。
- 语言**不提供隐式类型转换**。
- 类型标注可省略（`(a, b: number)` 中 `a` 无类型约束）。

### 5.3 真值规则

在 `if` 语句的条件位置，值的真值判断如下：

| 值 | 真值 |
|----|------|
| `false` | 假 |
| `null` | 假 |
| `placeholder` | 假 |
| `nonExist` | 假 |
| `0` (number) | 假 |
| `""` (空字符串) | 假 |
| `[]` (空数组) | 假 |
| `{}` (空 block) | 假 |
| 所有其他值 | 真 |

*（注：此规则与大多数动态语言一致。若有不同意见可调整。）*

---

## 6. 内置特殊值

### 6.1 `null`

- 表示空值。
- JSON 输出为 `null`。
- 脚本可以读取、覆盖。

### 6.2 `placeholder`

- 含义："值待脚本填入"。
- 可被脚本读取，读到的就是 `placeholder` 这个特殊值本身。
- 若最终 JSON 输出时某键的值仍为 `placeholder`，输出为 JSON `null`。
- 典型用法：声明一个键，值完全由脚本区决定。

```
{
    result: placeholder,
    ---
    result # {computed: 42},
}
// 最终输出: {"result": {"computed": 42}}
```

### 6.3 `nonExist`

- 含义："该键不出现在最终结果中"。
- 可被脚本读取，读到的就是 `nonExist` 这个特殊值本身。
- 最终 JSON 输出时，值为 `nonExist` 的键被**移除**（如同从未声明）。
- 典型用法：条件性删除。

```
{
    debug: "sensitive data",
    ---
    if isProduction =? true {
        debug # nonExist,
    },
}
// 若 isProduction 为 true，输出中不包含 debug 键
```

### 6.4 `derefer`

- 含义："自身消失，子 block 提升一级"。
- 可被脚本读取，读到的就是 `derefer` 这个特殊值本身。
- 在输出/求值时，`derefer` 节点被移除，其子键合并到父级。
- 在 `@` 键中和在普通值位置效力相同。

**示例 1** — 普通值位置：

```
// 输入
{
    a: derefer {
        b: 1,
        c: 2,
    },
    d: 3,
}

// 输出
{
    "b": 1,
    "c": 2,
    "d": 3
}
```

**示例 2** — 在 `@` 键中：

```
// 输入
{
    wrapper: {
        @: derefer {
            x: 10,
        },
        y: 20,
    },
}

// 输出
{
    "wrapper": {
        "x": 10,
        "y": 20
    }
}
```

---

## 7. JSON 互操作

### 7.1 序列化规则

Block Set 运行结束后，数据区被序列化为 JSON。

| Block Set 值 | JSON 输出 |
|-------------|-----------|
| `number` | JSON number |
| `string` | JSON string |
| `true` / `false` | JSON boolean |
| `null` | JSON `null` |
| `placeholder` | JSON `null` |
| `nonExist` | 键被移除，不输出 |
| `derefer` | 节点消失，子键提升 |
| `array` | JSON array |
| `block` | JSON object |

### 7.2 `@` 键

`@` 键在序列化时**始终不输出**。它仅用于内部排序和 `=` 赋值。

### 7.3 尾随逗号

Block Set 允许尾随逗号。序列化时输出标准 JSON（无尾随逗号）。

### 7.4 JSON 作为 Block Set 子集

任何合法 JSON 文档都是合法的 Block Set 文档：

- JSON 的 `{}` → Block Set 的无脚本区 `block_block`
- JSON 的 `[]` → Block Set 的 `array`
- JSON 的字符串键 → Block Set 的 `identifier`（字符串形式）
- JSON 的 `null`/`true`/`false` → Block Set 对应字面量

---

## 8. 标准库

`fn` 命名空间用于访问外部库。具体库由宿主环境提供。本规范定义最小标准库的预期接口。

### 8.1 `fn~string`

| 函数 | 签名 | 说明 |
|------|------|------|
| `len` | `fn~string~len(s: string): number` | 返回字符串长度 |
| `concat` | `fn~string~concat(a: string, b: string): string` | 拼接两个字符串 |
| `slice` | `fn~string~slice(s: string, start: number, end: number): string` | 取子串 `[start, end)` |
| `contains` | `fn~string~contains(s: string, sub: string): bool` | 是否包含子串 |

### 8.2 `fn~array`

| 函数 | 签名 | 说明 |
|------|------|------|
| `len` | `fn~array~len(a: array): number` | 返回数组长度 |
| `get` | `fn~array~get(a: array, i: number): block` | 取第 i 个元素 |
| `set` | `fn~array~set(a: array, i: number, v: block): array` | 设置第 i 个元素，返回数组 |

*（注：完整标准库由具体实现定义，此处仅为最小约定。）*

---

## 9. 附录：Block Set Def

`Block Set Def` 是用于定义 Block Set 语法的元语言。本附录给出其自举定义。

### 9.1 元语法符号

| 符号 | 含义 |
|------|------|
| `"..."` | 字面终结符 |
| `;` | 串联（sequence） |
| `/` | 或（alternative） |
| `( ... )` | 分组 |
| `( ... )(N >= 0)` | 重复 0 次或以上 |
| `( ... )(N >= 1)` | 重复 1 次或以上 |
| `( ... )(N = [m,n])` | 重复 m 到 n 次（含两端） |
| `not { ... }` | 排除集合 |
| `// --- ...` | 注释（到行尾） |

### 9.2 `nospace` 约束

`nospace` 表示两个语法元素之间不允许出现空白字符（空格、tab、换行）。在数字字面量等场景中，这由词法分析器通过将整个构造读取为单个 token 来实现。
