---
layout:     post
title:      "mysql数据库备份与恢复"
subtitle:   "mysql 5.6 数据库备份与恢复"
date:       2016-04-28 11:00:00
author:     "京城小表哥"
header-img: "img/mmmmmm.jpg"
header-mask: 0.3
catalog:    true
tags:
    - MySql
    - DBA
    - backup
    - Linux
---


## 1、mysqldump


#### （1）应用mysqldump备份

> mysqldump -u root -p password --databases databasename | gzip > /file/databasename.sql.gz

#### （2）应用mysqldump恢复

> mysql -u root -p < /file/databasesname.sql


#### （3）备份脚本

```
#!/bin/sh
#设置用户名和密码
v_user="root"
v_password="password"

#mysql安装全路径
MysqlDir=/usr/bin

#备份数据库
database="hive_mata"

#设置备份路径，创建备份文件夹
BackupDir=/var/lib/mysqlbackup/hive_mata
Full_Backup=$BackupDir/Full_backup

mkdir -p $Full_Backup/$(date +%Y%m%d)

#开始备份,记录备份开始时间
echo '=========='$(date +"%Y-%m-%d %H:%M:%S")'=========='"备份开始">>$Full_Backup/full_buckup.log

$MysqlDir/mysqldump -u$v_user -p$v_password   --databases $database>$Full_Backup/$(date +%Y%m%d)/full_backup.sql

#压缩备份文件
gzip $Full_Backup/$(date +%Y%m%d)/full_backup.sql
cp -rf $Full_Backup/$(date +%Y%m%d) /misc/backup/hdn8/.
rm -rf $Full_Backup/$(date +%Y%m%d --date='5 days ago')
rm -rf /misc/backup/hdn8/$(date +%Y%m%d --date='5 days ago')

echo '=========='$(date +"%Y-%m-%d %H:%M:%S")'=========='"备份完成">>$Full_Backup/full_buckup.log
```

## 2、xtrabackup

#### （1）安装xtrabackup

```
yum install perl-DBI -y
yum install perl-DBD-MySQL -y
yum install perl-Time-HiRes -y
yum install perl-IO-Socket-SSL –y

wget http://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.2.12/binary/tarball/percona-xtrabackup-2.2.12-Linux-x86_64.tar.gz
tar -zxvf percona-xtrabackup-2.2.12-Linux-x86_64.tar.gz
cp percona-xtrabackup-2.2.12-Linux-x86_64/bin/* /usr/bin/.

```


#### （2）完全备份

```
/usr/bin/innobackupex --rsync --parallel=3 --defaults-file=/etc/my.cnf  --slave-info  --host=10.10.0.0 --user=backup --password=mysqlpassword --port=3306 /data/backup/ > /data/backup/tmp.log 2>&1
```

#### （3）增量备份

```
/usr/bin/innobackupex --rsync --parallel=3  --defaults-file=/etc/my.cnf --slave-info  --host=10.10.0.0 --user=backup --password=mysqlpassword --port=3306 --incremental /INCRBACKUP_DIR/LATEST_FULL_BACKUP --incremental-basedir /INCRBACKUP_DIR/LATEST_FULL_BACKUP/date  >  /tmp/innobackupex.incr.log 2>&1
```

#### （4）数据恢复

```
完全恢复：
innobackupex  --user=root  --password=""  --apply-log --redo-only /my/bak/full/2015-11-09_01-30-05/

增量恢复：
innobackupex  --user=root  --password=""  --apply-log --redo-only /my/bak/full/2015-11-09_01-30-05/
innobackupex  --user=root  --password=""  --apply-log --redo-only /my/bak/full/2015-11-09_01-30-05/ --incremental-dir=/my/bak/inc/2015-11-09_01-30-05/2015-11-10_01-30-02/
innobackupex  --user=root  --password=""  --apply-log --redo-only /my/bak/full/2015-11-09_01-30-05/ --incremental-dir=/my/bak/inc/2015-11-09_01-30-05/2015-11-11_01-30-08/
innobackupex  --user=root  --password=""  --apply-log --redo-only /my/bak/full/2015-11-09_01-30-05/ --incremental-dir=/my/bak/inc/2015-11-09_01-30-05/2015-11-12_01-30-01/
```

#### （5）备份脚本

```
#!/bin/bash

#on xtrabackup 2.2.10
# 第一次运行脚本的时候会检查是否有完全备份,否则先创建一个全库备份
# 当你再次运行它的时候，它会根据脚本中的设定来基于之前的全备或增量备份进行增量备份
#by 小表哥

INNOBACKUPEX_PATH=innobackupex  #INNOBACKUPEX的命令
INNOBACKUPEXFULL=/usr/bin/$INNOBACKUPEX_PATH  #INNOBACKUPEX的命令路径

#mysql目标服务器以及用户名和密码

#DB备份信息
MYSQLDIR=/usr/bin
MYSQL=$MYSQLDIR/mysql
MYSQL_ADMIN=$MYSQLDIR/mysqladmin
MYSQL_USER=backup
MYSQL_PASSWORD=password
#HOST=`ifconfig  | grep 'inet addr:'| grep -v '127.0.0.1' | cut -d: -f2 | awk '{ print $1}'`
HOST=10.10.10.10
HOSTNAME=`hostname`
PORT=3306
MY_CNF=/my/data/percona/my.cnf

MYSQL_CMD="--host=$HOST --user=$MYSQL_USER --password=$MYSQL_PASSWORD --port=$PORT"  

BACKUP_DIR=/misc/backup # 备份的主目录
FULLBACKUP_DIR=$BACKUP_DIR/full # 全库备份的目录
INCRBACKUP_DIR=$BACKUP_DIR/incre # 增量备份的目录

FULLBACKUP_INTERVAL=432000 # 全库备份的间隔周期，时间：秒,暂放5天
KEEP_FULLBACKUP=2 # 至少保留几个全库备份

#要删除的全备文件
DT=`date +%Y-%m-%d -d "5 day ago"`

#开始时间
STARTED_TIME=`date +%s`

TMPLOG="/tmp/innobackupex.$$.log"

#备份时间-日志
LOGFILEDATE=`date +%Y%m%d%H%M`.txt

#############################################################################

# 显示错误并退出

#############################################################################

error()
{
    echo "$1" 1>&2
    exit 1
}


# 检查执行环境

if [ ! -x $INNOBACKUPEXFULL ]; then
  error "$INNOBACKUPEXFULL未安装或未链接到/usr/bin."
fi


if [ ! -d $BACKUP_DIR ]; then
  error "备份目标文件夹:$BACKUP_DIR不存在."
fi


mysql_status=`netstat -nl | awk 'NR>2{if ($4 ~ /.*:3306/) {print "Yes";exit 0}}'`

if [ "$mysql_status" != "Yes" ];then
    error "MySQL 没有启动运行."
fi


if ! `echo 'exit' | $MYSQL -s $MYSQL_CMD` ; then
 error "提供的数据库用户名或密码不正确!"
fi

# 备份的头部信息

echo "----------------------------"
echo
echo "$0: MySQL备份脚本"
echo "开始于: `date +%F' '%T' '%w`"
echo 

#新建全备和差异备份的目录

mkdir -p $FULLBACKUP_DIR
mkdir -p $INCRBACKUP_DIR


#查找最新的完全备份
LATEST_FULL_BACKUP=`find $FULLBACKUP_DIR -mindepth 1 -maxdepth 1 -type d -printf "%P\n" | sort -nr | head -1`

 
# 查找最近修改的最新备份时间

LATEST_FULL_BACKUP_CREATED_TIME=`stat -c %Y $FULLBACKUP_DIR/$LATEST_FULL_BACKUP`

#expr_1=`$LATEST_FULL_BACKUP_CREATED_TIME + $FULLBACKUP_INTERVAL + 5`
#tem123=`expr $LATEST_FULL_BACKUP_CREATED_TIME + $FULLBACKUP_INTERVAL + 5`
#如果全备有效进行增量备份否则执行完全备份
#tem123=`expr $LATEST_FULL_BACKUP_CREATED_TIME + $FULLBACKUP_INTERVAL + 5`
if [ "$LATEST_FULL_BACKUP" -a `expr $LATEST_FULL_BACKUP_CREATED_TIME + $FULLBACKUP_INTERVAL + 5` -ge $STARTED_TIME ] ; then
#if [ "$LATEST_FULL_BACKUP" -a $tem123 -ge $STARTED_TIME ] ; then
	# 如果最新的全备未过期则以最新的全备文件名命名在增量备份目录下新建目录
	echo -e "完全备份$LATEST_FULL_BACKUP未过期,将根据$LATEST_FULL_BACKUP名字作为增量备份基础目录名"
	echo "					   "
	NEW_INCRDIR=$INCRBACKUP_DIR/$LATEST_FULL_BACKUP
	mkdir -p $NEW_INCRDIR

	# 查找最新的增量备份是否存在.指定一个备份的路径作为增量备份的基础
	LATEST_INCR_BACKUP=`find $NEW_INCRDIR -mindepth 1 -maxdepth 1 -type d -printf "%P\n"  | sort -nr | head -1`
		if [ ! $LATEST_INCR_BACKUP ] ; then
			INCRBASEDIR=$FULLBACKUP_DIR/$LATEST_FULL_BACKUP
			echo -e "增量备份将以$INCRBASEDIR作为备份基础目录"
			echo "					   "
		else
			INCRBASEDIR=$INCRBACKUP_DIR/${LATEST_FULL_BACKUP}/${LATEST_INCR_BACKUP}
			echo -e "增量备份将以$INCRBASEDIR作为备份基础目录"
			echo "					   "
		fi

	echo "使用$INCRBASEDIR作为基础本次增量备份的基础目录."
	#$INNOBACKUPEXFULL --defaults-file=$MY_CNF --use-memory=4G $MYSQL_CMD --incremental $NEW_INCRDIR --incremental-basedir $INCRBASEDIR > $TMPLOG 2>&1
	$INNOBACKUPEXFULL --rsync --parallel=3  --defaults-file=$MY_CNF  --slave-info --safe-slave-backup $MYSQL_CMD --incremental $NEW_INCRDIR --incremental-basedir $INCRBASEDIR > $TMPLOG 2>&1

	#保留一份备份的详细日志

	cat $TMPLOG>$BACKUP_DIR/$LOGFILEDATE

	if [ -z "`tail -1 $TMPLOG | grep 'innobackupex: completed OK!'`" ] ; then
	 echo "$INNOBACKUPEX命令执行失败:"; echo
	 echo -e "---------- $INNOBACKUPEX_PATH错误 ----------"
#    cat $TMPLOG
#	 rm -f $TMPLOG
	 exit 1
	fi

	THISBACKUP=`awk -- "/Backup created in directory/ { split( \\\$0, p, \"'\" ) ; print p[2] }" $TMPLOG`
#	rm -f $TMPLOG


	echo -n "数据库成功备份到:$THISBACKUP"
#mail_file=`tail -n 50 $TMPLOG`
	service sendmail restart
	tail -n 10 $TMPLOG | mail -v -s "$HOSTNAME backup_inc infomation" huangwenyi@babytree-inc.com
	echo

	# 提示应该保留的备份文件起点

	LATEST_FULL_BACKUP=`find $FULLBACKUP_DIR -mindepth 1 -maxdepth 1 -type d -printf "%P\n" | sort -nr | head -1`

	NEW_INCRDIR=$INCRBACKUP_DIR/$LATEST_FULL_BACKUP

	LATEST_INCR_BACKUP=`find $NEW_INCRDIR -mindepth 1 -maxdepth 1 -type d -printf "%P\n"  | sort -nr | head -1`

	RES_FULL_BACKUP=${FULLBACKUP_DIR}/${LATEST_FULL_BACKUP}

	RES_INCRE_BACKUP=`dirname ${INCRBACKUP_DIR}/${LATEST_FULL_BACKUP}/${LATEST_INCR_BACKUP}`

	echo
	echo -e '\e[31m NOTE:---------------------------------------------------------------------------------.\e[m' #红色
	echo -e "必须保留$KEEP_FULLBACKUP份全备即全备${RES_FULL_BACKUP}和${RES_INCRE_BACKUP}目录中所有增量备份."
	echo -e '\e[31m NOTE:---------------------------------------------------------------------------------.\e[m' #红色
	echo

else
	echo  "*********************************"
	echo -e "正在执行全新的完全备份...请稍等..."
	echo  "*********************************"
	#$INNOBACKUPEXFULL --defaults-file=$MY_CNF --slave-info --use-memory=4G  $MYSQL_CMD $FULLBACKUP_DIR > $TMPLOG 2>&1 
	$INNOBACKUPEXFULL --rsync --parallel=3 --defaults-file=$MY_CNF  --slave-info  $MYSQL_CMD $FULLBACKUP_DIR > $TMPLOG 2>&1 
	#保留一份备份的详细日志

	cat $TMPLOG>$BACKUP_DIR/$LOGFILEDATE


	if [ -z "`tail -1 $TMPLOG | grep 'innobackupex: completed OK!'`" ] ; then
	 echo "$INNOBACKUPEX命令执行失败:"; echo
	 echo -e "---------- $INNOBACKUPEX_PATH错误 ----------"
	 cat $TMPLOG
#	 rm -f $TMPLOG
	 exit 1
	fi

	 
	THISBACKUP=`awk -- "/Backup created in directory/ { split( \\\$0, p, \"'\" ) ; print p[2] }" $TMPLOG`
#rm -f $TMPLOG
#mail_file=`tail -n 50 $TMPLOG`
	echo -n "数据库成功备份到:$THISBACKUP"
#	mail -s "$HOST backup infomation" huangwenyi@babytree-inc.com < $mail_file
	service sendmail restart
	tail -n 10 $TMPLOG | mail -v -s "$HOSTNAME backup_all infomation" huangwenyi@babytree-inc.com
	echo

	# 提示应该保留的备份文件起点

	LATEST_FULL_BACKUP=`find $FULLBACKUP_DIR -mindepth 1 -maxdepth 1 -type d -printf "%P\n" | sort -nr | head -1`

	RES_FULL_BACKUP=${FULLBACKUP_DIR}/${LATEST_FULL_BACKUP}

	echo
	echo -e '\e[31m NOTE:---------------------------------------------------------------------------------.\e[m' #红色
	echo -e "无增量备份,必须保留$KEEP_FULLBACKUP份全备即全备${RES_FULL_BACKUP}."
	echo -e '\e[31m NOTE:---------------------------------------------------------------------------------.\e[m' #红色
	echo

fi
#rm -f $TMPLOG
/usr/bin/find /tmp/innobackupex* -mtime +5 -exec rm -rf {} \;
#/usr/bin/find $LOGFILEDATE -mitme +5


#删除过期的全备
#这里手动删除15天以前的
#/usr/bin/find $FULLBACKUP_DIR/ -type d -mtime 15 rm -rf {} \;
for subdir in `ls $FULLBACKUP_DIR`;
do
    if [ "${subdir}" \< "${DT}" ];
		 then 
		rm -rf $FULLBACKUP_DIR/$subdir >/dev/null
		echo "The directory $FULLBACKUP_DIR/$subdir has been removed."
	fi
done

#删除过期的差备
for subdir in `ls $INCRBACKUP_DIR`;
do
	if [ "${subdir}" \< "${DT}" ];
		then
	rm -rf $INCRBACKUP_DIR/$subdir >/dev/null
	echo "The directory $INCRBACKUP_DIR/$subdir has been removed."
	fi
done

:<<eof
echo -e "find expire backup file...........waiting........."
echo -e "寻找过期的全备文件并删除">>$BACKUP_DIR/$LOGFILEDATE
for efile in $(/usr/bin/find $FULLBACKUP_DIR/ -mtime +6)
do
	if [ -d ${efile} ]; then
	rm -rf "${efile}"
	echo -e "删除过期全备文件:${efile}" >>$BACKUP_DIR/$LOGFILEDATE
	elif [ -f ${efile} ]; then
	rm -rf "${efile}"
	echo -e "删除过期全备文件:${efile}" >>$BACKUP_DIR/$LOGFILEDATE
	fi;
	
done

if [ $? -eq "0" ];then
   echo
   echo -e "未找到可以删除的过期全备文件"
fi



echo
echo "完成于: `date +%F' '%T' '%w`"
exit 0
eof
```


### 著作权声明
