---
date: 2017-05-10 11:12:52+08:00
layout: post
title: zabbix监控kafka消息列队
categories: linux
tags: zabbix kafka
---

#一、Kafka监控的几个指标#

	1、lag：多少消息没有消费
	2、logsize：Kafka存的消息总数
	3、offset：已经消费的消息


#二、查看zookeeper配置#

	cat /home/app/zookeeper/zookeeper/conf/zoo.cfg | egrep -v "^$|^#"
	clientPort=2181

#三、查看kafka配置#

	cat /home/app/kafka/kafka/config/server.properties | egrep -v "^$|^#"
	port=9092
	host.name=192.168.38.87
	zookeeper.connect=192.168.38.87:2181,192.168.38.88:2181

#四、查看kafka的group name#

	cd /home/app/zookeeper/zookeeper/bin
	./zkCli.sh -server 192.168.38.87:2181
	ls /consumers/
	lijieGroup
	quit

#五、查看kafka的topic_name#

	/home/app/kafka/kafka/bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --group=lijieGroup --zookeeper=192.168.38.87:2181


#六、修改zabbix配置文件

因为zabbix用户不能调用kafka的脚本，需要root用户启动zabbix_agent

	vim /opt/zabbix/zabbix_agentd.conf
	AllowRoot=1
	User=root
	Include=/opt/zabbix/zabbix_agentd.conf.d/
	
	vim /opt/zabbix/zabbix_agentd.conf.d/kafka_status.conf
	UserParameter=kafka.lag[*],/home/zabbix_scripts/kafka_mon.sh $1 $2 lag
	UserParameter=kafka.offset[*],/home/zabbix_scripts/kafka_mon.sh $1 $2 offset
	UserParameter=kafka.logsize[*],/home/zabbix_scripts/kafka_mon.sh $1 $2 logsize
	
	chown -R zabbix:zabbix /opt/zabbix/zabbix_agentd.conf.d/kafka_status.conf
	chmod -R 777 /opt/zabbix/zabbix_agentd.conf.d/kafka_status.conf

#七、创建监控脚本#
	mkdir -pv /home/zabbix_scripts/
	vim /home/zabbix_scripts/kafka_mon.sh
	#!/bin/bash
	
	#Group           Topic                          Pid Offset          logSize         Lag             Owner
	#lijieGroup      RouterOnOfflineStateChange     0   17073689        17073689        0               lijieGroup_localhost.localdomain-1492764195889-abda96d0-0
	
	#cat /tmp/kafka-tp.info | grep -v Offset | awk '{print $4}'
	
	kafka_ip="192.168.38.87"
	kafka_port=2181
	topic_name=$1
	group_id=$2
	pn=$3
	/home/app/kafka/kafka/bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --topic=$topic_name --group=$group_id --zookeeper=$kafka_ip:$kafka_port | grep -v Offset > /tmp/kafka-tp-${topic_name}-${group_id}.info
	
	Offset=0
	logSize=0
	Lag=0
	while read line
	do
	    Offset=$((${Offset}+`echo $line |awk '{print $4}'`))
	    logSize=$((${logSize}+`echo $line |awk '{print $5}'`))
	    Lag=$(($Lag+`echo $line |awk '{print $6}'`))
	done < /tmp/kafka-tp-${topic_name}-${group_id}.info
	
	#echo Offset :$Offset
	#echo logSize :$logSize
	#echo Lag : $Lag
	case $pn in
	    offset|Offset)
	    echo $Offset
	    ;;
	    logsize|logSize)
	    echo $logSize
	    ;;
	    lag|Lag)
	    echo $Lag
	    ;;
	    *)
	    echo Error
	    ;;
	esac

#八、给脚本和对应文件权限#

	chown -R zabbix:zabbix /home/zabbix_scripts/kafka_mon.sh
	chmod -R 777 /home/zabbix_scripts/kafka_mon.sh
	
	touch /tmp/kafka-tp-RouterOnOfflineStateChange-lijieGroup.info
	chmod 777 /tmp/kafka-tp-RouterOnOfflineStateChange-lijieGroup.info
	chown zabbix:zabbix /tmp/kafka-tp-RouterOnOfflineStateChange-lijieGroup.info
	
	chmod 777  /home/app/kafka/kafka/bin/kafka-run-class.sh

#九、重启zabbix

	/opt/zabbix/sbin/zabbix_agentd -c /opt/zabbix/zabbix_agentd.conf


#十、监控上增加3个键值

	kafka.offset[RouterOnOfflineStateChange,lijieGroup]
	kafka.logsize[RouterOnOfflineStateChange,lijieGroup]
	kafka.lag[RouterOnOfflineStateChange,lijieGroup]