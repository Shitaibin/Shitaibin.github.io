---
title: WAL(预写式日志)简介
date: 2020-01-02 20:56:32
tags: ['分布式', '存储']
---

## 引言

Write Ahead Logging，简称WAL，也被翻译成**预写式日志**，是数据库技术中实现事务日志(Transaction Journal)的一种标准方法，可以实现**单机**事务的原子性，同时可以提高数据库的写入效率。


思考如下场景，如何确保原子性：写操作修改数据库中a和b的值，二者是一个事务，需要把a和b的最新值持久化到磁盘，假如保存完a的值，系统宕机了，重新启动后，a的值已经写入，但b待写入的值已经丢失，如何发现事务没有完成呢？如何保证事务的原子性呢？

可以为事务加锁，也为事务增加标志位，修改完磁盘数据后，标志位设置事务为完成，事务状态保存在磁盘中，假使保存事务状态的过程中宕机了，就把事务回滚掉。实现REDO和UNDO，就能实现原子性。

数据库中针对**Crash**和**Recovery**的解决方案是WAL。

## 原理

WAL的核心思想是**先写日志再写数据文件**，修改数据文件必须发生在修改操作记录在日志文件之后。

> 本文的日志指事务的操作日志，本文提到的日志都是指事务日志，不再特殊声明。

![WAL](http://img.lessisbetter.site/2019-12-wal.png)

我们看WAL怎么**解决宕机和恢复的问题**：

- 写WAL前宕机了，重启后，数据处于事务未执行的状态。
- 写WAL时宕机了，重启后，可以检查到WAL数据不正确，回滚当事务前的状态。
- 写WAL后宕机了，重启后，把WAL中记录的操作，应用到数据库文件中，得到事务执行后的状态。

如此，保证了数据的恢复和事务的原子性。

上面提到的都是写操作，看一下使用WAL时的**读操作**。WAL中可能包含了未写入到数据库文件中的最新值，如果读最新值就需要从WAL中读取，如果WAL中未读到，从数据库读到的就是最新的数据。

**检查点**：写入到WAL文件中的操作记录并不一定会立刻应用到数据库文件上，这个过程是异步的，设计检查点来记录已经被应用到数据库文件上的操作序号，检查点后面的操作记录等待被应用到数据库文件上。


## 优点

WAL的作用是解决宕机和恢复的问题，同时也有其他优点：

1. **提高写数据的性能**
    1. WAL是顺序写，数据库文件是随机写，顺序写性能高于随机写
    2. 减少写磁盘次数
        1. 不直接修改数据库真实数据
        2. 合并若干小的事务，一次性commit到数据库
2. 保证事务**原子性**
3. 保证事务**一致性**
4. **并发读写**，比如SQLite中，读写、读读都是可以并行的，比如读时需要找到WAL某个值最后写入的位置，就可以从该位置读数据，而写操作是在WAL文件后Append，二者并行。但写写不能并行，因为2次写操作都要向WAL文件Append数据，无法同时进行。
5. WAL文件中记录了数据的历史版本，因此可以读取历史版本的值，甚至把状态回滚到某个历史版本。


## 缺点

SQLite提到了WAL的几项缺点：
1. WAL需要VFS的支持。
1. 所有使用数据库的进程必须在同一个机器上，以为WAL是单机的。
1. 多读少写的场景WAL比rollback-journal类型要慢1%~2%。


## 使用场景

WAL几乎是**数据存储**(数据库只是数据存储的一个类别，只不过这个类别很大)的标配：
- Raft可以使用WAL保存log Entry以及状态
- 数据库
    - PgSQL使用WAL实现事务日志实现事务原子性、一致性，提升性能
    - SQLite使用WAL实现原子性
    - MySQL使用WAL实现持久性，保证数据不丢失的情况下提升性能
    - leveldb也使用WAL提升性能，保证操作原子性


## 资料

- [菜鸟学数据库——WAL模式及其原理](https://juejin.im/post/5b04a93151882542672666e8)
- [PgSQL · 特性分析 · Write-Ahead Logging机制浅析](http://mysql.taobao.org/monthly/2017/03/02/)
- [PostgreSQL 9.1.24 Documentation: Chapter 29. Reliability and the Write-Ahead Log](https://www.postgresql.org/docs/9.1/wal-intro.html)
- [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html)
- [MySQL 8.0: New Lock free, scalable WAL design](https://mysqlserverteam.com/mysql-8-0-new-lock-free-scalable-wal-design/)