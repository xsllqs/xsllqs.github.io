---
date: 2017-08-20 10:22:52+08:00
layout: post
title: zabbix高可用部署
categories: linux
tags: zabbix 高可用 zabbix-proxy
---

4台主机：

	192.168.13.54
	192.168.13.55
	192.168.13.56
	192.168.13.57
	
2个vip：

	192.168.13.59
	192.168.13.60

启动mysql时需要同时启动keepalived

	service mysql start

导出zabbix库

	mysqldump -uzabbix -pZabbix1344 zabbix > /home/app/zabbix.sql
	
导入zabbix库

	nohup mysql -uzabbix -pZabbix1344 zabbix < /home/app/zabbix.sql &

# 一、数据库高可用 #

192.168.13.56和192.168.13.57的mysql的root密码为zabbix@1344

安装的mysql-server版本为5.7

	dpkg -l | grep mysql
	apt-get install mysql-server
	service mysql stop

	vim /etc/mysql/my.cnf
	[mysqld]
	skip-name-resolve
	max_connections = 1000
	bind-address = 0.0.0.0
	server-id = 1	#192.168.13.56设置为1，192.168.13.57设置为2
	log-bin = /var/log/mysql/mysql-bin.log
	binlog-ignore-db = mysql,information_schema
	auto-increment-increment = 2
	auto-increment-offset = 1
	slave-skip-errors = all
	user            = mysql
	pid-file        = /var/run/mysqld/mysqld.pid
	socket          = /var/run/mysqld/mysqld.sock
	port            = 3306
	basedir         = /usr
	datadir         = /var/lib/mysql
	tmpdir          = /tmp
	lc-messages-dir = /usr/share/mysql
	skip-external-locking
	key_buffer_size         = 16M
	max_allowed_packet      = 16M
	thread_stack            = 192K
	thread_cache_size       = 8
	myisam-recover-options  = BACKUP
	query_cache_limit       = 1M
	query_cache_size        = 16M
	log_error = /var/log/mysql/error.log
	expire_logs_days        = 5
	max_binlog_size   = 100M

	[mysqld_safe]
	socket          = /var/run/mysqld/mysqld.sock
	nice            = 0
	syslog

	service mysql start

配置192.168.13.56

	mysql -u root -pzabbix@1344
	show master status;
	+------------------+----------+--------------+--------------------------+-------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB         | Executed_Gtid_Set |
	+------------------+----------+--------------+--------------------------+-------------------+
	| mysql-bin.000001 |      154 |              | mysql,information_schema |                   |
	+------------------+----------+--------------+--------------------------+-------------------+
	GRANT REPLICATION SLAVE ON *.* TO 'replication'@'192.168.13.%' IDENTIFIED  BY 'replication';
	flush privileges;
	change master to
	master_host='192.168.13.57',
	master_user='replication',
	master_password='replication',
	master_log_file='mysql-bin.000002',
	master_log_pos=154;
	start slave;

配置192.168.13.57

	mysql -u root -pzabbix@1344
	show master status;
	+------------------+----------+--------------+--------------------------+-------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB         | Executed_Gtid_Set |
	+------------------+----------+--------------+--------------------------+-------------------+
	| mysql-bin.000001 |      154 |              | mysql,information_schema |                   |
	+------------------+----------+--------------+--------------------------+-------------------+
	GRANT REPLICATION SLAVE ON *.* TO 'replication'@'192.168.13.%' IDENTIFIED  BY 'replication';
	flush privileges;
	change master to
	master_host='192.168.13.56',
	master_user='replication',
	master_password='replication',
	master_log_file='mysql-bin.000002',
	master_log_pos=154;
	start slave;

配置完成后在两台主机上都执行

	show slave status\G;

查看Slave_IO和Slave_SQL是否为YES

在192.168.13.56上执行

	mysql -pzabbix@1344
	create database zabbix character set utf8 collate utf8_bin;
	grant all privileges on zabbix.* to zabbix@'%' identified by 'Zabbix1344';
	grant all privileges on zabbix.* to zabbix@localhost identified by 'Zabbix1344';
	flush privileges;

在192.168.13.57上测试是否添加账号密码和zabbix库成功

	mysql -uzabbix -pZabbix1344
	show databases;

在192.168.13.57上创建和删除数据库，看192.168.13.56是否同步

	CREATE DATABASE my_db1;
	show databases;
	DROP DATABASE my_db1;

192.168.13.56和192.168.13.57安装keepalived

	apt-get install keepalived

查看是否加载ip_vs模块到内核，如果没有加载会导致VIP转移失败

	lsmod | grep ip_vs
	modprobe ip_vs
	modprobe ip_vs_wrr

192.168.13.56设置

	vim /etc/keepalived/keepalived.conf
	! Configuration File for keepalived
	global_defs {
	notification_email {
	iot@iot.com
	 }
	notification_email_from  iot@iot.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id MYSQL_HA
	 }
	vrrp_instance VI_1 {
	 state BACKUP
	 interface eth0
	 virtual_router_id 51
	 priority 100
	 advert_int 1
	 nopreempt
	 authentication {
	 auth_type PASS
	 auth_pass 1111
	 }
	 virtual_ipaddress {
	 192.168.13.60
	 }
	}
	virtual_server 192.168.13.60 3306 {
	 delay_loop 2
	 persistence_timeout 50
	 protocol TCP
	 real_server 192.168.13.56 3306 {
	 weight 3
	 notify_down /etc/keepalived/mysql.sh
	 TCP_CHECK {
	 connect_timeout 3
	 nb_get_retry 3
	 delay_before_retry 3
	  }
	}
	}

192.168.13.57设置

	vim /etc/keepalived/keepalived.conf
	! Configuration File for keepalived
	global_defs {
	notification_email {
	iot@iot.com
	 }
	notification_email_from  iot@iot.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id MYSQL_HA
	 }
	vrrp_instance VI_1 {
	 state BACKUP
	 interface eth0
	 virtual_router_id 51
	 priority 90
	 advert_int 1
	 authentication {
	 auth_type PASS
	 auth_pass 1111
	 }
	 virtual_ipaddress {
	 192.168.13.60
	 }
	}
	virtual_server 192.168.13.60 3306 {
	 delay_loop 2
	 persistence_timeout 50
	 protocol TCP
	 real_server 192.168.13.57 3306 {
	 weight 3
	 notify_down /etc/keepalived/mysql.sh
	 TCP_CHECK {
	 connect_timeout 3
	 nb_get_retry 3
	 delay_before_retry 3
	  }
	}
	}

192.168.13.56和192.168.13.57都配置

	vim /etc/keepalived/mysql.sh
	#!/bin/bash
	pkill keepalived
	chmod +x /etc/keepalived/mysql.sh
	service keepalived start

查看Mysql客户端最大连接数

	show variables like 'max_connections';


# 二、zabbix_server高可用 #

1、配置部署keepalived

在192.168.13.54和192.168.13.55上配置

	apt-get install keepalived
	apt-get install open-jdk

查看是否加载ip_vs模块到内核，如果没有加载会导致VIP转移失败

	lsmod | grep ip_vs
	modprobe ip_vs
	modprobe ip_vs_wrr

	vim /etc/keepalived/mysql.sh
	#!/bin/bash
	pkill keepalived
	chmod +x /etc/keepalived/mysql.sh

配置192.168.13.54的keepalived

	vim /etc/keepalived/keepalived.conf
	! Configuration File for keepalived
	global_defs {
	notification_email {
	iot@iot.com
	 }
	notification_email_from  iot@iot.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id ZABBIX_HA
	 }
	vrrp_instance VI_1 {
	 state BACKUP
	 interface eth0
	 virtual_router_id 55
	 priority 100
	 advert_int 1
	 nopreempt
	 authentication {
	 auth_type PASS
	 auth_pass 1111
	 }
	 virtual_ipaddress {
	 192.168.13.59
	 }
	}
	virtual_server 192.168.13.59 10051 {
	 delay_loop 2
	 persistence_timeout 50
	 protocol TCP
	 real_server 192.168.13.54 10051 {
	 weight 3
	 notify_down /etc/keepalived/zabbix.sh
	 TCP_CHECK {
	 connect_timeout 3
	 nb_get_retry 3
	 delay_before_retry 3
	  }
	}
	}

配置192.168.13.55的keepalived

	vim /etc/keepalived/keepalived.conf
	! Configuration File for keepalived
	global_defs {
	notification_email {
	iot@iot.com
	 }
	notification_email_from  iot@iot.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id ZABBIX_HA
	 }
	vrrp_instance VI_1 {
	 state BACKUP
	 interface eth0
	 virtual_router_id 55
	 priority 90
	 advert_int 1
	 authentication {
	 auth_type PASS
	 auth_pass 1111
	 }
	 virtual_ipaddress {
	 192.168.13.59
	 }
	}
	virtual_server 192.168.13.59 10051 {
	 delay_loop 2
	 persistence_timeout 50
	 protocol TCP
	 real_server 192.168.13.55 10051 {
	 weight 3
	 notify_down /etc/keepalived/zabbix.sh
	 TCP_CHECK {
	 connect_timeout 3
	 nb_get_retry 3
	 delay_before_retry 3
	  }
	}
	}


2、配置部署zabbix_server

从192.168.13.45复制了/opt/zabbix_home/到192.168.13.54和192.168.13.55上，目录位置没变

在192.168.13.54和192.168.13.55上都执行

修改zabbix前端对应的数据库

	vim /opt/zabbix_home/frontends/php/conf/zabbix.conf.php
	$DB['SERVER']   = '192.168.13.60';

修改zabbix_server指向的数据库

	vim /opt/zabbix_home/conf/zabbix/zabbix_server.conf
	DBHost=192.168.13.60

修改zabbix_server的SourceIP指向虚拟IP

	vim /opt/zabbix_home/conf/zabbix/zabbix_server.conf
	SourceIP=192.168.13.59

解决依赖关系
	ldd $(which /opt/zabbix_home/app/httpd/bin/httpd)
	ldd $(which /opt/zabbix_home/sbin/zabbix_server)
		
	apt-get install libaprutil1
	apt-get install libpcre3
	apt-get install libmysqlclient18:amd64
	apt-get install libnet-snmp-perl
	apt-get install snmp
	apt-get install snmp-mibs-downloader

	find / -name "libpcre.so*"
	ln -sv /lib/x86_64-linux-gnu/libpcre.so.3.13.3 /lib/x86_64-linux-gnu/libpcre.so.1

启动web

	/opt/zabbix_home/app/httpd/bin/httpd -k start

启动server

	/opt/zabbix_home/sbin/zabbix_server -c /opt/zabbix_home/conf/zabbix/zabbix_server.conf

查看server日志

	tail -200f /opt/zabbix_home/logs/zabbix/zabbix_server.log

启动keepalived

	service keepalived start

keepalived开启日志

	vim /etc/default/keepalived
	DAEMON_ARGS="-D -d -S 0"

解决微信python脚本依赖

	apt-get install python-simplejson
	/opt/zabbix_home/app/zabbix/share/zabbix/alertscripts/wechat.py 1 1 1


# 三、zabbix-proxy部署 #

1、下载安装proxy包，解决依赖关系

192.168.13.45上执行

	cd /opt
	wget http://repo.zabbix.com/zabbix/3.0/ubuntu/pool/main/z/zabbix/zabbix-proxy-mysql_3.0.4-1+trusty_amd64.deb
	dpkg -i zabbix-proxy-mysql_3.0.4-1+trusty_amd64.deb

2、导入数据库

192.168.13.44上执行

	mysql -uroot -p
	mysql >
	create database zabbix_proxy character set utf8 collate utf8_bin;
	grant all privileges on zabbix_proxy.* to zabbix@'%' identified by 'Zabbix1344';
	grant all privileges on zabbix_proxy.* to zabbix@localhost identified by 'Zabbix1344';
	flush privileges;

192.168.13.45上执行

	zcat /usr/share/doc/zabbix-proxy-mysql/schema.sql.gz | mysql -h192.168.13.44 -uzabbix -p"Zabbix1344" zabbix_proxy

3、修改proxy的配置文件

	vim /etc/zabbix/zabbix_proxy.conf
	Server=192.168.13.59
	ServerPort=10051
	Hostname=Zabbix_proxy
	LogFile=/var/log/zabbix/zabbix_proxy.log
	PidFile=/var/run/zabbix/zabbix_proxy.pid
	DBHost=192.168.13.44
	DBName=zabbix_proxy
	DBUser=zabbix
	DBPassword=Zabbix1344
	DBPort=3306
	ConfigFrequency=600
	DataSenderFrequency=3
	StartPollers=100
	StartPollersUnreachable=50
	StartTrappers=30
	StartDiscoverers=6
	JavaGateway=127.0.0.1
	JavaGatewayPort=10052
	StartJavaPollers=5
	CacheSize=320M
	StartDBSyncers=20
	HistoryCacheSize=512M
	Timeout=4
	ExternalScripts=/usr/lib/zabbix/externalscripts
	FpingLocation=/usr/bin/fping
	Fping6Location=/usr/bin/fping6
	LogSlowQueries=3000
	AllowRoot=1

4、修改server端和proxy端的hosts

	vim /etc/hosts
	192.168.13.45 Zabbix_proxy

5、启动zabbix_java监控jmx

	/opt/zabbix_home/app/zabbix/sbin/zabbix_java/startup.sh

6、启动zabbix-proxy

	service zabbix-proxy start

7、zabbix_proxy首次更新等待时间长用命令刷新解决
	zabbix_proxy -R config_cache_reload