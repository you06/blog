title: yasp 开发日记【2】
author: you06
date: 2020-08-17 14:25:50
tags:
---
[yasp 开发日记【1】](/2020/06/01/yasp-diary-1/)提到了我遇到关键字 ambiguous 的问题，这个问题在`lalrpop`的[Precedence of fixed strings](https://lalrpop.github.io/lalrpop/lexer_tutorial/001_lexer_gen.html#precedence-of-fixed-strings)章节也有所描述，并提供了解决方法。

`SQL`语言是一个流行的关键字大小写不敏感的语言，例如:

```sql
select uta from sakura;
SeLecT uta fRom sakura;
```

这两句`SQL`的语义实际上是一样的，但是如果我们这么写一个简陋的`lexer`:

```lalrpop
Name: String = r"\w+" => <>.into();
SELECT: &'input str = "select" => <>;
FROM: &'input str = "from" => <>;
pub Expr: Expr = {
    SELECT <field: Name> FROM <table: Name> => Expr::Select(SelectNode{
        field,
        table,
    })
};
```

那么这个`lexer`对于`select uta from sakura`来说没问题；但是对于`SeLecT uta fRom sakura`来说，`SeLecT`和`fRom`不能够被匹配成关键字。

需要注意的另一点是，根据[Precedence of fixed strings](https://lalrpop.github.io/lalrpop/lexer_tutorial/001_lexer_gen.html#precedence-of-fixed-strings)的描述，固定的字符串拥有比正则表达式更高的匹配优先级，所以这个词法分析器不存在歧义。

为了匹配`SeLecT`和`fRom`关键字，我们将`lexer`改成:

```lalrpop
Name: String = r"\w+" => <>.into();
SELECT: &'input str = r"(?i)select" => <>;
FROM: &'input str = r"(?i)from" => <>;
pub Expr: Expr = {
    SELECT <field: Name> FROM <table: Name> => Expr::Select(SelectNode{
        field,
        table,
    })
};
```

此时，`lalrpop`会认为我们所描述的语法存在歧义

```
error: ambiguity detected between the terminal `r#"\\w+"#` and the terminal `r#"(?i)from"#`
```

好吧，`lalrpop`不会处理推断多个正则表达式之间的优先级，相应的，文档中也给出了处理方式——预处理输入。

需要这样修改`lexer`:

```lalrpop
match {
    r"(?i)select" => "SELECT",
    r"(?i)from" => "FROM",
} else {
    _
}

Name: String = r"\w+" => <>.into();
SELECT: &'input str = "SELECT" => <>;
FROM: &'input str = "FROM" => <>;
pub Expr: Expr = {
    SELECT <field: Name> FROM <table: Name> => Expr::Select(SelectNode{
        field,
        table,
    })
};
```

虽然`lalrpop`不会自动推断多个正则表达式之间的优先级，但是在`match else`中，我们可以手动指定这些`token`的优先级。另外，这个写法也将关键词罗列在了一起，让代码可读性变的更强。
