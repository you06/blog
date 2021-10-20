title: 理解 TPC-C
author: you06
date: 2021-01-19 15:42:23
tags:
---
TPC-C 是[联机交易](https://en.wikipedia.org/wiki/Online_transaction_processing)（下称 OLTP）型数据库最重要的 benchmark 之一（或许没有之一），做这类数据库的同学都应该有所耳闻，TPC-C 规范复杂，本文结合标准和实现详细讲解 TPC-C 的运行细节和难点。

- 标准: http://www.tpc.org/tpc_documents_current_versions/pdf/tpc-c_v5.11.0.pdf
- 实现: https://github.com/pingcap/go-tpc

![rikai-tpcc](./rikai-tpcc.jpg)

## TPC-C 是什么

TPC-C 是 TPC 系列 benchmark 之一，也是目前最权威的联机交易型数据库 benchmark 之一，关于其严格的运行要求可以阅读 [OceanBase 的解析文章](https://zhuanlan.zhihu.com/p/86186256)。

这里翻译标准文档里的描述作为 TPC-C 的介绍：

TPC-C 是一个 OLTP 型 workload，包含了只读事务和密集的更新事务。TPC-C 模拟了复杂 OLTP 应用场景，其中包含了：

- 同时执行多种类型的事务，来扩展系统复杂度
- 包含了联机事务和延迟事务执行的机制
- 保持多个 session 连接
- 可调整的系统和应用执行时间（？）
- 重 io 读写
- 要求事务（ACID 特性）
- 非均匀的数据分布，包含 primary key 和 secondary keys
- 包含了多张表，每张表有很多属性
- 数据读写存在冲突

TPC-C 的性能衡量指标是 tpmC，表示每分钟处理的交易数，一般商业数据库还会附加每 tpmC 所需要花费的硬件费用作为附加指标。

## 表类型

TPC-C 的模拟环境是一个公司，下属有 n 个 warehouses。

- 每个 warehouse 负责 10 districts 的供货
- 每个 district 有 3000 customers
- 公司提供 100000 items 销售货物，他们被存放在 warehouse 里

![tables](./tables.png)

TPC-C 会进行的操作有

- customer 会创建 order
- customer 会读取 order
- order 由 order_line 组成，平均有 10 条 order_lines
- 读取的 1% 的 order_lines 需要不在和 customer 所属的 warehouse 上
- 公司会根据客户订单创建 payments，发货并且补货

**注：在说明表结构时，n 代表 warehouse 数量，右上角的 <sup>*</sup> 角标代表主键索引，类型的长度均为标准要求的最小值。**

### warehouse 表

|Field Name|Field Definition|Comment|
|-|-|-|
||||

|Field Name|Field Definition|Comment|
|-|-|-|
|w_id <sup>*</sup>|unique id(2*n)|warehouse id|
|w_name|varchar(10)| |
|w_street_1|varchar(20)| |
|w_street_2|varchar(20)| |
|w_city|varchar(20)| |
|w_stage|varchar(20)| |
|w_zip|varchar(2)| |
|w_city|varchar(9)| |
|w_tax|decimal(4, 4)|Sales tax|
|w_ytd|decimal(12, 2)|Year to date balance|

### district 表

district 由 d_id 和 d_w_id 共同唯一确定，在初始化时会生成 $10 * n$ districts，表示为{% mathjax %}
\{\,(d\_id, d\_w\_id) \mid d\_id < 10 \wedge d\_w\_id<n\,\}
{% endmathjax %}。


|Field Name|Field Definition|Comment|
|-|-|-|
|d_id <sup>*</sup>|unique id(20)|每个 warehouse 生成 10 个|
|d_w_id <sup>*</sup>|unique id(2*n)|对应的 warehouse id|
|d_name|varchar(10)||
|d_street_1|varchar(20)||
|d_street_2|varchar(20)||
|d_city|varchar(20)||
|d_state|char(2)||
|d_zip|char(9)||
|d_tax|decimal(4, 4)|Sales tax|
|d_ytd|decimal(12, 2)|Year to date balance|
|d_next_o_id|unique id(10,000,000)|Next available Order number|

### customer 表

customer 表由 c_id, c_d_id, c_w_id 唯一标识，初始化时会生成 $10 * 3000 * n$ customers，表示为{% mathjax %}
\{\,(c\_id, c\_d\_id, c\_w\_id) \mid c\_id < 3000 \wedge c\_d\_id < 10 * n \wedge c\_w\_id<n\,\}
{% endmathjax %}。

|Field Name|Field Definition|Comment|
|-|-|-|
|c_id <sup>*</sup>|unique id(96,000)|每个 district 生成 3,000 个|
|c_d_id <sup>*</sup>|unique id(20)||
|c_w_id <sup>*</sup>|unique id(2*n)||
|c_first|varchar(15)||
|c_middle|char(2)||
|c_last|varchar(16)||
|c_street_1|varchar(20)||
|c_street_2|varchar(20)||
|c_city|varchar(20)||
|c_state|char(2)||
|c_zip|char(9)||
|c_phone|char(16)||
|c_since|datetime||
|c_credit|char(2)|"GC" = good, "BC" = bad|
|c_credit_lim|decimal(12, 2)||
|c_discount|decimal(4, 4)||
|c_balance|decimal(12, 2)||
|c_ytd_payment|decimal(12, 2)||
|c_payment_cnt|decimal(4, 0) => smallint <sup>[1]</sup>||
|c_delivery_cnt|decimal(4, 0) => smallint||
|c_data|varchar(500)||

- [1]： smallint size = 2 bytes (16 bit), $2^{16-1} > 9999$

### history 表

乱七八糟的东西都可以往 history 表里面塞，这张表会很大，很大。

|Field Name|Field Definition|Comment|
|-|-|-|
|h_c_id|unique id(96,000)||
|h_c_d_id|unique id(20)||
|h_c_w_id|unique id(2*n)||
|h_d_id|unique id(20)||
|h_w_id|unique id(2*n)||
|h_date|datetime||
|h_amount|decimal(6, 2)||
|h_data|varchar(24)|Misc|

### new_order 表

|Field Name|Field Definition|Comment|
|-|-|-|
|no_o_id <sup>*</sup>|unique id(10,000,000)||
|no_d_id <sup>*</sup>|unique id(20)||
|no_w_id <sup>*</sup>|unique id(2*n)||

### order 表

|Field Name|Field Definition|Comment|
|-|-|-|
|o_id <sup>*</sup>|unique id(10,000,000)||
|o_d_id <sup>*</sup>|unique id(20)||
|o_w_id <sup>*</sup>|unique id(2*n)||
|o_c_id|unique id(96,000)||
|o_entry_id|datetime||
|o_carrier_id|unique id(10) null||
|o_ol_cnt|decimal(2, 0) => tinyint <sup>[1]</sup>|Count of order lines|
|o_all_local|decimal(1, 0) => tinyint||

- [1]： tinyint size = 1 bytes (8 bit), $2^{8-1} > 99$

### order_line 表

|Field Name|Field Definition|Comment|
|-|-|-|
|ol_o_id <sup>*</sup>|unique id(10,000,000)||
|ol_d_id <sup>*</sup>|unique id(20)||
|ol_w_id <sup>*</sup>|unique id(2*n)||
|ol_number <sup>*</sup>|unique id(15)||
|ol_i_id|unique id(200,000)||
|ol_supply_w_id|unique id(2*n)||
|ol_delivery_d|datetime||
|ol_quantity|decimal(2, 0) => tinyint||
|ol_amount|decimal(6, 2)||
|ol_dist_info|char(24)||

### item 表

|Field Name|Field Definition|Comment|
|-|-|-|
|i_id <sup>*</sup>|unique id(200,000)||
|i_im_id|unique id(200,000)|item 对应的 image id|
|i_name|varchar(24)||
|i_price|decimal(5, 2)||
|i_date|varchar(50)|无意义的商品信息|

### stock 表

|Field Name|Field Definition|Comment|
|-|-|-|
|s_i_id|unique id(200,000)||
|s_w_id|unique id(2*n)||
|s_quantity|decimal(4, 0) => smallint||
|s_dist_01|char(24)||
|s_dist_02|char(24)||
|s_dist_03|char(24)||
|s_dist_04|char(24)||
|s_dist_05|char(24)||
|s_dist_06|char(24)||
|s_dist_07|char(24)||
|s_dist_08|char(24)||
|s_dist_09|char(24)||
|s_dist_10|char(24)||
|s_ytd|decimal(8, 0) => int <sup>[1]</sup>||
|s_order_cnt|decimal(4, 0) => smallint||
|s_remote_cnt|decimal(4, 0) => smallint||
|s_data|varchar(50)|无意义的仓库信息|

- [1]： int size = 4 bytes (32 bit), $2^{32-1} > 99,999,999$















