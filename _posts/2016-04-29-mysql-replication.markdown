---
layout:     post
title:      "mysql主从复制"
subtitle:   "mysql 5.6 replication"
date:       2016-04-29 20:00:00
author:     "京城小表哥"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 数据库
    - MySql
    - DBA
    - Replication
---
## 1、主从服务器分别满足以下条件
#### （1）、从库版本大于或等于主库版本
#### （2）、初始化表，后台启动数据库
#### （3）、用于同步主从的用户需要有REPLICATION SLAVE权限
## 2、修改主服务器master
> vi /etc/my.cnf

```
[mysqld]
log-bin=mysql-bin   //[必须]启用二进制日志
server-id=67      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
binlog-do-db=repl    #需要同步的数据库，如果没有本行，即表示同步所有的数据库
binlog-ignore-db=mysql #被忽略的数据库
```
## 3、修改从服务器slave
> vi /etc/my.cnf

```
 [mysqld]
log-bin=mysql-bin   //[不是必须]启用二进制日志
server-id=68      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```

## 4、重启两台服务器的mysql
> /data/mysql/base/bin/mysqld_safe --defaults-file=/data/mysql/data/my.cnf --basedir=/data/mysql/base/ --datadir=/data/mysql/data/ --user=mysql &

## 5、在主服务器上建立帐户并授权slave

```
mysql>GRANT REPLICATION SLAVE ON *.* to 'replication'@'%' identified by 'replication'; //一般不用root帐号，&ldquo;%&rdquo;
```
表示所有客户端都可能连，只要帐号，密码正确，此处可用具体客户端IP代替，如10.1.3.68，加强安全。


## 6、登录主服务器的mysql，查询master的状态
```
mysql>show master status;
   +---------------------------+------------+---------------------+----------------------------+
   | File				| Position | Binlog_Do_DB | Binlog_Ignore_DB	|
   +---------------------------+------------+---------------------+----------------------------+
   | mysql-bin.000004	|	308	|             |                 |
   +---------------------------+------------+---------------------+----------------------------+
   1 row in set (0.00 sec)
   注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化
```
## 7、配置从服务器Slave
```
mysql>change master to master_host='10.1.3.67',master_user='replication',master_password='replication',master_log_file='mysql-bin.000004',master_log_pos=308;   //
```
注意不要断开，308数字前后无单引号。
```
Mysql>start slave;    //启动从服务器复制功能
```

## 8、检查从服务器复制功能状态

```
   mysql> show slave status\G
   *************************** 1. row ***************************

              Slave_IO_State: Waiting for master to send event
              Master_Host: 192.168.2.222  //主服务器地址
              Master_User: mysync   //授权帐户名，尽量避免使用root
              Master_Port: 3306    //数据库端口，部分版本没有此行
              Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
              Read_Master_Log_Pos: 600     //#同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos
              Relay_Log_File: ddte-relay-bin.000003
              Relay_Log_Pos: 251
              Relay_Master_Log_File: mysql-bin.000004
              Slave_IO_Running: Yes    //此状态必须YES
              Slave_SQL_Running: Yes     //此状态必须YES
                    ......
```
注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。

以上操作过程，主从服务器配置完成。

## 9、主从服务器测试

主服务器Mysql，建立数据库，并在这个库中建表插入一条数据：


### 著作权声明