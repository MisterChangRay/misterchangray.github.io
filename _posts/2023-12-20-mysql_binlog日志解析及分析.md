mysql binlog 日志解析及分析流程

在mysql中，binlog记录了所有记录操作(insert,update,delete)当然也包括表变更等操作
本篇文章将详细讲解binlog格式以及常见的问题，为广大道友快速入门！

执行环境为:mysql8

---
layout: post
title:  "mysql binlog 日志解析及分析"
date:   2023-12-20 10:29:20 +0800
categories:
      - 命令脚本
      - mysql
tags:
      - mysql常用命令
      - mysqlbinlog解析
---
目录：
* awsl  
{:toc}

### 1. 确认是否打开binlog
```sql
-- 查询是否打开binlog
show variables like 'log_bin%';
-- 查询log列表
show master logs;
-- 查询当前使用的binlog是哪一个
show master status;
```

### 2. 解析日志文件
此步骤需要注意, mysqlbinlog 版本和数据库要一致，否则可能有些莫名其妙的错误

```shell
# 解析特定日期范围
mysqlbinlog --no-defaults -vvv --database=test  --base64-output=decode-rows -v --start-datetime='2023-11-19 13:28:01' --stop-datetime='2023-12-29 13:28:03'  E:\phpstudy_pro\Extensions\MySQL8.0.12\data\binlog.000027>D:\test\111.txt

# 解析整个日志文件
mysqlbinlog  --no-defaults -vvv --database=test  --base64-output=decode-rows  E:\phpstudy_pro\Extensions\MySQL8.0.12\data\binlog.000027 > D:\test\1.txt
```

### 3. 分析日志
一个完整的日志文件有很多数据，但是都是由以下部分组成

#### 1. 执行update 的 binlog日志 sql
```sql
-- 执行的sql
UPDATE `test`.`t_user` SET `name` = '4444', `phone` = '22' WHERE `id` = 1;
```
这里可以看出来,日志大概格式为: 
- 日志一般起始的地方为: `# at xxxx`
- 接下来为分配的线程id和初始化操作
- `COMMIT`代表当前sql或事务结束
- 虽然更新条件只有`id=1`但是binlog日志将会使用所有数据当作`where`条件

完整日志如下:

```
# at 1118568
#231219 14:15:33 server id 1  end_log_pos 1118643 CRC32 0xcefdd078 	Anonymous_GTID	last_committed=60	sequence_number=61	rbr_only=yes	original_committed_timestamp=1702966533187510	immediate_commit_timestamp=1702966533187510	transaction_length=371
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1702966533187510 (2023-12-19 14:15:33.187510 中国标准时间)
# immediate_commit_timestamp=1702966533187510 (2023-12-19 14:15:33.187510 中国标准时间)
/*!80001 SET @@session.original_commit_timestamp=1702966533187510*//*!*/;
/*!80014 SET @@session.original_server_version=999999*//*!*/;
/*!80014 SET @@session.immediate_server_version=999999*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1118643
#231219 14:15:33 server id 1  end_log_pos 1118727 CRC32 0xc674f08c 	Query	thread_id=2524	exec_time=0	error_code=0
SET TIMESTAMP=1702966533/*!*/;
BEGIN
/*!*/;
# at 1118727
#231219 14:15:33 server id 1  end_log_pos 1118788 CRC32 0x1b5d03ab 	Table_map: `test`.`t_user` mapped to number 5021
# at 1118788
#231219 14:15:33 server id 1  end_log_pos 1118854 CRC32 0xf1ca05fa 	Update_rows: table id 5021 flags: STMT_END_F
### UPDATE `test`.`t_user`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='1233' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
###   @3='22' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='4444' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
###   @3='22' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
# at 1118854
#231219 14:15:33 server id 1  end_log_pos 1118939 CRC32 0x02cb3a70 	Query	thread_id=2524	exec_time=0	error_code=0
SET TIMESTAMP=1702966533/*!*/;
COMMIT
/*!*/;
```



#### 2. 创建table的binlog日志
```sql
--  创建表的sql
CREATE TABLE `t_account` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `account` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
 PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```
从binlog日志可以看出这里没有嵌套事务，仔细想想也正常。 创建表是不需要事务的。

binlog日志如下：
```
# at 1118939
#231219 15:10:34 server id 1  end_log_pos 1119014 CRC32 0xc3c825c3 	Anonymous_GTID	last_committed=61	sequence_number=62	rbr_only=no	original_committed_timestamp=1702969834875690	immediate_commit_timestamp=1702969834875690	transaction_length=283
# original_commit_timestamp=1702969834875690 (2023-12-19 15:10:34.875690 中国标准时间)
# immediate_commit_timestamp=1702969834875690 (2023-12-19 15:10:34.875690 中国标准时间)
/*!80001 SET @@session.original_commit_timestamp=1702969834875690*//*!*/;
/*!80014 SET @@session.original_server_version=999999*//*!*/;
/*!80014 SET @@session.immediate_server_version=999999*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1119014
#231219 15:10:34 server id 1  end_log_pos 1119222 CRC32 0x9f7b1631 	Query	thread_id=2528	exec_time=0	error_code=0
SET TIMESTAMP=1702969834/*!*/;
CREATE TABLE `test`.`t_account`  (

  `id` int(0) NOT NULL AUTO_INCREMENT,

  `account` varchar(255) NULL,

  PRIMARY KEY (`id`)

)
/*!*/;
```

#### 3. 执行一个事务，里面执行3条sql:
```sql
-- 执行的sql
start TRANSACTION
INSERT INTO `test`.`t_user`(`id`, `name`, `phone`) VALUES (3, 'www3', '22');
update t_user set name = 'rrr3' where id = 3;
insert into t_account values(null, 'test3');
commit
```
从binlog中可以看出，事务执行也是遵循上述模板，所有的sql只有一个commit;

完整binlog：
```binlog
# at 1120650
#231219 15:34:23 server id 1  end_log_pos 1120725 CRC32 0x1e836084 	Anonymous_GTID	last_committed=67	sequence_number=68	rbr_only=yes	original_committed_timestamp=1702971263546547	immediate_commit_timestamp=1702971263546547	transaction_length=527
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1702971263546547 (2023-12-19 15:34:23.546547 中国标准时间)
# immediate_commit_timestamp=1702971263546547 (2023-12-19 15:34:23.546547 中国标准时间)
/*!80001 SET @@session.original_commit_timestamp=1702971263546547*//*!*/;
/*!80014 SET @@session.original_server_version=999999*//*!*/;
/*!80014 SET @@session.immediate_server_version=999999*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1120725
#231219 15:34:12 server id 1  end_log_pos 1120800 CRC32 0x893a38dd 	Query	thread_id=2537	exec_time=0	error_code=0
SET TIMESTAMP=1702971252/*!*/;
BEGIN
/*!*/;
# at 1120800
#231219 15:34:12 server id 1  end_log_pos 1120861 CRC32 0x4b4b94d5 	Table_map: `test`.`t_user` mapped to number 5027
# at 1120861
#231219 15:34:12 server id 1  end_log_pos 1120911 CRC32 0xc726c115 	Write_rows: table id 5027 flags: STMT_END_F
### INSERT INTO `test`.`t_user`
### SET
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='www3' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
###   @3='22' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
# at 1120911
#231219 15:34:13 server id 1  end_log_pos 1120972 CRC32 0x78e88a6d 	Table_map: `test`.`t_user` mapped to number 5027
# at 1120972
#231219 15:34:13 server id 1  end_log_pos 1121038 CRC32 0xbc8c37d9 	Update_rows: table id 5027 flags: STMT_END_F
### UPDATE `test`.`t_user`
### WHERE
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='www3' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
###   @3='22' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
### SET
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='rrr3' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
###   @3='22' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
# at 1121038
#231219 15:34:15 server id 1  end_log_pos 1121099 CRC32 0xfc5925ba 	Table_map: `test`.`t_account` mapped to number 5026
# at 1121099
#231219 15:34:15 server id 1  end_log_pos 1121146 CRC32 0xffd24deb 	Write_rows: table id 5026 flags: STMT_END_F
### INSERT INTO `test`.`t_account`
### SET
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='test3' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
# at 1121146
#231219 15:34:23 server id 1  end_log_pos 1121177 CRC32 0x390cb94e 	Xid = 126382
COMMIT/*!*/;

```



### 4. sql执行源头跟踪分析

首先查询当前执行的是否有当前线程，如果有的话则可以直接定位到发起者的ip, 用户等信息
```sql
select * from  information_schema.processlist where Id=#id
```

但更多的时候，sql发起者可能已经执行完毕并且断开连接，而`binlog`只能定位到当时是由哪个线程执行的，

mysql并不支持查询历史链接分配信息，所以需要自己来设计完成。

以下操作可以保存连接分配日志并：

1. 首先需要创建审计库以及表sql
```sql
create dabase  db_monitor;
use db_monitor;
CREATE TABLE `accesslog` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `thread_id` int(11) DEFAULT NULL,
  `log_time` datetime DEFAULT NULL,
  `localname` varchar(255) CHARACTER SET utf8 COLLATE utf8_unicode_ci DEFAULT NULL,
  `matchname` varchar(255) CHARACTER SET utf8 COLLATE utf8_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=8 DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;

```

2. 在mysql`init`链接中增加以下参数

修改my.cnf配置文件以下位置,修改后记得重启mysql。

```
# 原配置
init_connect='SET NAMES utf8'

# 修改为以下内容,主要是在默认配置前增加插入语句
init_connect='insert into db_monitor.accesslog(thread_id,log_time,localname,matchname) values(connection_id(),now(),user(),current_user());SET NAMES utf8'
```

3. 创建用户需要授权用户审计库权限
需要注意两点, root用户登录不会记录审计日志
```sql
 -- 创建用户test密码123456,并运行从任意IP链接
 create user 'test'@'%' identified by '123456';
 
 -- 赋予test用户 test,db_monitor 库的所有权限
 grant all privileges on test.* to 'test'@'%';
 grant all privileges on db_monitor.* to 'test'@'%';
 
 -- 只赋予查询权限
 grant select on db_monitor.* to 'test'@'%';
```

4. 测试审计功能

使用test链接进数据库，然后查看审计表。此时会有 `thread_id` 分配信息。 自此以后，就可以根据`bin_log中`的`thread_id`来进行定位了

值得一提的是,mysql中的`thread_id` 是根据连接数单调递增的，不是线程池循环利用，但是重启后会从1开始
