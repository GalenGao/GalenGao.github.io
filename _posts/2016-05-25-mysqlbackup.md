---
layout: post
title:  "mysql cp复制和mysqldump备份测试"
date:   2016-05-25 14:32:04 +0700
categories: [linux, mysql]
---

# 备份策略

针对不同的场景下, 我们应该制定不同的备份策略对数据库进行备份, 一般情况下, 备份策略一般为以下三种：  


- 直接cp,tar复制数据库文件  
- mysqldump 复制BIN LOGS  
- lvm2快照 复制BIN LOGS
- xtrabackup  
以上的几种解决方案分别针对于不同的场景
- 如果数据量较小, 可以使用第一种方式, 直接复制数据库文件  
- 如果数据量还行, 可以使用第二种方式, 先使用mysqldump对数据库进行完全备份, 然后定期备份BINARY LOG达到增量备份的效果  
- 如果数据量一般, 而又不过分影响业务运行, 可以使用第三种方式, 使用lvm2的快照对数据文件进行备份, 而后定期备份BINARY LOG达到增量备份的效果  
- 如果数据量很大, 而又不过分影响业务运行, 可以使用第四种方式, 使用xtrabackup进行完全备份后, 定期使用xtrabackup进行增量备份或差异备份  
 
# 实战演练

## cp复制数据文件备份及恢复

{% highlight sql %}  
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.03 sec)

mysql> create database test;
Query OK, 1 row affected (0.03 sec)

mysql> use test
Database changed
mysql> show tables;
Empty set (0.00 sec)

mysql> create table a(id int,name varchar(10));
Query OK, 0 rows affected (0.04 sec)

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| a              |
+----------------+
1 row in set (0.00 sec)

mysql> insert into a(id,name)values(1,'gao')
    -> ;
Query OK, 1 row affected (0.01 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from a
    -> ;
+------+------+
| id   | name |
+------+------+
|    1 | gao  |
+------+------+
1 row in set (0.00 sec)
{% endhighlight %}

{% highlight bash %} 
[root@my57 mysql]# mkdir /backup

[root@my57 mysql]# cp -a /data/mysql/data/* /backup/

[root@my57 mysql]# ls /backup/
auto.cnf  ib_buffer_pool  ibdata1  ib_logfile0  ib_logfile1  ibtmp1  my57.err  my57.pid  mysql  mysqld_safe.pid  performance_schema  sys  test  xtrabackup_info

[root@my57 mysql]# rm -rf /data/mysql/data/*
[root@my57 mysql]# service mysql restart

 ERROR! MySQL server PID file could not be found!
Starting MySQL... ERROR! The server quit without updating PID file (/data/mysql/data//my57.pid).
{% endhighlight %}

这时启动不了，我们再把备份文件拷贝回来，在启动就可以了。

```Bash
cp -a /backup/* /data/mysql/data/
```

## mysqldump的复制与恢复

{% highlight SQL %}
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.02 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| a              |
| b              |
+----------------+
2 rows in set (0.00 sec)
mysql> select * from b;
+------+
| id   |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

# 记住备份前position的值
mysql> show master status
    -> ;
+----------------+----------+--------------+------------------+-------------------+
| File           | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+----------------+----------+--------------+------------------+-------------------+
| bin-log.000001 |      567 |              |                  |                   |
+----------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
{% endhighlight %}

**开始备份**

```Bash
[root@my57 data]# mysqldump --all-databases --lock-all-tables  > /backup/backup.sql
```

**再创建一个数据库做增量测试**

{% highlight SQL %}
mysql> create database test1;
Query OK, 1 row affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
| test1              |
+--------------------+
6 rows in set (0.00 sec)

# 再记下现在的position位置
mysql> SHOW MASTER STATUS;
+----------------+----------+--------------+------------------+-------------------+
| File           | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+----------------+----------+--------------+------------------+-------------------+
| bin-log.000001 |      869 |              |                  |                   |
+----------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
{% endhighlight %}

**备份二进制日志**

```Bash
cp bin-log.000001 /backup/
```

**停止mysql在启动，编译安装的启动不了，必须重新初始化**

{% highlight Bash %}
[root@my57 data]# service mysql stop
Shutting down MySQL. SUCCESS! 

# 模拟删除数据文件
[root@my57 data]# rm -rf *

# 编译安装的数据库删除数据文件后启动不了
[root@my57 data]# service mysql start
Starting MySQL... ERROR! The server quit without updating PID file (/data/mysql/data//my57.pid).

# 重新初始化mysql
[root@my57 data]# /usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql
2016-06-14T02:43:04.084864Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2016-06-14T02:43:04.084944Z 0 [Warning] 'NO_ZERO_DATE', 'NO_ZERO_IN_DATE' and 'ERROR_FOR_DIVISION_BY_ZERO' sql modes should be used with strict mode. They will be merged with strict mode in a future release.
2016-06-14T02:43:04.084952Z 0 [Warning] 'NO_AUTO_CREATE_USER' sql mode was not set.
2016-06-14T02:43:04.459619Z 0 [Warning] InnoDB: New log files created, LSN=45790
2016-06-14T02:43:04.502310Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2016-06-14T02:43:04.567746Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: b7fa4c2e-31d9-11e6-983a-080027fff0b0.
2016-06-14T02:43:04.571117Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2016-06-14T02:43:04.575259Z 1 [Warning] root@localhost is created with an empty password ! Please consider switching off the --initialize-insecure option.

# 启动
[root@my57 data]# service mysql start
Starting MySQL. SUCCESS! 

# 利用原来的别分还原，发现还原了 但是缺少test1
mysql> source backup.sql

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.00 sec)

# 利用mysqlbinlog二进制恢复test1，这时上面的开始位置和结束位置就有用了
[root@my57 backup]# mysqlbinlog --start-position=567 --stop-position=869 /backup/bin-log.000001 |mysql -uroot -p
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
| test1              |
+--------------------+
6 rows in set (0.00 sec)
{% endhighlight %}

# xtrabackup备份参考另外文章

