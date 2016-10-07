---
date: 2016-10-7 10:44:24+08:00
layout: post
title: pinpoint的安装部署
categories: linux
tags: pinpoint apm hbase zookeeper tomcat
---

# 一、pinpoint介绍 #

Pinpoint是一个开源的APM工具，分布式事务跟踪系统的平台，思路基于google Dapper，用于基于java的大规模分布式系统，通过跟踪分布式应用之间的调用来提供解决方案，以帮助分析系统的总体结构和内部模块之间如何相互联系。

git地址为：https://github.com/naver/pinpoint

架构

![1]({{ site.url }}/assets/2016-10-7-pinpoint1.png)

Pinpoint的特点如下:

	分布式事务跟踪，跟踪跨分布式应用的消息
	
	自动检测应用拓扑，帮助你搞清楚应用的架构
	
	水平扩展以便支持大规模服务器集群
	
	提供代码级别的可见性以便轻松定位失败点和瓶颈
	
	使用字节码增强技术，添加新功能而无需修改代码
	
	安装探针不需要修改哪怕一行代码及trace server端部署简单，支持hdfs存储
	
	具有简单的阀值触发报警功能
	
	移植性比较强的，会比较讨人喜欢（相比cat）
	
	插件化功能可扩展（https://github.com/naver/pinpoint/wiki/Pinpoint-Plugin-Developer-Guide）

# 二、安装 #

## 1、环境准备 ##

	apache-tomcat-8.0.37.tar.gz

(下载地址：http://tomcat.apache.org/download-80.cgi#8.0.37)

	hbase-1.0.3-bin.tar.gz

（下载地址：http://apache.fayea.com/hbase/）

	jdk-8u102-linux-x64.rpm

（下载地址：http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html）

	pinpoint-agent-1.5.2.tar.gz

（下载地址：https://github.com/naver/pinpoint/releases/tag/1.5.2）

	pinpoint-collector-1.5.2.war

（下载地址：https://github.com/naver/pinpoint/releases/tag/1.5.2）

	pinpoint-web-1.5.2.war

（下载地址：https://github.com/naver/pinpoint/releases/tag/1.5.2）

	zookeeper-3.4.6-10.el6.x86_64.rpm

（下载地址：http://apache.fayea.com/zookeeper/）

全部放入/root/pp目录

## 2、环境部署位置 ##

192.168.80.144    CentOS6(jdk1.8.0)    Pinpoint-collector,Pinpoint-web,zookeeper,Hbase  

192.168.80.143    CentOS6(jdk1.8.0)    Pinpoint-agent(demo应用)

## 3、安装jdk(192.168.44.1) ##

	[root@localhost ~]# rpm -ivh jdk-8u102-linux-x64.rpm

## 4、安装hbase ##

定义好环境变量中jdk的位置

	[root@localhost ~]# vim ~/.bashrc
		export JAVA_HOME=/usr/java/default
		export PATH=$PATH:$JAVA_HOME/bin
	[root@localhost ~]# source ~/.bashrc

安装配置hbase

	[root@localhost ~]# tar xf /root/pp/hbase-1.0.3-bin.tar.gz /usr/local/
	[root@localhost ~]# cd /usr/local/hbase-1.0.3/conf
	[root@localhost ~]# vim hbase-env.sh
		这里修改为环境变量中JAVA_HOME的位置
		export JAVA_HOME=/usr/java/default/
	[root@localhost ~]# vim hbase-site.xml
		<configuration>
		  <property>
		    <name>hbase.rootdir</name>
		    <value>file:///data/hbase</value>        # 这里我们指定Hbase本地来存储数据，生产环境将数据建议存入HDFS中。
		 </property>
		</configuration>

启动HBASE

	[root@localhost ~]# /usr/local/hbase-1.0.3/bin/start-hbase.sh

关闭方法

	[root@localhost ~]# /usr/local/hbase-1.0.3/bin/stop-hbase.sh

下载HBASE初始化脚本

	[root@localhost ~]# wget https://raw.githubusercontent.com/naver/pinpoint/master/hbase/scripts/hbase-create.hbase
	[root@localhost ~]# /usr/local/hbase-1.0.3/bin/hbase shell /root/hbase-create.hbase
	[root@localhost ~]# /usr/local/hbase-1.0.3/bin/hbase shell
	hbase(main):001:0> status 'detailed'

启动HBASE验证

	[root@localhost ~]# /usr/local/hbase-1.0.3/bin/start-hbase.sh
	[root@localhost ~]# jps
	12659 Jps
	5115 HMaster

进入HBASE页面查看

	http://192.168.80.144:16010/master-status

## 5、安装zookeeper ##

安装启动zookeeper

	[root@localhost ~]# rpm -ivh /root/zookeeper-3.4.6-10.el6.x86_64.rpm
	[root@localhost ~]# /etc/init.d/zookeeper start

启动时间有点长

	[root@localhost ~]# ss -tnlp | grep 2181

## 6、安装pinpoint-collector和pinpoint-web ##

安装配置pinpoint-collector

	[root@localhost ~]# mkdir -pv /data/service/
	[root@localhost ~]# tar xf /root/pp/apache-tomcat-8.0.37.tar.gz -C /data/service/
	[root@localhost ~]# cd /data/service/
	[root@localhost ~]# mv apache-tomcat-8.0.37/ pinpoint-collector
	修改tomcat端口不要让collector的端口和web的端口冲突
	[root@localhost ~]# vim /data/service/pinpoint-collector/conf/server.xml
		<Server port="8005" shutdown="SHUTDOWN">     #修改端口
		<Connector port="8085" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />		#修改端口
		<!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" /> -->     #  注释该行
	[root@localhost ~]# rm -rf /data/service/pinpoint-collector/webapps/*
	[root@localhost ~]# unzip pinpoint-collector-1.5.2.war -d /data/service/pinpoint-collector/webapps/ROOT/
	[root@localhost ~]# cd /data/service/pinpoint-collector/webapps/ROOT/WEB-INF/classes
	[root@localhost ~]# vim hbase.properties
		hbase.client.host=192.168.80.144       # 修改这里让collector向hbase存储数据
		hbase.client.port=2181

安装配置pinpoint-web

	[root@localhost ~]# cd /data/service/
	[root@localhost ~]# tar xf /root/pp/apache-tomcat-8.0.37.tar.gz -C /data/service/
	[root@localhost ~]# mv apache-tomcat-8.0.37 pinpoint-web
	[root@localhost ~]# cd /data/service/pinpoint-web/webapps/
	[root@localhost ~]# rm -rf /data/service/pinpoint-web/webapps/*
	[root@localhost ~]# mkdir /data/service/pinpoint-web/webapps/ROOT
	[root@localhost ~]# cd /data/service/pinpoint-web/webapps/ROOT/
	[root@localhost ~]# unzip /root/pinpoint-web-1.5.2.war -d /data/service/pinpoint-web/webapps/ROOT/
	[root@localhost ~]# vim /data/service/pinpoint-web/conf/server.xml
		<Server port="8006" shutdown="SHUTDOWN">      #修改端口
		<Connector port="8086" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />		#修改端口
		<!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" /> -->     #  注释该行
	[root@localhost ~]# vim /data/service/pinpoint-web/webapps/ROOT/WEB-INF/classes/hbase.properties
		hbase.client.host=192.168.80.144
		hbase.client.port=2181

启动collector和web

	[root@localhost ~]# /data/service/pinpoint-web/bin/catalina.sh start
	[root@localhost ~]# /data/service/pinpoint-collector/bin/catalina.sh start
	验证web端口是否启动
	[root@localhost ~]# ss -tnlp | grep 8086
	验证collector端口是否启动
	[root@localhost ~]# ss -tnlp | grep 8085

查看页面是否能打开

http://192.168.80.144:8086/#/main/

## 7、安装demo应用（192.168.80.143） ##

demo应用我选择了一个简单的blog

	[root@localhost ~]# yum install java-1.7.0-openjdk
	[root@localhost ~]#  yum install java-1.7.0-openjdk-devel.x86_64
	[root@localhost ~]# wget http://mirror.bit.edu.cn/apache/tomcat/tomcat-7/v7.0.70/bin/apache-tomcat-7.0.70.tar.gz
	[root@localhost ~]# vim /etc/profile.d/java.sh
		export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.111-2.6.7.2.el7_2.x86_64
		export PATH=$JAVA_HOME/bin:$PATH
	[root@localhost ~]# . /etc/profile.d/java.sh 
	[root@localhost ~]# java -version
	[root@localhost ~]# tar xf apache-tomcat-7.0.70.tar.gz -C /usr/local
	[root@localhost ~]# ln -sv /usr/local/apache-tomcat-7.0.70 /usr/local/tomcat
	[root@localhost ~]# vim /etc/profile.d/tomcat.sh
		export CATALINA_HOME=/usr/local/tomcat
		export PATH=$CATALINA_HOME/bin:$PATH

应用介绍地址：http://git.oschina.net/94fzb/zrlog

下载zrlog到本地：http://dl.zrlog.com/release/zrlog.war
	
	[root@localhost ~]# wget http://dl.zrlog.com/release/zrlog.war
	[root@localhost ~]# cp -a /root/zrlog.war /usr/local/tomcat/webapps/ROOT.war

安装配置mysql

	[root@localhost ~]# yum install mariadb-server
	[root@localhost ~]# service mariadb start
	[root@localhost ~]# mysql
	MariaDB [(none)]> create database zrlog;
	MariaDB [(none)]> grant all privileges on zrlog.* to java1234@'%' identified by 'abc123';
	MariaDB [(none)]> grant all privileges on zrlog.* to java1234@localhost identified by 'abc123';
	MariaDB [(none)]> flush privileges;
	[root@localhost ~]# catalina.sh start

访问应用设置基本配置

	访问地址：http://192.168.80.143:8080/install
	数据库服务器:127.0.0.1
	数据库名:zrlog
	数据库用户名:java1234
	数据库密码:abc123
	数据库端口:3306
	系统邮箱:405812999@qq.com

	管理员账号:admin
	管理员密码:abc123
	管理员 Email:405812999@qq.com
	网站标题:我的Blog
	网站副标题:zrLog

## 8、安装agent ##

	[root@localhost ~]# mkdir -pv /data/projects/service/
	[root@localhost ~]# tar xf /root/pp/pinpoint-agent-1.5.2.tar.gz -C /data/projects/service/
	改为colletor地址
	[root@localhost ~]# vim /data/projects/service/pinpoint-agent-1.5.2/pinpoint.config 
		profiler.collector.ip=192.168.80.144
	停掉应用
	[root@localhost ~]# catalina.sh stop
	在应用的catalina.sh最开头处添加
	[root@localhost ~]# vim /usr/local/tomcat/bin/catalina.sh
		CATALINA_OPTS="$CATALINA_OPTS -javaagent:/data/projects/service/pinpoint-agent-1.5.2/pinpoint-bootstrap-1.5.2.jar"
		CATALINA_OPTS="$CATALINA_OPTS -Dpinpoint.agentId=0000003"
		CATALINA_OPTS="$CATALINA_OPTS -Dpinpoint.applicationName=192.168.80.143_8080_blog"
	启动应用
	[root@localhost ~]# catalina.sh start

## 9、测试效果 ##

先进入demo应用随便点几个按钮操作一下

![5]({{ site.url }}/assets/2016-10-7-pinpoint5.png)

打开pinpoint页面，选中对应的主机和查看时间，查询时间不能超过2天外的

![2]({{ site.url }}/assets/2016-10-7-pinpoint2.png)

框选指定时间段的散点图，框选后会弹出这个时间段对应的数据

![3]({{ site.url }}/assets/2016-10-7-pinpoint3.png)

![4]({{ site.url }}/assets/2016-10-7-pinpoint4.png)

## 10、说明 ##

本文参考了**扁豆焖面先生**的博客：https://sconts.com/11

感谢**扁豆焖面先生**的分享
