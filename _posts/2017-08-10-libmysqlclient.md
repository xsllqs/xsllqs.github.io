---
date: 2017-08-10 9:12:52+08:00
layout: post
title: 编译安装时出现依赖文件故障的解决方法
categories: linux
tags: 编译安装 故障
---


编译安装时出现/usr/bin/ld: cannot find -lxxx故障的解决方法
编译安装时出现/usr/bin/ld: cannot find -lmysqlclient_r故障的解决方法

# 1、查看依赖文件位置locate libmysqlclient_r #

	locate libmysqlclient_r
	/usr/lib64/mysql/libmysqlclient_r.so
	/usr/lib64/mysql/libmysqlclient_r.so.16
	/usr/lib64/mysql/libmysqlclient_r.so.16.0.0

发现文件不存在

	ll /usr/lib64/mysql/libmysqlclient_r.so
	ll /usr/lib64/mysql/libmysqlclient_r.so.16
	ll /usr/lib64/mysql/libmysqlclient_r.so.16.0.0

查找系统是否存在该文件

	find / -name libmysqlclient_r*
	/usr/lib64/libmysqlclient_r.so.14.0.0
	/usr/lib64/libmysqlclient_r.so.12
	/usr/lib64/libmysqlclient_r.so.12.0.0
	/usr/lib64/libmysqlclient_r.so.16.0.0
	/usr/lib64/libmysqlclient_r.so.16
	/usr/lib64/libmysqlclient_r.so.15.0.0
	/usr/lib64/libmysqlclient_r.so.15
	/usr/lib64/libmysqlclient_r.so.14


# 2、把找到的文件指向locate中定义的文件 #

	ln -sf /usr/lib64/libmysqlclient_r.so.16 /usr/lib64/mysql/libmysqlclient_r.so
	ln -sf /usr/lib64/libmysqlclient_r.so.16 /usr/lib64/mysql/libmysqlclient_r.so.16
	ln -sf /usr/lib64/libmysqlclient_r.so.16 /usr/lib64/mysql/libmysqlclient_r.so.16.0.0


# 3、修改配置和环境变量 #

	vim /etc/ld.so.conf
	include ld.so.conf.d/*.conf
	/usr/local/ssl/lib
	/usr/lib64/mysql/
	/usr/lin64/

	ldconfig

	vim ~/.bashrc
	export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib64/mysql/
	export LIBRARY_PATH=/usr/lib64/mysql/:$LIBRARY_PATH

	source ~/.bashrc

# 4、重新编译 #

	make clean
	make

