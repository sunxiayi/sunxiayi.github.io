---
title: MYSQL原生Online Schema Change
date: 2021-06-27
categories: CS
tags:
- mysql
- database
---

打算用三篇文章来介绍一下对于改变表结构的方案，分别是：
- 简介mysql原生online schema change（本篇）
- mysql以及其他db的instant DDL方案
- 在my集群中实现online schema change的工具


### 元数据锁（MDL）

表级的锁有两种，一种表锁，一种元数据锁（metadata lock，MDL）。MDL不需要显式使用，在访问一个表的时候会被自动加上，在语句执行开始时申请，在整个事务提交后释放。当对一个表做增删改查操作的时候，加MDL读锁，当要对表结构变更操作的时候，加MDL写锁。读锁之间不互斥，因此可以有多个线程同时对一张表增删改查。读写锁之间、写锁之间互斥，因此如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

这个锁的操作机制可能会对小表DDL都产生严重的影响。举个例子，session A在一个transaction中读表但没有commit（即没有释放MDL读锁），session B此时要对表进行DDL。后果是，session B拿不到写锁（因为MDL读写锁互斥），而后面的session也都拿不到读锁（被sessioon B阻塞了）。对这样的情况，比较理想的机制是，在alter table语句里设定等待的时间。在mariaDB/AliSQL里，有这个语法：

```mysql
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 
```


### Online DDL执行阶段

因为删除mysql的表和行的操作本质上是对数据标记“可复用”，而不是真的去删除文件，因此当碎片化到一定程度，可以通过重建表来优化/收缩所用空间。当我们用alter table A engine=InnoDB的命令来重建表，mysql会进行Online DDL，具体操作如下：

1. 启动alter语句，创建新的临时frm文件（存metadata文件，如表结构），拿MDL写锁
2. （除copy类型外）降级成MDL读锁，这样就不会阻塞增删改查
3. 真正做DDL，表可以正常读写数据（真正费时间的步骤）
4. 升级成MDL写锁
5. 释放MDL锁

对于第一点：为什么不能直接拿MDL读锁呢？是为了防止两个并发线程同时做表结构的并更，因为两个读锁是可以同时被拿到的，而两个写锁是不能同时被拿到的。

对于第二点：为什么需要拿着读锁，不直接解锁呢？还是为了禁止其它线程对这个表同时做DDL，因为读写锁是互斥的。

对于第三点和第四点，具体是这么操作的：

1. 建立一个临时文件，扫描A主键的所有数据页；
2. 用数据页中表A的记录生成B+树，存储到临时文件中；
3. 生成临时文件的过程中，将所有对A的操作记录在一个日志文件（raw log）中；
4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表A相同的数据文件；
5. 用临时文件替换表A的数据文件。



### Algorithm & Lock

在alter table的时候可以指定algorithm方式，有default, inplace, copy, instant四种。

- copy指的是在server层进行阻塞的拷贝新表操作，所有存储引擎都支持，操作时会创建临时表，执行全表拷贝和重建，过程中会写入redo log和大量的undo log。这不是一种online的算法。
- inplace指的是在储存引擎层进行的操作，不需要redo log等，其中大部分是非阻塞的（即用上面提到的方式进行），也有阻塞性质的（如第一次添加FULLTEXT和SPATIAL index的时候）。inplace算法分为重建表和非重建表两类方式，非重建表直接在原表上更新，重建表需要copy数据。
- instant用于避免inplace算法在需要修改数据文件时异常低效的问题，所有涉及到表拷贝和重建的操作都会被禁止

Online的DDL一定是用inplace/instant algorithm的。通过对小表DDL并检查rows affected的方式，可以知道table data有没有被copy。

Lock可指定4种模式，NONE允许增删改查，SHARED允许查询不允许DML，DEFAULT会看情况使用NONE或者SHARED，EXCLUSIVE会阻塞增删改查。



### Operation Details

|                   Operation                    | Instant | In Place | Rebuilds Table(1) | Only Modifies Metadata | Permits DML | Syntax                                                       |
| :--------------------------------------------: | ------- | -------- | ----------------- | ---------------------- | ----------- | ------------------------------------------------------------ |
|      Creating or adding a secondary index      | No      | Yes      | No                | No                     | Yes         | CREATE INDEX name ON table (col_list);<br>ALTER TABLE tbl_name ADD INDEX name (col_list); |
|               Dropping an index                | No      | Yes      | No                | Yes                    | Yes         | DROP INDEX name ON table;<br>ALTER TABLE tbl_name DROP INDEX name; |
|               Renaming an index                | No      | Yes      | No                | Yes                    | Yes         |                                                              |
|            Adding a FULLTEXT index             | No      | Yes      | No                | No                     | No          | CREATE FULLTEXT INDEX name ON table(column);                 |
|             Adding a SPATIAL index             | No      | Yes      | No                | No                     | No          |                                                              |
|            Changing the index type             | Yes     | Yes      | No                | Yes                    | Yes         | ALTER TABLE tbl_name DROP INDEX i1, ADD INDEX i1(key_part,...) USING BTREE \| HASH, ALGORITHM=INPLACE; |
|                 Adding a PK(2)                 | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE; |
|                 Dropping a PK                  | No      | No       | Yes               | No                     | No          | ALTER TABLE tbl_name DROP PRIMARY KEY, ALGORITHM=COPY;       |
|        Dropping a PK and adding another        | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name DROP PRIMARY KEY, ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE; |
|                Adding a col(3)                 | Yes     | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name ADD COLUMN column_name column_definition, ALGORITHM=INPLACE, LOCK=NONE;<br><br>ALTER TABLE t1 ADD COLUMN c2 INT, ADD COLUMN c3 INT, ALGORITHM=INSTANT; |
|                 Dropping a col                 | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name DROP COLUMN column_name, ALGORITHM=INPLACE, LOCK=NONE; |
|                 Renaming a col                 | No      | Yes      | No                | Yes                    | Yes         | ALTER TABLE tbl CHANGE old_col_name new_col_name data_type, ALGORITHM=INPLACE, LOCK=NONE; |
|                Reordering a col                | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name MODIFY COLUMN col_name column_definition FIRST, ALGORITHM=INPLACE, LOCK=NONE; |
|          Setting a col default value           | Yes     | Yes      | No                | Yes                    | Yes         | ALTER TABLE tbl_name ALTER COLUMN col SET DEFAULT literal, ALGORITHM=INPLACE, LOCK=NONE; |
|           Changing the col data type           | No      | No       | Yes               | No                     | No          | ALTER TABLE tbl_name CHANGE c1 c1 BIGINT, ALGORITHM=COPY;    |
|         Extending VARCHAR col size(4)          | No      | Yes      | No                | Yes                    | Yes         | ALTER TABLE tbl_name CHANGE COLUMN c1 c1 VARCHAR(255), ALGORITHM=INPLACE, LOCK=NONE; |
|         Dropping the col default value         | Yes     | Yes      | No                | Yes                    | Yes         | ALTER TABLE tbl ALTER COLUMN col DROP DEFAULT, ALGORITHM=INPLACE, LOCK=NONE; |
|       Changing the auto-increment value        | No      | Yes      | No                | No                     | Yes         | ALTER TABLE table AUTO_INCREMENT=next_value, ALGORITHM=INPLACE, LOCK=NONE; |
|               Making a col NULL                | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name MODIFY COLUMN column_name data_type NULL, ALGORITHM=INPLACE, LOCK=NONE; |
|             Making a col NOT NULL              | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name MODIFY COLUMN column_name data_type NOT ULL, ALGORITHM=INPLACE, LOCK=NONE; |
| Modifying the definition of an ENUM or SET col | Yes     | Yes      | No                | Yes                    | Yes         | ALTER TABLE t1 MODIFY COLUMN c1 ENUM('a', 'b', 'c', 'd'), ALGORITHM=INPLACE, LOCK=NONE; |
|             Adding a FK constraint             | No      | Yes      | No                | Yes                    | Yes         | ALTER TABLE *tbl1* ADD CONSTRAINT *fk_name* FOREIGN KEY *index* (*col1*)  REFERENCES *tbl2*(*col2*) *referential_actions*; |
|            Dropping a FK constraint            | No      | Yes      | No                | Yes                    | Yes         | ALTER TABLE tbl DROP FOREIGN KEY fk_name;<br>ALTER TABLE *table* DROP FOREIGN KEY *constraint*, DROP INDEX *index*; |
|            Changing the ROW_FORMAT             | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name ROW_FORMAT = row_format, ALGORITHM=INPLACE, LOCK=NONE; |
|          Changing the KEY_BLOCK_SIZE           | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name KEY_BLOCK_SIZE = value, ALGORITHM=INPLACE, LOCK=NONE; |
|      Setting persistent table statistics       | No      | Yes      | No                | Yes                    | Yes         | ALTER TABLE tbl_name STATS_PERSISTENT=0, STATS_SAMPLE_PAGES=20, STATS_AUTO_RECALC=1, ALGORITHM=INPLACE, LOCK=NONE; |
|           Specifying a character set           | No      | Yes      | Yes               | No                     | No          | ALTER TABLE tbl_name CHARACTER SET = charset_name, ALGORITHM=INPLACE, LOCK=NONE; |
|           Converting a character set           | No      | No       | Yes               | No                     | No          | ALTER TABLE tbl_name CONVERT TO CHARACTER SET charset_name, ALGORITHM=COPY; |
|               Optimizing a table               | No      | Yes      | Yes               | No                     | Yes         | OPTIMIZE TABLE tbl_name;                                     |
|        Rebuilding with the FORCE option        | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name FORCE, ALGORITHM=INPLACE, LOCK=NONE;    |
|           Performing a no-op rebuild           | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE tbl_name ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE; |
|                Renaming a table                | Yes     | Yes      | No                | Yes                    | Yes         | ALTER TABLE old_tbl_name RENAME TO new_tbl_name, ALGORITHM=INPLACE, LOCK=NONE; |
|              Adding a STORED col               | No      | No       | Yes               | No                     | No          | ALTER TABLE t1 ADD COLUMN (c2 INT GENERATED ALWAYS AS (c1 + 1) STORED), ALGORITHM=COPY; |
|           Modifying STORED col order           | No      | No       | Yes               | No                     | No          | ALTER TABLE t1 MODIFY COLUMN c2 INT GENERATED ALWAYS AS (c1 + 1) STORED FIRST, ALGORITHM=COPY; |
|             Dropping a STORED col              | No      | Yes      | Yes               | No                     | Yes         | ALTER TABLE t1 DROP COLUMN c2, ALGORITHM=INPLACE, LOCK=NONE; |
|              Adding a VIRTUAL col              | Yes     | Yes      | No                | Yes                    | Yes         | ALTER TABLE t1 ADD COLUMN (c2 INT GENERATED ALWAYS AS (c1 + 1) VIRTUAL), ALGORITHM=INSTANT; |
|          Modifying VIRTUAL col order           | No      | No       | Yes               | No                     | No          | ALTER TABLE t1 MODIFY COLUMN c2 INT GENERATED ALWAYS AS (c1 + 1) VIRTUAL FIRST, ALGORITHM=COPY; |
|             Dropping a VIRTUAL col             | Yes     | Yes      | No                | Yes                    | Yes         | ALTER TABLE t1 DROP COLUMN c2, ALGORITHM=INSTANT;            |
|                                                |         |          |                   |                        |             |                                                              |

(1) rebuild table要求重建clustered index和数据变更，会使操作更昂贵。

(2) 要重建在clustered index上的数据页，因此要rebuild table。加UNIQUE/PRIMARY index的时候MySQL会检查unique/not null的constraint是否成立，若有null的存在也不能用inplace模式。加PK是一个expensive的操作，强烈建议在建表的时候就加PK。

(3) 在加auto-increment column的时候会阻塞DML。对于8.0的instant algorithm，必须满足以下条件：

- ALTER TABLE语句中不能有其他非instant的操作；
- 被加的col只能是最后一列；
- 不支持ROW_FORMAT=COMPRESSED；
- 不支持有FULLTEXT index的表；
- 不支持加到temporary table；
- 不支持在data dictionary tablespace的表。

(4) 对于inplace algorithm，只支持VARCHAR从0-255，或者从256到之上的变化。如果是类似从254到256这样的变化不支持，也不支持减小VARCHAR的变化，这时要用copy。



### 原生DDL的局限性

- 在我看来最大的局限性在：在主从复制的环境下，必须等主库跑完DDL再跑备库的DDL（以及主库DDL期间的DML），因此对于大表来说replica lag是不可承受的
- 对于pause, throttle, rollback等没有支持



### References

- [MySQL :: MySQL 5.6 Reference Manual :: 14.13 InnoDB and Online DDL](https://fburl.com/o0fu0jpr)
- 极客时间 林晓斌 MYSQL实战45讲 第13章

