# mysql deadlock错误排查记录

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
`CREATE UNIQUE INDEX uix_user_intergral_uid_otype_source ON user_intergral (uid, otype, source, ts);`
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
RECORD LOCKS space id 0 page no 163602 n bits 424 index `uix_user_intergral_uid_otype_source` of table `intergral`.`user_intergral` trx id 12E0BBD2 lock_mode X locks gap before rec insert intention waiting
*** (2) TRANSACTION:
TRANSACTION 12E0BBD1, ACTIVE 0 sec fetching rows
mysql tables in use 1, locked 1
14 lock struct(s), heap size 3112, 2191 row lock(s)
MySQL thread id 1311763, OS thread handle 0x7f6cfb803700, query id 538285481 event_scheduler updating
DELETE FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW())
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 0 page no 163602 n bits 424 index `uix_user_intergral_uid_otype_source` of table `intergral`.`user_intergral` trx id 12E0BBD1 lock mode S
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
`X Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source`
#### **transaction(2)12E0BBD1**
##### sql
```
DELETE FROM user_intergral WHERE time_limit > 0 AND ts + time_limit < UNIX_TIMESTAMP(NOW())
```
##### 持有锁
`S Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source`
##### 等待锁
`X Lock: space 0 page 163711 bit 272 index PRIMARY`
#### 资源分析
transaction|hold|wait
-----------|----|----
12E0BBD2|space 0 page 163711 bit 272 index PRIMARY|X Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source
12E0BBD1|S Lock: space 0 page 163602 bit 424 index uix_user_intergral_uid_otype_source|X Lock: space 0 page 163711 bit 272 index PRIMARY

