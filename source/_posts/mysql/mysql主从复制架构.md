---
title: mysql主从复制架构
date: 2018-07-07 08:55:29   
categories: mysql   
toc: false  
top: 2
copyright: ture
password: 123456
---

# 1. 主从复制架构

## 1.1 为什么要做主从复制
- 在业务复杂的系统中，有这么一个情景，有一句sql语句需要锁表，导致暂时不能使用读的服务，那么就很影响运行中的业务，使用主从复制，让主库负责写，从库负责读，这样，即使主库出现了锁表的情景，通过读从库也可以保证业务的正常运作。
- 做数据的热备
- 架构的扩展。业务量越来越大，I/O访问频率过高，单机无法满足，此时做多库的存储，降低磁盘I/O访问的频率，提高单个机器的I/O性能。

## 1.2 mysql主从复制的原理
binlog: binary log，主库中保存更新事件日志的二进制文件。


## 1.3 主从复制架构的演变
- 一主一从

![mark](http://p7pxipt5l.bkt.clouddn.com/picture/180809/agfdaDJ1cH.png?imageslim)

- 一主多从

![mark](http://p7pxipt5l.bkt.clouddn.com/picture/180809/fICGBfGKAI.png?imageslim)

- 多级主从

![mark](http://p7pxipt5l.bkt.clouddn.com/picture/180809/GLljjiDhhd.png?imageslim)

- 双主

![mark](http://p7pxipt5l.bkt.clouddn.com/picture/180809/12G15E8Al7.png?imageslim)

- 多主一从（5.7之后开始支持）

![mark](http://p7pxipt5l.bkt.clouddn.com/picture/180809/BjG1BC8DA0.png?imageslim)

- 循环复制
> 对于循环数据库镜像，就是多个数据库A、B、C、D等，对其中任一个数据库的修改，都要同时镜像到其它的数据库里。  replicate-same-server-id = 0 

![mark](http://p7pxipt5l.bkt.clouddn.com/picture/180809/c9mD0HcFb0.png?imageslim)

# 2. MySQL主从复制搭建
在多个从库之间扩展负载以提高性能。在这种环境中，所有写入和更新在主库上进行。但是，读取可能发生在一个或多个从库上。该模型可以提高写入的性能（由于主库专用于更新），同时在多个从库上读取，可以大大提高读取速度。

参考：
- [mysql官网主从复制](https://dev.mysql.com/doc/refman/5.7/en/replication.html)
- [MySQL Replication 主从复制全方位解决方案](https://www.cnblogs.com/clsn/p/8150036.html)

## 2.1 准备工作
| 172.20.28.36                                                 | MySQL-master | yum install mysql mysql-server -y |
| ------------------------------------------------------------ | ------------ | --------------------------------- |
| 172.20.28.37                                                 | MySQL-slave1 | yum install mysql mysql-server -y |
| 172.20.28.38                                                 | MySQL-slave2 | yum install mysql mysql-server -y |
| 小结：mysql服务是yum安装的，配置文件：/etc/my.cnf  数据存放目录：/var/lib/mysql |              |                                   |
## 2.2 安装 MySQL
bin安装方式：https://dev.mysql.com/doc/refman/5.7/en/binary-installation.html

yum安装方式：https://dev.mysql.com/doc/refman/5.7/en/linux-installation-yum-repo.html

首先在两台机器上装上，保证正常启动，可以使用

## 2.3 修改主库和从库的配置文件
修改 my.cnf
> 配置 Master 以使用基于二进制日志文件位置的复制，必须启用二进制日志记录并建立唯一的服务器ID,否则则无法进行主从复制。

![mark](http://p7pxipt5l.bkt.clouddn.com/picture/180809/b9B28BJK0f.png?imageslim)

>  小结：
>  - 库开启binlog日志
>  - 主从server-id不同
>  - 从库服务器能连通主库

master中my.cnf：

```
[root@i-t27hedd8 ~]# cat /etc/my.cnf

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
server-id=1
log-bin=/var/lib/mysql/mysql-bin

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```


## 2.4 在master端操作
启动MySQL服务
```
$ service mysql.server start
或
systemctl start mysqld.service
```
登录MySQL
```
mysql -uroot -p
```
查看master端
```
mysql> show variables like "log_bin";

+---------------+-------+

| Variable_name | Value |

+---------------+-------+

| log_bin       | ON    |

+---------------+-------+

1 row in set (0.00 sec)

# 查看 Master-Server ， binlog File 文件名称和 Position值位置 并且记下来
mysql> show master status;

+------------------+----------+--------------+------------------+

| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |

+------------------+----------+--------------+------------------+

| mysql-bin.000001 |      341 |              |                  |

+------------------+----------+--------------+------------------+

1 row in set (0.00 sec)
```
**配置主库通信：** 查看 Master-Server ， binlog File 文件名称和 Position值位置 并且记下来

查看主服务器的运行状态
```
mysql>  show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      341 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
在主库创建复制用户
```
mysql> grant replication slave on *.* to 'oldboy123'@'172.20.28.%' identified by 'oldboy123';

Query OK, 0 rows affected (0.00 sec)

 

mysql> flush privileges;

Query OK, 0 rows affected (0.00 sec)
```
列出当前连接自己的从库列表
```
mysql> SHOW SLAVE HOSTS;
+-----------+--------+------+-------------------+-----------+
| Server_id | Host   | Port | Rpl_recovery_rank | Master_id |
+-----------+--------+------+-------------------+-----------+
|        10 | slave1 | 3306 |                 0 |         1 |
+-----------+--------+------+-------------------+-----------+
1 row in set (0.00 sec)
```
获取binlog文件列表
```
mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |      1190 |
+------------------+-----------+
```

## 2.5 分别在两台从库上操作

### 2.5.1 设置从库与主库通信信息
要设置从库与主库进行通信，进行复制，使用必要的连接信息配置从库在从库上执行以下语句

参数格式，请勿执行：
```
mysql> CHANGE MASTER TO
    ->     MASTER_HOST='master_host_name',
    ->     MASTER_USER='replication_user_name',
    ->     MASTER_PASSWORD='replication_password',
    ->     MASTER_LOG_FILE='recorded_log_file_name',
    ->     MASTER_LOG_POS=recorded_log_position;
```

```
# 分别在两台从库上操作
mysql> CHANGE MASTER TO
    -> MASTER_HOST='172.20.28.36',
    -> MASTER_USER='oldboy123',
    -> MASTER_PASSWORD='oldboy123',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=341;
Query OK, 0 rows affected, 2 warnings (0.02 sec)

mysql> flush privileges；
```
### 2.5.2 启动从服务器复制线程
启动从库复制线程
```
mysql> START SLAVE;
Query OK, 0 rows affected (0.00 sec)
```
停止从库复制线程
```
mysql> STOP SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

### 2.5.3 查看复制状态

```
mysql>  show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.20.28.36
                  Master_User: oldboy123
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 341
               Relay_Log_File: master2-relay-bin.000003
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
......
```
检查主从复制通信状态

```
# 必须都是 Yes
Slave_IO_State #从站的当前状态 
Slave_IO_Running： Yes #读取主程序二进制日志的I/O线程是否正在运行 
Slave_SQL_Running： Yes #执行读取主服务器中二进制日志事件的SQL线程是否正在运行。与I/O线程一样 
Seconds_Behind_Master #是否为0，0就是已经同步了
```
必须都是 Yes；如果不是原因主要有以下 4 个方面：
- 1、网络不通 
- 2、密码不对 
- 3、MASTER_LOG_POS 不对 ps 
- 4、mysql 的 auto.cnf server-uuid 一样（可能你是复制的mysql）
```
$ find / -name 'auto.cnf'
$ cat /var/lib/mysql/auto.cnf
[auto]
server-uuid=6b831bf3-8ae7-11e7-a178-000c29cb5cbc # 按照这个16进制格式，修改server-uuid，重启mysql即可
```

## 2.6 一些命令
查看主服务器的运行状态
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     1190 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
查看从服务器主机列表
```
mysql> show slave hosts;
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|         2 |      | 3306 |         1 | 6b831bf2-8ae7-11e7-a178-000c29cb5cbc |
+-----------+------+------+-----------+--------------------------------------+
```
获取binlog文件列表
```
mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |      1190 |
+------------------+-----------+
```
只查看第一个binlog文件的内容
```
mysql> mysql> show binlog events;
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                                                                                                                                                                  |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| mysql-bin.000001 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.19-log, Binlog ver: 4                                                                                                                                                                 |
| mysql-bin.000001 | 123 | Previous_gtids |         1 |         154 |                                                                                                                                                                                                       |
| mysql-bin.000001 | 420 | Anonymous_Gtid |         1 |         485 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                  |
| mysql-bin.000001 | 485 | Query          |         1 |         629 | GRANT REPLICATION SLAVE ON *.* TO 'replication'@'192.168.252.124'                                                                                                                                     |
| mysql-bin.000001 | 629 | Anonymous_Gtid |         1 |         694 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                  |
| mysql-bin.000001 | 694 | Query          |         1 |         847 | CREATE DATABASE `replication_wwww.ymq.io`                                                                                                                                                             |
| mysql-bin.000001 | 847 | Anonymous_Gtid |         1 |         912 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                  |
| mysql-bin.000001 | 912 | Query          |         1 |        1190 | use `replication_wwww.ymq.io`; CREATE TABLE `sync_test` (`id` int(11) NOT NULL AUTO_INCREMENT, `name` varchar(255) NOT NULL, PRIMARY KEY (`id`) ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
查看指定binlog文件的内容
```
mysql> mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                                                                                                                                                                  |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| mysql-bin.000001 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.19-log, Binlog ver: 4                                                                                                                                                                 |
| mysql-bin.000001 | 123 | Previous_gtids |         1 |         154 |                                                                                                                                                                                                       |
| mysql-bin.000001 | 420 | Anonymous_Gtid |         1 |         485 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                  |
| mysql-bin.000001 | 485 | Query          |         1 |         629 | GRANT REPLICATION SLAVE ON *.* TO 'replication'@'192.168.252.124'                                                                                                                                     |
| mysql-bin.000001 | 629 | Anonymous_Gtid |         1 |         694 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                  |
| mysql-bin.000001 | 694 | Query          |         1 |         847 | CREATE DATABASE `replication_wwww.ymq.io`                                                                                                                                                             |
| mysql-bin.000001 | 847 | Anonymous_Gtid |         1 |         912 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                  |
| mysql-bin.000001 | 912 | Query          |         1 |        1190 | use `replication_wwww.ymq.io`; CREATE TABLE `sync_test` (`id` int(11) NOT NULL AUTO_INCREMENT, `name` varchar(255) NOT NULL, PRIMARY KEY (`id`) ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 |
+------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
启动从库复制线程
```
mysql> START SLAVE;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```
停止从库复制线程
```
mysql> STOP SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

# 3. 复制实现细节分析
MySQL主从复制功能使用三个线程实现，一个在主服务器上，两个在从服务器上

每个主/从连接有三个线程。主服务器为每个当前连接的从服务器创建一个二进制日志转储线程，每个从服务器都有自己的I/O和SQL线程。

从服务器使用两个线程将读取更新与主服务器更新事件，并将其执行为独立任务。因此，如果语句执行缓慢，则读取语句的任务不会减慢。

## 3.1 Binlog转储线程
当从服务器与主服务器连接时，主服务器会创建一个线程将二进制日志内容发送到从服务器。 该线程可以使用 语句 SHOW PROCESSLIST(下面有示例介绍) 在服务器 sql 控制台输出中标识为Binlog Dump线程。

二进制日志转储线程获取服务器上二进制日志上的锁，用于读取要发送到从服务器的每个事件。一旦事件被读取，即使在将事件发送到从服务器之前，锁会被释放。

## 3.2 从服务器I/O线程
当在从服务器sql 控制台发出 START SLAVE语句时，从服务器将创建一个I/O线程，该线程连接到主服务器，并要求它发送记录在主服务器上的二进制更新日志。

从机I/O线程读取主服务器Binlog Dump线程发送的更新 （参考上面 Binlog转储线程 介绍），并将它们复制到自己的本地文件二进制日志中。

该线程的状态显示详情 Slave_IO_running 在输出端 使用 命令SHOW SLAVE STATUS
```
# 使用\G语句终结符,而不是分号,是为了，易读的垂直布局
mysql> SHOW SLAVE STATUS\G
```

## 3.3 从服务器SQL线程
从服务器创建一条SQL线程来读取由主服务器I/O线程写入的二级制日志，并执行其中包含的事件。

## 3.4 查看复制细节
在 Master 主服务器，查看Binlog转储线程
```
mysql>  SHOW FULL PROCESSLIST\G
*************************** 1. row ***************************
     Id: 22
   User: repl
   Host: node2:39114
     db: NULL
Command: Binlog Dump
   Time: 4435
  State: Master has sent all binlog to slave; waiting for more updates
   Info: NULL
```
> - Id: 22是Binlog Dump服务连接的从站的复制线程 
> - Host: node2:39114 是从服务，主机名 级及端口 
> - State: 信息表示所有更新都已同步发送到从服务器，并且主服务器正在等待更多更新发生。 

在 Slave 从服务器 ，查看两个线程的更新状态
```
mysql> SHOW PROCESSLIST\G
*************************** 1. row ***************************
     Id: 6
   User: system user
   Host: 
     db: NULL
Command: Connect
   Time: 6810
  State: Waiting for master to send event
   Info: NULL
*************************** 2. row ***************************
     Id: 7
   User: system user
   Host: 
     db: NULL
Command: Connect
   Time: 3069
  State: Slave has read all relay log; waiting for more updates
   Info: NULL
```
> - Id: 6是与主服务器通信的I/O线程 
> - Id: 7是正在处理存储在中继日志中的更新的SQL线程

如果在主服务器上在设置的超时，时间内 Binlog Dump线程没有活动，则主服务器会和从服务器断开连接。超时取决于的服务器系统变量 值 net_write_timeout(在中止写入之前等待块写入连接的秒数，默认10秒)和 net_retry_count;(如果通信端口上的读取或写入中断，请在重试次数，默认10次) 设置 [服务器系统变量](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html)

# 4 更多常见主从复制问题
https://dev.mysql.com/doc/refman/5.7/en/faqs-replication.html

























