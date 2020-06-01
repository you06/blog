title: yasp 开发日记【1】
author: you06
tags: []
categories: []
date: 2020-06-01 23:23:23
---
[`yasp`](https://github.com/Airyworks/yasp)(Yet Another SQL Parser) 是一个`SQL`解析工具，此前我用[`TiDB Parser`](https://github.com/pingcap/parser) 用得很爽，但是用`TiDB Parser`写测试工具存在一个固有问题是测试工具嫖了被测对象的代码是不合理的。本着严谨科学（和主要是喜欢造轮子）的态度，开了个`SQL Parser`（下简称`Parser`）的坑。

起初我打算用[`lalrpop`](https://github.com/lalrpop/lalrpop)一把梭，词法语法一起做了，但是这玩意的词法分析有个小坑是正则匹配不能冲突，以`SELECT`语句的解析为例：

```lalrpop
Comma<T>: Vec<T> = {
    <mut v:(<T> ",")*> <e:T?> => match e {
        None=> v,
        Some(e) => {
            v.push(e);
            v
        }
    }
};

Name: CIStr = r"[0-9a-zA-Z_]+" => <>.into();

pub Fields = Comma<Field>;

pub Field: Field = {
    "*" => Field::new_all(),
    Name => Field::new_column(<>),
    <table: Name>"."<column: Name> => Field::new_column(column).with_table(table),
};

ResultTable = Name;

pub Expr: Expr = {
    "select" <fields: Fields> "from" <result_table: ResultTable> => Expr::Select(SelectNode{
        fields,
        result_table,
    })
};
```

<details>
<summary>完整代码</summary>

```lalrpop
use crate::ast::{
    dml::*,
    expr::*,
    model::*
};

grammar;

Comma<T>: Vec<T> = {
    <mut v:(<T> ",")*> <e:T?> => match e {
        None=> v,
        Some(e) => {
            v.push(e);
            v
        }
    }
};

Semicolon<T>: Vec<T> = {
    <mut v:(<T> ";")*> <e:T?> => match e {
        None=> v,
        Some(e) => {
            v.push(e);
            v
        }
    }
};

pub Exprs = Semicolon<Expr>;

pub Expr: Expr = {
    "select" <fields: Fields> "from" <result_table: ResultTable> => Expr::Select(SelectNode{
        fields,
        result_table,
    })
};

Name: CIStr = r"[0-9a-zA-Z_]+" => <>.into();

pub Fields = Comma<Field>;

pub Field: Field = {
    "*" => Field::new_all(),
    Name => Field::new_column(<>),
    <table: Name>"."<column: Name> => Field::new_column(column).with_table(table),
};

ResultTable = Name;

```
</details>

这是一个简单`SELECT`语句的语法分析，从上往下看。
- `Comma`模板用于处理被逗号分隔的不定长项
- `Name`是一个匹配自定义字段名称的符号
- `Field`分析了`*`，`column`，`table.column`这三种情况
- `Fields`分析`SELECT`的多个目标字段
- `Expr`是一个简单的`SELECT`语法构成

这段代码的问题在于，`Expr`内部的`select`和`from`是固定关键词匹配，而`SQL`是一个关键词兼容大小写的语言，所以我们需要将其改为：

```lalrpop
pub Expr: Expr = {
    r"(?i)select" <fields: Fields> r"(?i)from" <result_table: ResultTable> => Expr::Select(SelectNode{
        fields,
        result_table,
    })
};
```

通过正则来匹配大小写的形式，看起来不错，但是编译却报错了。

```sh
~/workspace/rust/yasp(refactor*) » cargo test
   Compiling yasp v0.1.0 (/home/you06/workspace/rust/yasp)
error: failed to run custom build command for `yasp v0.1.0 (/home/you06/workspace/rust/yasp)`

Caused by:
  process didn't exit successfully: `/home/you06/workspace/rust/yasp/target/debug/build/yasp-01bf64e1e5e0f353/build-script-build` (exit code: 1)
--- stdout
processing file `/home/you06/workspace/rust/yasp/src/grammar.lalrpop`
/home/you06/workspace/rust/yasp/src/grammar.lalrpop:46:15: 46:30 error: ambiguity detected between the terminal `r#"[0-9a-zA-Z_]+"#` and the terminal `r#"(?i)from"#`

--- stderr
  Name: CIStr = r"[0-9a-zA-Z_]+" => <>.into();
```

报错的原因是`select`和`from`既能够满足我们所期望的`Expr`的语法，也能够满足`Field`的语法，所以`lalrpop`不知道它属于那一个符号。在`SQL`语言里，是不能够将例如`select`，`from`这种关键词作为表名和字段名来使用的。对于固定的字符串，`lalrpop`会将它置于比正则匹配更高的优先级，所以纯小写的`SQL`能够被解析，但如果有多个正则表达式，他们之前将无法区分优先级（也无法做优先级标注），这造成了直接解析的失败。
