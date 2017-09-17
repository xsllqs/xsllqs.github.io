---
date: 2017-07-20 11:12:52+08:00
layout: post
title: zabbix���mysql
categories: linux
tags: zabbix mysql
---

# һ������һ��zabbix�õ����ݿ��û� #

	grant all privileges on zabbix.* to zabbix@'%' identified by 'zabbix';
	grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';
	flush privileges;


# �����޸�zabbix-agent�������ļ� #

	vim /home/zabbix/zabbix_agents/conf/zabbix_agentd.conf.d/userparameter_mysql.conf
	UserParameter=mysql.status[*],echo "show global status where Variable_name='$1';" | HOME=/home/zabbix/zabbix_agents/ mysql -N | awk '{print $$2}'
	UserParameter=mysql.size[*],bash -c 'echo "select sum($(case "$3" in both|"") echo "data_length+index_length";; data|index) echo "$3_length";; free) echo "data_free";; esac)) from information_schema.tables$([[ "$1" = "all" || ! "$1" ]] || echo " where table_schema=\"$1\"")$([[ "$2" = "all" || ! "$2" ]] || echo "and table_name=\"$2\"");" | HOME=/home/zabbix/zabbix_agents/ mysql -N'
	UserParameter=mysql.ping,HOME=/home/zabbix/zabbix_agents/ mysqladmin ping | grep -c alive
	UserParameter=mysql.version,mysql -V
	UserParameter=mysql.slavestatus,echo "show slave status\G" | HOME=/home/zabbix/zabbix_agents/ mysql -N | grep -E "Slave_IO_Running|Slave_SQL_Running" | awk '{print $2}'|grep -c Yes

	vim /usr/local/etc/zabbix_agentd.conf
	Include=/home/zabbix/zabbix_agents/conf/zabbix_agentd.conf.d/userparameter_mysql.conf

# ����д���ѯ�õ������ļ� #

	vim /home/zabbix/zabbix_agents/.my.cnf
	[mysql]
	host=172.19.2.68
	user=zabbix
	password=zabbix
	socket=/var/lib/mysql/mysql.sock
	[mysqladmin]
	host=172.19.2.68
	user=zabbix
	password=zabbix
	socket=/var/lib/mysql/mysql.sock

# �ġ�����zabbix-agent #

	/home/zabbix/zabbix_agents/sbin/zabbix_agentd -c /usr/local/etc/zabbix_agentd.conf


