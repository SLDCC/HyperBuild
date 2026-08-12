---
version: 1
---

# Hyper Build File 规范

## 1. 概述

Hyper Build File（HBF）是一种声明式构建配置语言。文件扩展名为 `.hbf`。

HBF 的设计目标如下：

- 配置文件的导入关系构成一棵树。循环导入被禁止。
- 符号分为三种类型：句柄、值、方法。
- 不存在隐式查找链。未定义的符号引用导致错误。
- 方法体直接透传给 Shell。HBF 通过 `@` 指令介入控制流。

## 2. 词法规则

### 2.1 字符集

源文件采用 UTF-8 编码。换行符为 LF（`\n`）或 CRLF（`\r\n`）。

### 2.2 空白字符

空白字符包括空格（`0x20`）和水平制表符（`0x09`）。换行符不属于空白字符。

### 2.3 注释

注释以 `#` 字符开始，延伸至当前行末尾。注释内容被丢弃。

注释规则存在以下例外：

- 在方法体内部，`#` 不被识别为注释起始符。方法体内部的 `#` 原样透传给 Shell。
- 在双引号字符串内部，`#` 不被识别为注释起始符。

### 2.4 行约束

以下元素必须独占一整行。行内不得存在其他元素：

- 区块头：`[...]`
- 赋值语句的等号左侧及其关键字

双引号字符串值可以跨越多行。

### 2.5 字符串

字符串必须使用双引号（`"`）定界。字符串内容原样保留，不执行转义序列解析。

字符串内部可以包含 `${...}` 插值表达式。插值表达式在值解析阶段被求值。

字符串可以跨越多行。换行符原样保留。

### 2.6 标识符

标识符（IDENT）由以下字符组成：

- 首字符：大写字母、小写字母或下划线（`[A-Za-z_]`）
- 后续字符：大写字母、小写字母、数字或下划线（`[A-Za-z0-9_]`）

### 2.7 关键字

以下字符串为关键字，不可作为标识符使用：

区块关键字：`config`、`extern`、`import`、`input`、`vars`、`methods`、`export`

指令关键字：`call`、`let`、`foreach`、`for`、`in`、`if`、`else`、`setenv`、`log`、`error`、`warn`、`assert`、`silent`

### 2.8 符号

以下单字符或双字符为语法符号：

单字符：`{`、`}`、`(`、`)`、`[`、`]`、`=`、`,`、`.`、`@`、`$`、`#`

双字符：`${`、`$@`、`@@`、`$$`

### 2.9 元数据

元数据位于文件头部。元数据由零个或多个 `IDENT VALUE` 行组成。

元数据行中，`IDENT` 为标识符，`VALUE` 为从标识符结束至行尾（或注释起始）的裸字符串。元数据不包含等号。

元数据被词法分析器识别后丢弃。元数据对语法分析器不可见。

### 2.10 转义

转义序列在词法分析阶段被替换。

| 转义序列 | 替换结果      | 适用场景                         |
| -------- | ------------- | -------------------------------- |
| `@@`     | 单个 `@` 字符 | 方法体内部需要字面量 `@` 时      |
| `$$`     | 单个 `$` 字符 | 需要阻止 `${` 被识别为插值起始时 |

`$` 字符后跟非 `{` 字符时，`$` 不被识别为特殊字符，无需转义。

## 3. 语法规则

### 3.1 文件结构

```
file ::= metadata* section*
```

文件由零个或多个元数据行，后接零个或多个区块组成。

### 3.2 区块

```
section ::= '[' IDENT ']' entry*
```

区块以 `[IDENT]` 开始。`IDENT` 为区块类型。同一文件内，同一类型的区块最多出现一次。

支持的区块类型如下：

| 区块类型    | 内容           |
| ----------- | -------------- |
| `[config]`  | 引擎配置项     |
| `[extern]`  | 外部值符号声明 |
| `[import]`  | 子配置文件导入 |
| `[input]`   | 外部值接收声明 |
| `[vars]`    | 内部值符号定义 |
| `[methods]` | 方法定义       |
| `[export]`  | 符号导出声明   |

### 3.3 条目

```
entry ::= export_entry
       | import_entry
       | input_entry
       | var_entry
       | method_entry
```

#### 3.3.1 导出条目

```
export_entry ::= path_expr
              | IDENT '=' path_expr
```

`path_expr` 导出符号本身。`IDENT '=' path_expr` 执行重命名导出。导出的符号类型与被导出符号的类型一致。

#### 3.3.2 导入条目

```
import_entry ::= IDENT '=' STRING
              | '${' IDENT '}' '=' STRING
```

`IDENT '=' STRING` 将字符串指定的文件路径绑定到标识符，创建句柄。

`'${' IDENT '}' '=' STRING` 为模板导入。`'${' IDENT '}'` 中的标识符为模板形参。模板导入的具体求值时机由 Hyper Build（hb）引擎决定。

#### 3.3.3 输入条目

```
input_entry ::= IDENT '=' STRING
```

`IDENT` 为接收的符号名。`STRING` 为默认值。当父配置文件的 `[extern]` 区块中存在同名符号时，父级值覆盖默认值。`[input]` 定义的符号进入内部命名空间。

#### 3.3.4 变量条目

```
var_entry ::= IDENT '=' value_expr
```

`IDENT` 为变量名。`value_expr` 为值表达式。

#### 3.3.5 方法条目

```
method_entry ::= IDENT '=' '(' dep_list ')' block
              | '${' IDENT '}' suffix '=' '(' dep_list ')' block
```

`IDENT '=' '(' dep_list ')' block` 定义普通方法。

`'${' IDENT '}' suffix '=' '(' dep_list ')' block` 定义参数化方法。`'${' IDENT '}'` 中的标识符为形参。`suffix` 为 `.IDENT` 序列。

形参在方法签名和依赖列表中作为模式匹配占位符。方法体内部的 `${IDENT}` 为变量引用，读取内部命名空间。形参标识符与内部命名空间中的符号不允许同名。

### 3.4 依赖列表

```
dep_list ::= dep (',' dep)* | ε
dep      ::= path_expr | value_expr
```

`path_expr` 必须为方法类型。`value_expr` 必须为字符串类型。字符串类型的依赖项在增量构建中被解释为文件路径。

### 3.5 路径表达式

```
path_expr ::= IDENT ('.' IDENT)*
```

路径表达式由标识符通过 `.` 连接而成。路径表达式用于访问句柄成员或本地符号。

### 3.6 后缀

```
suffix ::= '.' IDENT
```

后缀用于参数化方法名的构造。

### 3.7 值表达式

```
value_expr ::= STRING
            | STRING_WITH_INTERPOLATION
```

值表达式为字符串，或包含 `${...}` 插值的字符串。

### 3.8 代码块

```
block ::= '{' stmt* '}'
```

`{` 和 `}` 作为 HBF 代码块定界符时，必须是所在行的第一个非空白字符。行中出现的 `{` 或 `}` 属于 Shell 代码，不参与代码块匹配。

代码块内部可以嵌套代码块（通过 `@` 指令）。

### 3.9 语句

```
stmt ::= '@' directive
      | shell_cmd
```

以 `@` 开头的语句为 HBF 指令。不以 `@` 开头的语句为 Shell 命令。Shell 命令从当前位置开始，直至遇到下一个以 `@` 开头的指令，或当前代码块结束。

### 3.10 指令

```
directive ::= 'call' call_target
             | 'let' IDENT '=' expr
             | 'foreach' IDENT 'in' expr block
             | 'if' expr block ('else' block)?
             | 'for' IDENT 'in' NUMBER '..' NUMBER block
             | 'setenv' IDENT expr
             | 'log' IDENT expr
             | 'error' expr
             | 'warn' expr
             | 'assert' expr expr
             | 'silent' block
```

#### 3.10.1 调用指令

```
call_target ::= path_expr
```

`path_expr` 必须解析为方法类型。`@call` 执行该方法，不捕获标准输出。

#### 3.10.2 局部绑定指令

```
'let' IDENT '=' expr
```

在方法体内部创建临时变量。作用域限于当前方法体。`IDENT` 不可与内部命名空间已有符号同名。

#### 3.10.3 迭代指令

```
'foreach' IDENT 'in' expr block
```

`expr` 必须求值为列表。`IDENT` 为迭代变量。对列表中的每个元素，执行一次 `block`。

```
'for' IDENT 'in' NUMBER '..' NUMBER block
```

`NUMBER` 为整数。`IDENT` 为迭代变量。对闭区间 `[NUMBER, NUMBER]` 内的每个整数，执行一次 `block`。

#### 3.10.4 条件指令

```
'if' expr block ('else' block)?
```

`expr` 在解析期求值。若求值结果为真，则包含 `if` 对应的 `block`；若存在 `else` 且求值结果为假，则包含 `else` 对应的 `block`。被排除的 `block` 不参与后续构建。

#### 3.10.5 环境变量指令

```
'setenv' IDENT expr
```

设置环境变量。`expr` 求值结果作为环境变量的值。作用域限于当前方法体的后续执行。

#### 3.10.6 日志指令

```
'log' IDENT expr
```

`IDENT` 为日志级别（如 `info`、`warn`、`error`）。`expr` 求值结果作为日志消息输出。

#### 3.10.7 错误指令

```
'error' expr
```

`expr` 求值结果作为错误消息输出。输出后，构建立即终止。

#### 3.10.8 警告指令

```
'warn' expr
```

`expr` 求值结果作为警告消息输出。构建继续执行。

#### 3.10.9 断言指令

```
'assert' expr expr
```

第一个 `expr` 为条件。第二个 `expr` 为失败消息。若条件求值为假，则输出失败消息并终止构建。

#### 3.10.10 静默指令

```
'silent' block
```

`block` 内部的 Shell 命令在执行时不打印到终端。

### 3.11 表达式

```
expr ::= '${' path_expr '}'
      | '$(' '@' IDENT expr* ')'
      | STRING
      | NUMBER
```

`'${' path_expr '}'` 为变量引用。`path_expr` 必须解析为值类型。

`'$(' '@' IDENT expr* ')'` 为 `@` 表达式调用。`IDENT` 为 `@` 表达式名称，后续 `expr*` 为参数。

`STRING` 为字符串字面量。

`NUMBER` 为整数。

## 4. 语义规则

### 4.1 符号类型

每个符号属于以下三种类型之一：

| 类型 | 定义位置                                    | 取值               | 调用              | 作为依赖               |
| ---- | ------------------------------------------- | ------------------ | ----------------- | ---------------------- |
| 句柄 | `[import]`                                  | 不可取值           | 不可调用          | 不可作为依赖           |
| 值   | `[vars]`、`[config]`、`[extern]`、`[input]` | 通过 `${...}` 取值 | 不可调用          | 可作为依赖（文件路径） |
| 方法 | `[methods]`                                 | 不可取值           | 通过 `@call` 调用 | 可作为依赖             |

### 4.2 命名空间

每个 `.hbf` 文件拥有三个独立的命名空间：

1. **内部命名空间**：包含 `[vars]`、`[config]`、`[methods]`、`[input]` 定义的符号。
2. **外部命名空间**：包含 `[extern]` 声明的符号。
3. **子命名空间**：包含 `[import]` 分配的句柄。每个句柄指向一个子配置文件的内部命名空间。

同一命名空间内不允许存在同名符号。不同命名空间允许同名符号。

### 4.3 作用域与查找

符号查找遵循严格规则：

- 内部命名空间优先于外部命名空间。
- 不存在隐式向上查找或向外穿透。未找到的符号导致错误。
- 外部符号必须通过 `[extern]` 声明的别名访问。
- 跨模块访问必须通过句柄路径（如 `K.output`、`K.build`）。
- 句柄本身不可取值。`${K}` 为非法表达式。句柄本身不可调用。`@call K` 为非法表达式。

### 4.4 导入与透传

- `[import]` 必须为每个子配置文件分配一个句柄符号。
- 导入链必须构成一棵树。检测到循环导入时，构建立即终止。
- `[export]` 可以导出本地符号、句柄本身、句柄成员、或重命名导出。
- 重命名导出继承被导出符号的类型。

### 4.5 输入语义

- `[input]` 声明子配置文件从父级 `[extern]` 接收的值符号。
- 父级 `[extern]` 存在同名符号时，父级值覆盖默认值。
- 父级 `[extern]` 不存在同名符号时，使用默认值。
- `[input]` 符号进入内部命名空间，与 `[vars]` 符号共享同一空间，不可同名。
- 仅值类型可通过 `[input]` 传递。句柄和方法不可通过 `[input]` 传递。

### 4.6 依赖语义

方法执行前，引擎按从左到右顺序处理依赖列表：

1. 若依赖项为方法，则执行该方法。若方法以非零退出码终止，则当前构建立即终止。
2. 若依赖项为值，则求值得到字符串，将该字符串解释为文件路径。检查该路径的文件是否存在。若不存在，则构建立即终止。

### 4.7 增量构建语义

引擎支持增量构建。增量构建通过 `[config]` 中的 `cache` 项启用。

方法执行前，引擎执行以下检查：

- 若任一方法依赖需要执行（递归判定其输入或自身存在变更），则触发当前方法。
- 若任一值依赖对应的文件自上次构建后存在更新（通过修改时间或内容哈希判定），则触发当前方法。
- 若所有方法依赖均未触发、所有值依赖文件均未更新、且当前方法的输出文件已存在，则跳过当前方法。

**当且仅当**上述三个条件同时满足时，当前方法被跳过。

### 4.8 方法体语义

- 方法体为 Shell 脚本超集。
- 不以 `@` 开头的语句原样透传给 Shell。
- `@` 开头的语句由 HBF 引擎解析执行。
- `@` 指令允许嵌套。

### 4.9 参数化方法

- 参数化方法的形参在方法签名和依赖列表中作为模式匹配占位符。
- 调用参数化方法时，实参替换形参，生成确定的方法名。
- 方法体内部的 `${IDENT}` 为变量引用，读取内部命名空间。形参标识符与内部命名空间符号不允许同名。

### 4.10 值解析

- 字符串内部的 `${...}` 在值解析阶段求值。
- 路径表达式在 `${...}` 中必须解析为值类型。
- `$$` 转义在词法分析阶段处理，阻止 `${` 被识别为插值起始。

### 4.11 解析期条件

`@if` 指令在解析期求值。条件表达式求值后，决定对应的代码块是否参与后续构建。被排除的代码块不进入执行阶段。

### 4.12 错误策略

构建采用 Fail-fast 策略。任何错误导致构建立即终止。

错误类别包括：

- 词法错误：非法字符、未闭合字符串。
- 语法错误：不符合语法规则。
- 语义错误：循环导入、符号未定义、类型不匹配、同名冲突。
- 运行时错误：依赖方法非零退出码、断言失败、文件不存在。

## 5. 配置项

`[config]` 区块支持以下配置项。所有配置项均为字符串类型。

| 配置项    | 默认值         | 说明                                                         |
| --------- | -------------- | ------------------------------------------------------------ |
| `shell`   | `"sh"`         | 方法体默认执行的 Shell。                                     |
| `verbose` | `"false"`      | 值为 `"true"` 时，打印执行的完整 Shell 命令。                |
| `cache`   | `".hbf-cache"` | 增量构建缓存目录路径。值为 `"none"` 时关闭增量构建。         |
| `default` | `""`           | 不指定目标时默认执行的方法名。空字符串表示必须显式指定目标。 |

`[config]` 配置项仅影响当前文件的方法体执行。子配置文件使用自身的 `[config]`。

## 6. 完整示例

### 6.1 `programs/editor.hbf`

```hbf
project Editor
version 3.1.0

[config]
    shell = "bash"
    verbose = "false"
    cache = "none"
    default = "build"

[vars]
    sources = "src/main.c src/buffer.c"
    output = "build/editor"

[export]
    output
    build

[methods]
    check = ()
    {
        @assert $(@exists "src/main.c") "Missing main.c"
    }

    build = (check)
    {
        ${compiler} -o ${output} ${sources}
    }
```

### 6.2 `kernel/kernel.hbf`

```hbf
project Kernel
version 5.2.0

[config]
    shell = "bash"
    verbose = "false"
    cache = "build/.hbf-cache"
    default = "build"

[import]
    ED = "../programs/editor.hbf"

[vars]
    compiler = "clang"
    output = "build/kernel"

[export]
    ED
    ED.output
    ED_OUT = ED.output
    build

[methods]
    check = ()
    {
        @assert $(@exists "arch/x86_64/boot.s") "Missing boot"
    }

    build = (check, ED.build)
    {
        @call ED.build
        ${compiler} -o ${output} arch/x86_64/boot.s
    }
```

### 6.3 `root.hbf`

```hbf
project CuberOS
version 1.0.0

[config]
    shell = "bash"
    verbose = "false"
    cache = "build/.hbf-cache"
    default = "all"

[extern]
    output
    B = "build"

[import]
    K = "./kernel/kernel.hbf"
    ${name} = "./programs/${name}.hbf"

[vars]
    output = "build/os.bin"
    compiler = "gcc"

[export]
    K.ED
    K.ED.output
    K.ED_OUT
    output
    all

[methods]
    init = ()
    {
        mkdir -p ${output}
    }

    main = (init)
    {
        ${compiler} -o ${output}/main src/main.c
    }

    ${name}.out = (init, ${name}.check)
    {
        @call ${name}.check
        ${compiler} -o ${output}/${name}.out src/${name}.cpp
    }

    all = ()
    {
        @call init
        @call main
        @call editor.out
        @call shell.out
        @call K.build

        @foreach src in $(@glob "programs/*.hbf")
        {
            @let stem = $(@stem ${src})
            @call ${stem}.out
        }

        @let hash = $(@call K.get_version)
        echo ${hash}
        echo ${B}
    }

    clean = ()
    {
        rm -rf ${output}
    }
```
