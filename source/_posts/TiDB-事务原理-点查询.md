title: TiDB 事务原理 - 点查询
author: you06
date: 2020-07-29 07:39:45
tags:
---
近来在阅读`TiDB`事务方面的代码，所以写一些学习记录。

点查询指的是获取单条数据的`SQL`语句，相比于范围查询，点查询的执行过程要简单很多，原因有以下几点：

* 点查询的结果只可能在一个 region 内
* 点查询只会涉及到一个 key 上的锁处理

将`TiDB`测试用例中的点查询稍作修改：

```SQL
use test;
create table point (id int primary key, c int, d varchar(10), unique c_d (c, d));
insert point values (1, 1, 'a');
insert point values (2, 2, 'b');

select * from point where id = 1;
select * from point where c = 2 and d = 'b';
```

在完成建表和插入数据后，我们进行了三个查询，其中第一条`SQL`可以通过在主键`id`的等于条件查询条件知道最多有一条结果；第二条`SQL`可以通过`c`和`d`的等于查询条件结合`unique c_d (c, d)`的约束知道最多有一条结果；第三条`SQL`和第一条一样。然而在实际执行时，还需要考虑许多情况。

## 执行过程

### 构造 key

我们先在不考虑优化的情况下谈论执行过程（即严格按照`SI`的含义，对点查询取一个单独的快照来执行），`TiKV`是一个分布式`KV`数据库，即想要在`TiKV`中查出值来，首先需要有一个`key`，所以要先在`TiDB`里面把`key`给构造出来。

这里以`point(id int primary key, c int, d varchar(10), unique c_d (c, d))`表为例，有两种构造出点查询的方式。

```SQL
select * from point where id = 1;
key:
  74 800000000000002d 5f72  8000000000000001

select * from point where c = 2 and d = 'b';
key:
  74 800000000000002d 5f69  8000000000000001 038000000000000002 016200000000000000f8
  74 800000000000002d 5f72  8000000000000002
```

为了分析便利，这里列出了接下来可能会用到的`ASCII`表的内容。

|  16进制   | 字符  |
|  ----  | ----  |
| 0x5f  | 	_ |
| 0x62  | b |
| 0x69  | i |
| 0x72  | r |
| 0x74  | t |

对于第一条查询，因为`point`表中的主键字段为`id`，所以能够直接通过`id`来构造`key`；而对于第二条查询，我们只知道`c`和`d`的约束条件，需要先通过已知的条件确定主键的值再进行查询。

我有意的将`key`拆成了几段来进行分析

* 第一段`74 800000000000002d 5f72`和`74 800000000000002d 5f69`

* 第二段`8000000000000001`和`8000000000000001 038000000000000002 016200000000000000 f8`

#### 第一段

`74 800000000000002d 5f72`和`74 800000000000002d 5f69`是两种查询类型，其中还包括了表信息。

其中`800000000000002d`代表了表`id`，用`int64`来表示，但是在编码的时候把符号位强行置为了1。

查`ASCII`表之后，我们可以将这两个`key`写成`t45_r`和`t45_i`，可以参考以下代码。

```go
// github.com/pingcap/tidb/tablecodec/tablecodec.go
var (
	tablePrefix     = []byte{'t'}
	recordPrefixSep = []byte("_r")
	indexPrefixSep  = []byte("_i")
	metaPrefix      = []byte{'m'}
)
```

那么很容易理解，`recordPrefixSep`就是要查询记录，而`indexPrefixSep`是要查询索引，回到上面，`select * from point where c = 2 and d = 'b'`这条`SQL`要经过两次查询，第一次通过`unique`索引查询到主键索引，第二次再通过主键索引查询记录值。

#### 第二段

根据刚才的分析，`8000000000000001`很容易理解，指的就是索引的值（这里是`id`列），这里不再分析。

`8000000000000001 038000000000000002 016200000000000000 f8`是一条对索引进行查询的`key`的值。

* `8000000000000001`代表的是索引的`id`（即使在一张表内也可能存在多个索引）
* `038000000000000002`中`03`代表是整数类型，`8000000000000002`代表查询的值为2
* `016200000000000000f8`中`01`代表是`bytes`类型，`6200000000000000f8`是字符串`b`编码后的结果，`0x62`是`b`，后面一串`0`是为了对齐，最后的`0xf8`表示 padding 长度，此处为 7，`0xff - 7 = 0xf8`，详细的编码方式可以参考 [TiDB代码](https://github.com/pingcap/tidb/blob/b16c46dba91eb05b5d60c92d185443ead8f40394/util/codec/bytes.go#L34-L68)

### 查询 key

在有了`key`之后，`TiDB`会向对应`region`所在的`TiKV leader`发出`RPC`请求，此时，我们的问题变成了如何从一个带有分布式事务的`KV`存储引擎上查询一个`key`。

`KV`存储引擎需要考虑的问题：

* 1 如果尝试读取一个正在`commit`中的`key`，并且`commit ts`小于`get ts`，那么应该读到`commit`完成之后的值
* 2 处理 Internal Read 的情况

`TiKV`会首先拿取一个`snapshot`，然后检查这个`key`上存在的锁，锁信息存在一个叫`lock`的`CF`中，如果存在锁，则说明这个值可能有变化，即上面所说的问题1，此时`TiKV`会将标志位`met_newer_ts_data`从`NotMetYet`改为`Met`，然后执行`check_ts_conflict`函数。

```Rust
/// Checks whether the lock conflicts with the given `ts`. If `ts == TimeStamp::max()`, the primary lock will be ignored.
pub fn check_ts_conflict(self, key: &Key, ts: TimeStamp, bypass_locks: &TsSet) -> Result<()> {
	// 1. 锁的 ts 大于 snapshot 的 ts，显然应该忽略这把锁
    // 2. 锁有四种类型 Put, Delete, Lock, Pessimistic，后两种和数据无关，忽略
    if self.ts > ts
        || self.lock_type == LockType::Lock
        || self.lock_type == LockType::Pessimistic
    {
        // Ignore lock when lock.ts > ts or lock's type is Lock or Pessimistic
        return Ok(());
    }

    // Put 或 Delete 记录的最小 commit ts 大于 snapshot 的 ts，snapshot 读不到这个数据更新
    if self.min_commit_ts > ts {
        // Ignore lock when min_commit_ts > ts
        return Ok(());
    }

    // 这是一个 internal read，直接读取未 commit 的数据即可，例
    // begin
    // write(x, 1)
    // read(x)
    if bypass_locks.contains(self.ts) {
        return Ok(());
    }

    let raw_key = key.to_raw()?;

    // 为什么可以忽略这把锁，这里我也没看懂...
    if ts == TimeStamp::max() && raw_key == self.primary {
        // When `ts == TimeStamp::max()` (which means to get latest committed version for
        // primary key), and current key is the primary key, we ignore this lock.
        return Ok(());
    }

    // 至此，锁生效了，这里的 Err 会直接返回给发送 get rpc 请求的客户端（TiDB）
    // 这个查询会在 TiDB 里面被重试
    // There is a pending lock. Client should wait or clean it.
    Err(Error::from(ErrorInner::KeyIsLocked(
        self.into_lock_info(raw_key),
    )))
}
```

通过锁检查后，`met_newer_ts_data`这个标志位可能会有两种状态，`NotMetYet`和`Met`。

对于`NotMetYet`的情况，没有遇到这个`key`上存在的锁，所以只需要拿取最新数据就可以了。取数据的过程涉及到`MVCC`的操作，这里不会对其详细介绍（其实我也不大懂），可以参考[TiKV 源码解析系列文章（十三）MVCC 数据读取](https://pingcap.com/blog-cn/tikv-source-code-reading-13/)。

```Rust
let mut use_near_seek = false;
let mut seek_key = user_key.clone();

if self.met_newer_ts_data == NewerTsCheckState::NotMetYet {
    // 将 ts 置为 max，找到 lower_bound 的位置
    // 如果找不到，说明 key 不存在
    seek_key = seek_key.append_ts(TimeStamp::max());
    if !self
        .write_cursor
        .seek(&seek_key, &mut self.statistics.write)?
    {
        return Ok(None);
    }
    // 把刚才写入的 TimeStamp::max() 删掉，在 lower_bound 附近寻找目标版本
    seek_key = seek_key.truncate_ts()?;
    use_near_seek = true;

    let cursor_key = self.write_cursor.key(&mut self.statistics.write);
    // lower_bound 的 key 不等于要查询的 user_key（为什么没有在上面就返回 Ok(None) 0.0），感觉这里理解有误
    if !Key::is_user_key_eq(cursor_key, user_key.as_encoded().as_slice()) {
        return Ok(None);
    }
    // 遇到了新数据！
    if Key::decode_ts_from(cursor_key)? > self.ts {
        self.met_newer_ts_data = NewerTsCheckState::Met;
    }
}

seek_key = seek_key.append_ts(self.ts);
// 刚才用过 seek 的情况只需要 near_seek 就可以了
let data_found = if use_near_seek {
    self.write_cursor
        .near_seek(&seek_key, &mut self.statistics.write)?
} else {
    self.write_cursor
        .seek(&seek_key, &mut self.statistics.write)?
};
if !data_found {
    return Ok(None);
}

// 遍历 statistics.write，直到遇到 Put 和 Delete 记录，逻辑真的没看懂...
loop {
    // We may seek to another key. In this case, it means we cannot find the specified key.
    {
        let cursor_key = self.write_cursor.key(&mut self.statistics.write);
        if !Key::is_user_key_eq(cursor_key, user_key.as_encoded().as_slice()) {
            return Ok(None);
        }
    }

    self.statistics.write.processed += 1;
    let write = WriteRef::parse(self.write_cursor.value(&mut self.statistics.write))?;

    match write.write_type {
        WriteType::Put => {
            if self.omit_value {
                return Ok(Some(vec![]));
            }
            // 小于 64 bytes 的数据会被直接放在 write CF 里
            // 否则需要去 default CF 里获取
            match write.short_value {
                Some(value) => {
                    // Value is carried in `write`.
                    return Ok(Some(value.to_vec()));
                }
                None => {
                    let start_ts = write.start_ts;
                    return Ok(Some(self.load_data_from_default_cf(start_ts, user_key)?));
                }
            }
        }
        // 数据已被删除
        WriteType::Delete => {
            return Ok(None);
        }
        WriteType::Lock | WriteType::Rollback => {
            // Continue iterate next `write`.
        }
    }

    if !self.write_cursor.next(&mut self.statistics.write) {
        return Ok(None);
    }
}
```

emm 这 blog 写着写着发现挖了好多坑，搞懂之后一定填...
