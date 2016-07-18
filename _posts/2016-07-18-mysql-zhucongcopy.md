---
date: 2016-07-18 23:09:25+08:00
layout: post
title: Mysql之主从复制
categories: linux
tags: mysql 主从复制 mysql主从复制 二进制日志 中继日志
---
# 节点一 #
修改配置文件设置唯一ID开起二进制日志

	[root@node1 ~]# vim /etc/my.cnf	增加以下内容
		[mysqld]
		log-bin=master_bin	开起二进制日志
		server_id=1		给主节点一个唯一的ID号
		innodb_file_per_table=on	innodb开起独立表空间
		skip_name_resolve=on	开启跳过主机名反解

启动服务创建有远程复制权限的账户

	[root@node1 ~]# service mariadb start
	[root@node1 ~]# mysql
	MariaDB [(none)]> show global variables like '%log%';	查看二进制日志log_bin是否开启了
	MariaDB [(none)]> show global variables like '%server%';	查看DI号是否为1
	MariaDB [(none)]> show master logs;	查看主节点二进制日志的位置，从节点从主节点最后一个日志的位置开始复制
	MariaDB [(none)]> grant replication slave,replication client on *.* to 'copy'@'192.168.%.%' identified by 'passwd';		创建并授权一个远程复制账号copy密码为passwd
	MariaDB [(none)]> flush privileges;	刷新用户权限

# 节点二 #
修改配置文件设置唯一ID开起中继日志

	[root@node2 ~]# vim /etc/my.cnf
		relay_log=relay_log	开起中继日志
		relay-log-index=relay-log.index	
		server_id=2		同样的也需要设置唯一的ID号
		innodb_file_per_table=on
		skip_name_resolve=on

	[root@node2 ~]# service mariadb start
	[root@node2 ~]# mysql
	MariaDB [(none)]> show global variables like '%log%';	查看中继日志relay_log是否开起
	MariaDB [(none)]> show global variables like '%server%';	查看ID号是否为2
	主节点为192.168.1.107，远程复制账号为copy,密码为passwd,复制二进制日志的起始位置为000003的245处
	MariaDB [(none)]> change master to master_host='192.168.1.107',master_user='copy',master_password='passwd',master_log_file='master_bin.000003',master_log_pos=245;
	MariaDB [(none)]> start slave;	启动从节点复制线程


	MariaDB [(none)]> show slave status\G;
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 192.168.1.107
	                  Master_User: copy
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: master_bin.000003
	          Read_Master_Log_Pos: 491
	               Relay_Log_File: relay_log.000003
	                Relay_Log_Pos: 776
	        Relay_Master_Log_File: master_bin.000003
	             Slave_IO_Running: Yes	这两项必须为yes
	            Slave_SQL_Running: Yes	这两项必须为yes
	              Replicate_Do_DB: 
	          Replicate_Ignore_DB: 
	           Replicate_Do_Table: 
	       Replicate_Ignore_Table: 
	      Replicate_Wild_Do_Table: 
	  Replicate_Wild_Ignore_Table: 
	                   Last_Errno: 0
	                   Last_Error: 
	                 Skip_Counter: 0
	          Exec_Master_Log_Pos: 491
	              Relay_Log_Space: 1064
	              Until_Condition: None
	               Until_Log_File: 
	                Until_Log_Pos: 0
	           Master_SSL_Allowed: No
	           Master_SSL_CA_File: 
	           Master_SSL_CA_Path: 
	              Master_SSL_Cert: 
	            Master_SSL_Cipher: 
	               Master_SSL_Key: 
	        Seconds_Behind_Master: 0
	Master_SSL_Verify_Server_Cert: No
	                Last_IO_Errno: 0
	                Last_IO_Error: 
	               Last_SQL_Errno: 0
	               Last_SQL_Error: 
	  Replicate_Ignore_Server_Ids: 
	             Master_Server_Id: 1
	1 row in set (0.00 sec)

# 注意 #

如果`Slave_IO_Running`不为yes的解决办法

如：ERROR 1201 (HY000)

	MariaDB [(none)]> slave stop;	停止从节点
	MariaDB [(none)]> reset slave;	重新设置从节点
查找设置有问题的地方重新给从节点授权

	MariaDB [(none)]> change master to master_host='192.168.1.107',master_user='copy',master_password='passwd',master_log_file='master_bin.000003',master_log_pos=245;
	MariaDB [(none)]> start slave;	启动从节点
	MariaDB [(none)]> show slave status\G;	查看状态

注意从节点上一定不能进行写操作

# 验证 #

主节点

	MariaDB [(none)]> create database msdb;
	MariaDB [msdb]> create table xx (id int(4) not null auto_increment,name varchar(30) not null,primary key(id)) engine=innodb charset=utf8;
	MariaDB [msdb]> insert into xx (id,name) values (1,'king');

从节点

	MariaDB [(none)]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| msdb               |
	| mysql              |
	| performance_schema |
	| test               |
	+--------------------+
	MariaDB [(none)]> use msdb;
	MariaDB [msdb]> show tables;
	+----------------+
	| Tables_in_msdb |
	+----------------+
	| xx             |
	+----------------+
	MariaDB [msdb]> select * from xx;
	+----+------+
	| id | name |
	+----+------+
	|  1 | king |
	+----+------+








