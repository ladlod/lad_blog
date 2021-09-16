# mysql deadlock错误排查记录

<div align='right'><b>auth:ladlod</b></div>

**********************************************
## 问题
使用on duplicate语句和唯一索引对数据进行upsert操作时报了deadlock
### table
```
CREATE TABLE IF NOT EXISTS user_intergral (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    uid VARCHAR(64) NOT NULL,
    otype VARCHAR(10) NOT NULL,
    source VARCHAR(16) NOT NULL DEFAULT 'default',
    intergral BIGINT NOT NULL DEFAULT 0,
    ts BIGINT NOT NULL DEFAULT 0,
    time_limit BIGINT NOT NULL DEFAULT 0
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
### index
`CREATE UNIQUE INDEX uix_user_intergral_uid_otype_source_ts ON user_intergral (uid, otype, source, ts);`
### event
```
DELIMITER $$
CREATE PROCEDURE user_intergral_delete_timeout()
BEGIN
    START TRANSACTION;
    INSERT INTO intergral_record (
    SELECT uid, otype, source, 3 AS type, sum(intergral) AS intergral, "intergral out date" AS detail, UNIX_TIMESTAMP(NOW()) AS ts
    FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW())
    GROUP BY uid, otype, source
    );
    DELETE FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW());
    COMMIT;
END$$
DELIMITER ;
CREATE EVENT IF NOT EXISTS user_intergral_delete_timeout_schedule
ON schedule every 1 SECOND
DO CALL user_intergral_delete_timeout();
```
### sql
```
insert into user_intergral(otype, uid, source, intergral, ts, time_limit)
values(?, ?, ?, ?, ?, ?)
on duplicate key update intergral = intergral + 25
```
### err
`Error 1213: Deadlock found when trying to get lock; try restarting transaction`

## 基础知识储备
### 数据库隔离级别
- read uncommitted
- read committed (oracle default)
- repeatable read (innodb/myIsam default)
- serializeable
### 数据库隔离级别相关命令
- 查看当前会话隔离级别 `select @@tx_isolation;`
- 设置当前会话隔离级别 `set session transaction isolation level *;`
- 查看系统隔离级别 `select @@global.tx_isolation;`
- 设置系统隔离级别 `set global transaction isolation level *;`
### innodb锁
- S锁：共享锁，一个事务对数据对象使用S锁，则可以读对象而不可以修改对象，其它事务只可以对该对象加S锁，不可以加X锁。
- X锁：排它锁，一个事务对数据对象使用X锁，则可以读且修改对象，其它事务不可对该对象加任何锁。
- 间隙锁：间隙锁是innodb在repeatable read隔离级别下为了解决幻读引入的锁机制。即对索引的某个范围加锁。
### show engine innodb status
innodb存储引擎使用该语句可以显示大量的引擎内部信息，下面逐行介绍

title|comment
-----|-------
`210820 11:43:25 INNODB MONITOR OUTPUT`|头部信息，当前日期、时间、计算所用时间间隔等
`BACKGROUND THREAD` | 后台线程信息
`SEMAPHORES`| 信号量，它包含两种信息，事件计数器及可选线程列表
`LATEST FOREIGN KEY ERROR`|外键错误信息
`LATEST DETECTED DEADLOCK`|死锁错误信息
`TRANSACTIONS`|事务总结信息，及活跃事务列表
`FILE I/O`|显示I/O辅助线程的状态及性能计数器的状态
`INSERT BUFFER AND ADAPTIVE HASH INDEX`|显示insert buffer和adaptive hash index两个结构的状态
`LOG`|显示了innodb事务日志
`BUFFER POOL AND MEMORY`|显示缓冲池和内存信息
`ROW OPERATIONS`|显示其它各项innodb的统计，如内核线程数、增删改查行数等

## 排查过程
### 查看mysql关于死锁的监控信息
```
LATEST DETECTED DEADLOCK
------------------------
210820 10:02:38
*** (1) TRANSACTION:
TRANSACTION 12E0BBD2, ACTIVE 0 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1248, 2 row lock(s), undo log entries 1
MySQL thread id 1311320, OS thread handle 0x7f6cfa69e700, query id 538285482 hostport user update
insert into user_intergral(otype, uid, source, intergral, ts, time_limit)
	values(?, ?, ?, ?, ?, ?)
	on duplicate key update intergral = intergral + 25
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 0 page no 163602 n bits 424 index `uix_user_intergral_uid_otype_source_ts` of table `intergral`.`user_intergral` trx id 12E0BBD2 lock_mode X locks gap before rec insert intention waiting
*** (2) TRANSACTION:
TRANSACTION 12E0BBD1, ACTIVE 0 sec fetching rows
mysql tables in use 1, locked 1
14 lock struct(s), heap size 3112, 2191 row lock(s)
MySQL thread id 1311763, OS thread handle 0x7f6cfb803700, query id 538285481 event_scheduler updating
DELETE FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW())
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 0 page no 163602 n bits 424 index `uix_user_intergral_uid_otype_source_ts` of table `intergral`.`user_intergral` trx id 12E0BBD1 lock mode S
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 0 page no 163711 n bits 272 index `PRIMARY` of table `intergral`.`user_intergral` trx id 12E0BBD1 lock_mode X waiting
*** WE ROLL BACK TRANSACTION (1)
```
### 分析transactions
#### **transaction(1)12E0BBD2**
##### sql
```
insert into user_intergral(otype, uid, source, intergral, ts, time_limit)
	values(?, ?, ?, ?, ?, ?)
	on duplicate key update intergral = intergral + 25
```
##### 等待锁：
`X Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source_ts`
#### **transaction(2)12E0BBD1**
##### sql
```
DELETE FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW())
```
##### 持有锁
`S Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source_ts_ts`
##### 等待锁
`X Lock: space 0 page 163711 bit 272 index PRIMARY`
### 资源分析
transaction|sql|hold|wait
-----------|---|----|----
12E0BBD2|`insert into user_intergral(otype, uid, source, intergral, ts, time_limit) values(?, ?, ?, ?, ?, ?) on duplicate key update intergral = intergral + 25`|S Lock:space 0 page 163711 bit 272 index PRIMARY|X Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source_ts
12E0BBD1|`DELETE FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW())`|S Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source_ts|X Lock: space 0 page 163711 bit 272 index PRIMARY
### 猜想
transaction2在进行delete操作时部分命中了uix_user_intergral_uid_otype_source_ts索引，并对对应的数据资源加S间隙锁，并等待主键索引上对应资源的X锁。
transaction1在进行insert时命中主键索引并对对应资源加S锁，并在check duplicate时等待uix_user_intergral_uid_otype_source_ts索引对应资源上的X锁。
所以构成了死锁。
### 验证
session1|session2
--------|--------
beign;|-
-|begin;
`insert into user_intergral(otype, uid, source, intergral, ts, time_limit) values('test', 'testid', 'testsrc', 10, 0, 0) on duplicate key update intergral = intergral + 25;`|-
-|`DELETE FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW());`
commit;|-
-|`ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction`
## 结论
### 解决方案
## 处理索引冲突
对transaction2查询的索引情况进行处理，由于transaction2查询中覆盖ts及time_limit，可以在time_left上创建索引，避免transaction2与transaction1争抢同一个索引上的锁。
但此方案只能解决当前场景下的问题，由于`insert * on duplicate key update *`语句在处理过程中是先对主键索引上对应资源加S锁，而后去请求唯一索引上的X锁，若同时有另一个事务占有了该唯一索引上对应资源的S锁，便会产生死锁，所以建议从根本上解决问题。
## 修改查询语句
将transaction1修改为先查询，后修改，取锁过程改为先获取唯一索引上的X锁，而后获取主键索引和唯一索引上的S锁，将该操作作为一个事务处理，可以从根本上避免这种死锁的情况。
### 思考
可以通过使用事务而不是使用lock table statements来避免死锁；在不同的事务update多个表或一个表的大范围行的时候，每个事务的操作都要保证相同的顺序；给select and update的列加索引也可以降低死锁发生的概率。

> 关于死锁，mysql的官方文档：[Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/5.6/en/innodb-deadlocks.html)