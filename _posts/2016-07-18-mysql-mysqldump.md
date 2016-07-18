---
date: 2016-07-18 22:59:53+08:00
layout: post
title: mysql备份之mysqldump
categories: linux
tags: mysql mysqldump mysql备份
---


注意：备份文件和二进制日志文件不能与mysql放在同一磁盘下

# 节点1 #

1、节点1上修改mysql配置文件，开起二进制日志保存

这里我将二进制日志放在/data/mysql/目录下，/data/是我创建的另外一个lvm磁盘，本来想直接放在/data/下，发现无法启动mysql，所以建议还是放在/data/mysql中

	[root@node1 ~]# mkdir -pv /data/mysql/
	[root@node1 ~]# chown mysql:mysql /data/*
	[root@node1 mysql]# cd /var/lib/mysql
	[root@node1 mysql]# cp -a mysql-bin.000001 mysql-bin.000002 mysql-bin.index /data/mysql/
	[root@node1 ~]# vim /etc/my.cnf.d/server.cnf
		[server]
		log_bin=/data/mysql/mysql-bin
	[root@node1 ~]# service mariadb restart

2、查看二进制日志的一些信息

	[root@node1 ~]# mysql
		MariaDB [(none)]> show master logs;
		+------------------+-----------+
		| Log_name         | File_size |
		+------------------+-----------+
		| mysql-bin.000001 |       264 |
		| mysql-bin.000002 |       245 |
		+------------------+-----------+

3、查看表的存储引擎类型并备份

	MariaDB [hellodb]> show table status\G;

如果engine是myisam则备份方案如下，需要对锁表后操作

	[root@node1 ~]# mysqldump -uroot --lock-tables --master-data=2 --flush-logs --databases hellodb > /root/hellodb_myis.sql

如果engine是innodb则备份方案如下

	[root@node1 ~]# mysqldump -uroot --single-transaction --master-data=2 --flush-logs --databases hellodb > /root/hellodb_inno.sql
	--single-transaction：热备
	--master-data=2：记录为注释的CHANGE MASTER TO语句
	--flush-logs：日志滚动

批量修改表的存储引擎【将得到的结果一次执行即可修改，不建议直接在mysql中修改】

	MariaDB [hellodb]> SELECT CONCAT('ALTER TABLE ',table_name,' ENGINE=InnoDB;') FROM information_schema.tables WHERE table_schema='hellodb' AND ENGINE='myisam';

4、修改表内数据

	MariaDB [(none)]> use hellodb;
	MariaDB [hellodb]> insert into students (Name,Age,Gender,ClassID,TeacherID) values ('caocao',99,'M',6,8);
	MariaDB [hellodb]> delete from students where stuid=3;

5、复制备份文件到另一节点

	[root@node1 ~]# scp hellodb_inno.sql 192.168.1.114:/root/

# 节点2 #

6、在另一个节点进行mysql恢复

修改节点2的配置文件

	[root@node2 ~]# mkdir -pv /data/mysql
	[root@node2 ~]# vim /etc/my.cnf
		[mysqld] 
		log_bin=/data/mysql/mysql-bin
	[root@node2 ~]# chown mysql:mysql /data/*
	[root@node2 ~]# chown mysql:mysql /data
	[root@node2 ~]# service mariadb start

还原备份文件

	[root@node2 ~]# mysql < /root/hellodb_inno.sql
	[root@node2 ~]# less hellodb_inno.sql
		-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=245;

根据表中的显示，在备份那一刻，二进制日志mysql-bin.000002，操作到了245

7、在节点2上恢复二进制日志

在节点1上将245之后的二进制日志文件转换为sql文件

	[root@node1 ~]# mysqlbinlog --start-position=245 /var/lib/mysql/mysql-bin.000002 > binlog.sql

复制给节点2

	[root@node1 ~]# scp binlog.sql 192.168.1.114:/root/

利用刚才生产的sql文件来恢复备份之后操作的内容

	[root@node2 ~]# mysql < /root/binlog.sql 

8、查看恢复情况

	[root@node2 ~]# mysql
	MariaDB [(none)]> use hellodb;
	MariaDB [hellodb]> select * from students;
	+-------+---------------+-----+--------+---------+-----------+
	| StuID | Name          | Age | Gender | ClassID | TeacherID |
	+-------+---------------+-----+--------+---------+-----------+
	|     1 | Shi Zhongyu   |  22 | M      |       2 |         3 |
	|     2 | Shi Potian    |  22 | M      |       1 |         7 |
	|     4 | Ding Dian     |  32 | M      |       4 |         4 |
	|     5 | Yu Yutong     |  26 | M      |       3 |         1 |
	|     6 | Shi Qing      |  46 | M      |       5 |      NULL |
	|     7 | Xi Ren        |  19 | F      |       3 |      NULL |
	|     8 | Lin Daiyu     |  17 | F      |       7 |      NULL |
	|     9 | Ren Yingying  |  20 | F      |       6 |      NULL |
	|    10 | Yue Lingshan  |  19 | F      |       3 |      NULL |
	|    11 | Yuan Chengzhi |  23 | M      |       6 |      NULL |
	|    12 | Wen Qingqing  |  19 | F      |       1 |      NULL |
	|    13 | Tian Boguang  |  33 | M      |       2 |      NULL |
	|    14 | Lu Wushuang   |  17 | F      |       3 |      NULL |
	|    15 | Duan Yu       |  19 | M      |       4 |      NULL |
	|    16 | Xu Zhu        |  21 | M      |       1 |      NULL |
	|    17 | Lin Chong     |  25 | M      |       4 |      NULL |
	|    18 | Hua Rong      |  23 | M      |       7 |      NULL |
	|    19 | Xue Baochai   |  18 | F      |       6 |      NULL |
	|    20 | Diao Chan     |  19 | F      |       7 |      NULL |
	|    21 | Huang Yueying |  22 | F      |       6 |      NULL |
	|    22 | Xiao Qiao     |  20 | F      |       1 |      NULL |
	|    23 | Ma Chao       |  23 | M      |       4 |      NULL |
	|    24 | Xu Xian       |  27 | M      |    NULL |      NULL |
	|    25 | Sun Dasheng   | 100 | M      |    NULL |      NULL |
	|    26 | caocao        |  99 | M      |       6 |         8 |
	+-------+---------------+-----+--------+---------+-----------+



