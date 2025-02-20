---
title: MySQL 的三大日志
tag: [MySQL] 
---

>  -  [必须了解的mysql三大日志-binlog、redo log和undo log](https://segmentfault.com/a/1190000023827696)
>  文章写的太好了，值得反复观看

`MySQL` 中存在各种日志（错误日志、查询日志、慢查询日志、事务日志、二进制日志等）
本文重点关注 `binlog`, `redo log`, `undo log` 这三大常用日志

## binlog

用于记录数据库执行的写入性操作（不包括查询信息 `select`），以二进制的形式保存在磁盘中

`binlog` 是 `MySQL` 的逻辑日志，并且由 `Server` 层进行记录，使用任何存储引擎的 `MySQL` 数据库都会记录 `binlog` 日志

*逻辑日志可以简单地理解为记录的就是 SQL 语句，而物理日志则具体的数据页变更*

`binlog` 通过追加的方式进行写入，可以通过参数设置每一个 `binlog` 文件的大小，当文件大小超过给定值之后，会生成新的文件来保存日志

### binlog 使用场景

- **主从复制**：从 `Master` 端开启 `binlog`，然后将 `binlog` 发送到各个 `Slave` 端，`Slave` 重放 `binlog` 从而达到主从数据一致
- **数据恢复**：通过 `mysqlbinlog` 工具来恢复数据
- **业务监控**：通过监控 `binlog` 的变更来达到业务上的一些需求，比如说实时更新缓存值

### binlog 刷盘时机

对于 `InnoDB` 引擎来说，只有事务提交时才会记录 `bin log`，此时记录还在内存中

`MySQL` 通过 `sync_binlog` 参数控制 `bin log` 的刷盘时机
`sync_binlog` 参数的取值范围为 `0 ~ N`
- 0: 不强制要求，由系统判断何时写入磁盘
- 1: 每次 `commit` 的时候都要将 `bin log` 写入磁盘
- N: 每 N 次事务才会将 `bin log` 写入磁盘

*`sync_binlog` 最安全的设置是 1，而显而易见，N 越大数据库性能越高，可以酌情牺牲一定的一致性来获取更好的性能*

### binlog 日志格式

`binlog` 有三中日志格式，分别为 `STATEMENT`、`ROW` 以及 `MIXED`

- `STATEMENT`
  基于 `SQL` 语句的复制（`statement-based replication`，`SBR`），每一条会修改数据的 `SQL` 语句会记录到 `binlog` 中
  
  - 优点：不需要记录每一行的变化，可以有效减少 `bin log` 的日志量，节省 `IO`
  - 缺点：某些情况下会导致主从数据不一致，比如执行 `sysdate ()`、 `sleep ()` 等

- `ROW`
  基于行的复制 （`row-based replication`，`RBR`），仅记录哪条数据修改了，不记录每条 `SQL` 的上下文信息
  
  - 优点：针对存储过程、`function`、`trigger` 的调用和触发也能被正确复制
  - 缺点：特定条件下会产生大量的日志，尤其是 `alter table` 时会让日志暴涨

- `MIXED`
  基于 `STATMENT` 和 `ROW` 两种模式的混合复制（`mix-based replication`，`MBR`）
  一般的复制使用 `STATEMENT` 模式保存 `bin log`，对于 `STATEMENT` 模式无法复制的操作使用 ROW `模式来保存` `binlog`

## redo log

### 为什么需要 redo log

事务四大特性之一的**持久性**，只要事务提交成功，那么对数据库做的修改就被永久保存下来了，不可能因为任何原因再回到原来的状态

那么，`MySQL` 是如何保证一致性的呢
最简单的办法就是每次事务提交的时候，就将该事务涉及修改的数据页全部刷新到磁盘中

但是，显然这存在严重的性能问题
- `InnoDB` 是以页为单位进行磁盘交互的，而一个事务很可能只修改一个数据页里面的几个字节，这个时候将完整的数据页刷到磁盘的话，就太浪费资源了
- 一个事务可能涉及多个数据页，并且这些数据页在物理上并不连续，使用**随机 `IO`** 写入性能太差

因此，`MySQL` 设计了 `redo log`，用来记录事务对数据页做了哪些修改
*文件更小且是顺序 IO*

### redo log 基本概念

`redo log` 包含两部分
- 内存中的日志缓冲 `redo log buffer`
- 磁盘上的日志文件 `redo log file`

`MySQL` 每执行一条 `DML` 语句，先将记录写入到 `redo log buffer`，后续某个时间点在一次性将多个操作记录写到 `redo log file`
*这种技术也就是 WAL (Write-Ahead Logging)*

在计算机操作系统中，用户空间下的缓冲区数据一般情况下是无法直接写入磁盘的
中间必须经过操作系统的内核空间缓冲区（`OS Buffer`）
因此，`redo log buffer` 写入到 `redo log file ` 的过程实际上是先写入 ` OS buffer`，然后再通过系统调用 `fsync() ` 将其刷到 ` redo log file`

`MySQL` 支持三种将 r`edo log buffer` 写入到 `redo log file` 的时机，通过 `innodb_flush_log_at_trx_comit` 参数
- `0`：延迟写
事务提交时不会将 `redo log buffer` 中日志写入到 `os buffer` ，而是每秒写入 `os buffer` 并调用 `fsync()` 写入到 `redo log file` 中
也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据

- `1`：实时写，实时刷
事务每次提交都会将 `redo log buffer` 中的日志写入 `os buffer` 并调用 `fsync()` 刷到 `redo log file` 中
这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO 的性能较差

- `2`：实时写，延迟刷
每次提交都仅写入到 `os buffer` ，然后是每秒调用 `fsync()` 将 `os buffer` 中的日志写入到 `redo log file`

### redo log 记录形式

`redo log ` 实际上记录的是数据页的变更
这种变更是没有必要全部保存的（因为已经落盘了）
所以，`redo log` 实现上采用了大小固定，循环写入的方式，当写到结尾时，会回到开头循环写日志
![image.png](https://cdn.jsdelivr.net/gh/logycoconut/pic-repo/tech/20240105184931.png)

在 `innoDB` 中，既有 `redo log` 需要刷盘，还有`数据页`需要刷盘
`redo log` 存在的意义主要就是降低对数据页刷盘的要求

在上图中，`write pos` 表示 `redo log` 当前记录的 `LSN` 位置
`check point` 表示数据页更改记录刷盘后对应 `redo log` 所处的 `LSN` 位置
*LSN：逻辑序列号，可以理解成不断递增的号码*

`write pos `和 `check point` 之间的部分就是 `redo log` 空着的部分，用于记录新的记录
`check point` 到 `write pos` 是 `redo log` 待落盘的数据页更改记录
当 `write pos` 追上 ` check point` 时，会先推动 `check point` 向前移动，空出位置再记录新的日志

启动 `innoDB` 的时候，**不管上次是正常关闭还是异常关闭，总是会进行恢复操作**
因为 `redo log` 记录的是数据页的物理变化，因为恢复的时候速度比 `bin log` 逻辑日志要快很多

重启 `innoDB` 时，首先会检查磁盘中数据页的 `LSN`，如果数据页的 `LSN` 小于日志中的 `LSN`，则会从 `check point` 开始恢复
还有一种情况，在宕机的时候正处于 `check point` 的刷盘过程，且数据页的刷盘进度超过了日志页的刷盘进度，此时会出现数据页中记录的 `LSN` 大于日志中的 `LSN`，这时超出日志进度的部分将不会恢复，因为这本身就表示已经做过的事情

### redo log 和 binlog 的区别

|          | redo log                                                          | binlog                                      |
| -------- | ----------------------------------------------------------------- | ------------------------------------------- |
| 文件大小 | `redo log` 的大小是固定的                                           | `binlog` 可通过配置设置每个 `bin log` 的文件大小 |
| 实现方式 | `redo log` 是 `InnoDB` 引擎层实现的                                   | `binlog` 是 `Server` 层实现的                   |
| 记录方式 | `redo log` 采用循环写的方式记录，当写在结尾时，会回到开头循环写日志 | `binlog` 通过追加的方式记录<br>当文件大小大于给定值后，后续的日志会记录到新的文件上                  |
| 使用场景 | 适用于崩溃恢复（`crash-safe`）                                      | 适用于主从复制和数据恢复                    |

由上述区别可知
- `binlog` 只用于归档，只依靠 `binlog` 是没有 `crash-safe` 能力的
- 只有 `redo log` 也是不行的，因为 `redo log` 是 `InnoDB` 引擎特有的，且日志上的记录落盘后会被覆盖掉
- 需要两者结合，同时记录，才能保证当数据库发生宕机重启时，数据不会丢失

## undo log

`MySQL` 的原子性就是通过 `undo log` 来实现的

对于每一条 `insert` 语句，都对应着相对的 `delete` 语句；每一条 `update` 语句，也有相反的 `update` 语句

这样，在发生错误需要回滚时，就能恢复到事务之前的数据状态，保证了原子性

***同时，`undo log` 也是 `MVCC` 实现的关键～***

## redo log的 crash safe 能力

> 什么是 `crash safe` ?
> 简单来说就是在 `MySQL` 发生 `crash` 后还能保证数据正确的能力

我们都知道，`redo log` 只会记录未刷盘的日志，已经刷入磁盘的数据都会从 `redo log` 中删除（可能暂时存在）

借助 `check point` 和 `write pos` 就能判断出哪些数据是需要恢复的数据，从而将其恢复到内存中

*这也解释了为什么 `binlog` 没有 `crash safe` 能力
在 `binlog` 中，并没有一个标志可以判断哪些数据已经落盘，哪些还没有*