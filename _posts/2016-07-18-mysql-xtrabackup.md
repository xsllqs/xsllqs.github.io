---
date: 2016-07-18 23:05:25+08:00
layout: post
title: mysql备份之xtrabackup（建议用来备份innodb）
categories: linux
tags: mysql mysql备份 xtrabackup innodb
---
下载地址：https://www.percona.com/downloads/XtraBackup/

安装xtrabackup

	[root@node1 ~]# yum install percona-xtrabackup

# 完全备份 #

## 节点一 ##

修改配置文件，设置为每张表单独一个表空间，此项必须在安装数据库的时候就设置

	[root@node1 ~]# vim /etc/my.cnf
	[mysqld]
	innodb_file_per_table=ON

创建备份目录

	[root@node1 ~]# mkdir /backpus/
备份

	[root@node1 ~]# innobackupex --user=root /backpus/

复制给节点2

	[root@node1 ~]# scp -r /backpus/2016-07-13_20-27-04 192.168.1.114:/root/

## 节点二 ##
（节点二的mysql安装后不要启动，启动后因生成有初始化文件无法还原）

	[root@node2 ~]# yum install percona-xtrabackup

把备份文件移动到/backups目录下

	[root@node2 ~]# mkdir /backups/
	[root@node2 ~]# mv 2016-07-13_20-27-04/ /backups/

对备份文件进行整理

	[root@node2 ~]# innobackupex --apply-log /backups/2016-07-13_20-27-04/

还原

	[root@node2 ~]# innobackupex --copy-back /backups/2016-07-13_20-27-04/

修改文件权限

	[root@node2 ~]# chown -R mysql:mysql /var/lib/mysql/*

# 增量备份 #

修改数据

	[root@node1 ~]# mysql
	MariaDB [(none)]> use hellodb;
	MariaDB [hellodb]> create table xxoo2 (id int);
	MariaDB [hellodb]> insert into xxoo2 values (1),(10),(83);

对之前完全备份的文件进行增量备份

	[root@node1 ~]# innobackupex --incremental /backpus/ --incremental-basedir=/backpus/2016-07-13_20-27-04

对完全备份做只读，为增量和完全合并做准备

	[root@node1 ~]# innobackupex --apply-log --redo-only /backpus/2016-07-13_20-27-04/

合并增量到完全中

	[root@node1 ~]# innobackupex --apply-log --redo-only /backpus/2016-07-13_20-27-04/ --incremental-dir=/backpus/2016-07-13_23-13-25/

查看增量备份文件

	[root@node1 ~]# less /backpus/2016-07-13_23-13-25/xtrabackup_checkpoints 
	backup_type = incremental
	from_lsn = 1642047
	to_lsn = 1646912
	last_lsn = 1646912
	compact = 0

查看完全备份文件

	[root@node1 ~]# less /backpus/2016-07-13_20-27-04/xtrabackup_checkpoints
	backup_type = full-prepared
	from_lsn = 0
	to_lsn = 1646912
	last_lsn = 1646912
	compact = 0

之后如果有新的增量备份文件还可以继续在完全备份文件上合并
还原时将完全备份文件拿去还原即可

注意：mysql的访问权限，我操作过程中多次出现错误，都是在mysql数据库的属主和属组权限出现的问题。





