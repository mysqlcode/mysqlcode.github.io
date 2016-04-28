---
layout:     post
title:      "mysql数据库安装"
subtitle:   "mysql 5.6 数据库安装"
date:       2016-04-27 20:00:00
author:     "京城小表哥"
header-img: "img/post-bg-re-vs-ng2.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 数据库
    - MySql
    - DBA
---

> 这篇文章记录安装mysql的每一个细节[适合小白](http://jcxbg.github.io/)。


## 1、编译安装

#### (1) 安装相关依赖包

> yum install bison zlib-devel openssl-devel ncurses-devel libxml2-devel curl-devel libjpeg-devel libpng-devel freetype-devel ntp gcc gcc-c++ openldap-devel lrzsz make cmake dialog –y

#### (2) 创建目录

```
mkdir -p /data/mysql/base/
mkdir -p /data/mysql/data/
mkdir -p /data/soft/
mkdir -p /data/mysql/data/tmp
mkdir -p /data/mysql/data/log/binlog/
mkdir -p /data/mysql/data/log/relaylog/
mkdir -p /data/mysql/data/log/errlog/
mkdir -p /data/mysql/data/log/slowlog/
mkdir -p /data/mysql/data/log/mysql/
```

#### (3) 创建mysql用户

```
/usr/sbin/groupadd mysql
/usr/sbin/useradd -s /sbin/nologin -M  -g mysql mysql
```

#### (4) 下载软件包

```
cd /data/soft/
wget https://www.percona.com/downloads/Percona-Server-5.6/Percona-Server-5.6.25-73.1/source/tarball/percona-server-5.6.25-73.1.tar.gz
```

#### (5) 编译安装

```
tar -zxvf percona-server-5.6.25-73.1.tar.gz
cd percona-server-5.6.25-73.1
cmake . -DCMAKE_INSTALL_PREFIX=/data/mysql/base/ -DMYSQL_DATADIR=/data/mysql/data/ -DMYSQL_UNIX_ADDR=/data/mysql/data/mysql.sock -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci  -DWITH_MYISAM_STORAGE_ENGINE=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 -DENABLED_LOCAL_INFILE=1 -DMYSQL_USER=mysql
make
make install
```

#### (6) 编写配置文件

> vim /data/mysql/data/my.cnf

```
[client]
port =3306
socket =/data/mysql/data/mysql.sock

[mysqld]
#bind-address= 10.1.3.68
port =3306
basedir =/data/mysql/base/
socket =/data/mysql/data/mysql.sock
pid-file =/data/mysql/data/mysql.pid
datadir =/data/mysql/data/

tmpdir =/data/mysql/data/tmp
slave-load-tmpdir =/data/mysql/data/tmp
#skip lever
skip-name-resolve
skip-symbolic-links
skip-external-locking
skip-slave-start

#thread pool
thread_handling=pool-of-threads
thread_pool_oversubscribe = 10

#thread level
table_open_cache = 5000
#lower_case_table_names=1

#############connect############
back_log = 200
max_connections = 5000
max_connect_errors = 10000
#open_files_limit = 10240

server_id = 21
read_only = 1
#log-slave-updates

##############timeout###########
connect-timeout = 10
wait-timeout = 800
interactive-timeout = 800
slave-net-timeout = 60
net_read_timeout = 30
net_write_timeout = 60
net_retry_count = 10
net_buffer_length = 16384
max_allowed_packet = 64M

#################cache #############
thread_stack = 192K
thread_cache_size = 500
thread_concurrency = 16

#qcache settings
query_cache_size = 512M
query_cache_limit = 2M
query_cache_min_res_unit = 2K

#default settings
#time zone
default-time-zone = system
character-set-server = utf8
default-storage-engine = InnoDB

#tmp & heap 
tmp_table_size = 512M
max_heap_table_size = 512M

binlog_format = MIXED
log-bin =/data/mysql/data/log/binlog/mysql-bin
log-bin-index =/data/mysql/data/log/binlog/mysql-bin.index
relay-log =/data/mysql/data/log/relaylog/relay-log
relay-log-index =/data/mysql/data/log/relaylog/relay-log.index

#warning & error log
log-warnings = 1
log_error =/data/mysql/data/log/errlog/mysql.err

log-output = FILE

#slow query log
slow_query_log = 1
long-query-time = 0.5
slow_query_log_file =/data/mysql/data/log/slowlog/slow.log
#log-queries-not-using-indexes
#log-slow-slave-statements

general_log = 0
general_log_file =/data/mysql/data/log/mysql/mysql.log
max_binlog_size = 1G
max_relay_log_size = 1G

#if use auto-ex, set to 0
relay-log-purge = 1

#max binlog keeps days
expire_logs_days = 5

binlog_cache_size = 1M

#replication
#slave_skip_errors=all

key_buffer_size = 256M
sort_buffer_size = 128M
read_buffer_size = 128M
join_buffer_size = 128M
read_rnd_buffer_size = 128M
bulk_insert_buffer_size = 64M
myisam_sort_buffer_size = 64M
myisam_max_sort_file_size = 5G
myisam_repair_threads = 1
myisam_recover

group_concat_max_len = 64K

transaction_isolation = REPEATABLE-READ

innodb_file_per_table

#############compress################
innodb_file_format = Barracuda

########################################

innodb_additional_mem_pool_size = 1024M
innodb_buffer_pool_size = 1G
innodb_data_home_dir =/data/mysql/data
innodb_data_file_path = ibdata1:1024M:autoextend
innodb_undo_directory = /data/mysql/data
innodb_undo_tablespaces = 0

################innodb threads############
innodb_read_io_threads = 8
innodb_write_io_threads = 8
innodb_purge_threads = 1

########################################

innodb_thread_concurrency = 16
innodb_flush_log_at_trx_commit = 2

innodb_log_buffer_size = 16M
innodb_log_file_size = 1024M
innodb_log_files_in_group = 4
innodb_log_group_home_dir =/data/mysql/data

innodb_max_dirty_pages_pct = 75
innodb_lock_wait_timeout = 50
innodb_flush_method = O_DIRECT

#########################################
innodb_buffer_pool_instances = 16
innodb_change_buffering = all
innodb_adaptive_flushing = 1
#innodb_io_capacity = 5000
innodb_io_capacity = 3000
innodb_old_blocks_time = 1000
innodb_stats_on_metadata = 0

#########################################
old-passwords = 0
#sql_mode = PIPES_AS_CONCAT,ANSI_QUOTES,STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION
innodb_stats_persistent = 0

[mysqldump]
quick
max_allowed_packet = 64M

[mysql]
no-auto-rehash
default-character-set = utf8
connect-timeout = 3

[myisamchk]
key_buffer_size = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout
```

#### (7) 安装数据库

```
chown -R mysql:mysql /data/mysql/
/data/mysql/base/scripts/mysql_install_db --defaults-file=/data/mysql/data/my.cnf --basedir=/data/mysql/base --datadir=/data/mysql/data --user=mysql
```

####（8）启动数据库

```
/data/mysql/base/bin/mysqld_safe --defaults-file=/data/mysql/data/my.cnf --basedir=/data/mysql/base --datadir=/data/mysql/data --user=mysql &
```
###（9）登录数据库

```
/data/mysql/base/bin/mysql -u root -p --socket=/data/mysql/data/mysql.sock --port=3306
```
###（10）设置密码

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY 'rootpassword' WITH GRANT OPTION;
delete from mysql.user where password='';
flush privileges;
```
- **基础知识要扎实
- **适合小白



### 著作权声明

