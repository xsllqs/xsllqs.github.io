---
date: 2016-12-18 15:14:52+08:00
layout: post
title: 利用docker-compose搭建ELK5.0
categories: linux
tags: docker docker-compose ELK ELK5.0 elasticsearch kibana logstash kopf filebeat
---

# 一、搭建环境 #

	172.19.2.51：elasticsearch+kibana+logstash+kopf
	172.19.2.50：elasticsearch+nginx+filebeat
	172.19.2.49：elasticsearch

其中nginx的访问日志为我们要采集的内容，用filebeat传输，所以nginx和filebeat都没有在docker中运行

其他所有组件都在docker中运行，版本为5

# 二、172.19.2.51安装elk组件 #

## 1、安装docker-compose ##

	curl -L https://github.com/docker/compose/releases/download/1.3.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
	chmod +x /usr/local/bin/docker-compose
	vim /etc/profile
	export PATH="$PATH:/usr/local/bin"
	source /etc/profile
	echo $PATH

## 2、调整单进程的虚拟内存数，如果不调启动容器会报错 ##

	sysctl -a | grep vm.max_map_count
	sysctl -w vm.max_map_count=262144

## 3、创建配置文件目录和文件 ##

创建elasticsearch数据存储目录

	mkdir -pv /root/elk/elasticsearch

创建elasticsearch配置文件目录

	mkdir -pv /root/elk/es

创建kibana配置文件目录

	mkdir -pv /root/elk/kibana

创建logstash配置文件目录

	mkdir -pv /root/elk/logstash

创建elasticsearch配置文件

	vim /root/elk/es/elasticsearch.yml
	network.bind_host: 0.0.0.0
	network.host: 172.19.2.51
	cluster.name: es-cluster
	node.name: "es-node1"
	node.master: true
	discovery.zen.minimum_master_nodes: 1
	discovery.zen.ping.unicast.hosts:
	   -  172.19.2.51
	   -  172.19.2.50
	   -  172.19.2.49

创建kibana配置文件

	vim /root/elk/kibana/kibana.yml
	port: 5601
	host: "0.0.0.0"
	elasticsearch_url: "http://172.19.2.50:9100"
	elasticsearch_preserve_host: true
	kibana_index: ".kibana"
	default_app_id: "discover"
	request_timeout: 300000
	shard_timeout: 0
	verify_ssl: true
	bundled_plugin_ids:
	 - plugins/dashboard/index
	 - plugins/discover/index
	 - plugins/doc/index
	 - plugins/kibana/index
	 - plugins/markdown_vis/index
	 - plugins/metric_vis/index
	 - plugins/settings/index
	 - plugins/table_vis/index
	 - plugins/vis_types/index
	 - plugins/visualize/index

创建logstash配置文件

	vim /root/elk/logstash/logstash.conf
	input {
	  beats {
	        port => 20000
	        codec => "json"
	    }
	}
	
	output {
	  elasticsearch {
	    hosts => "172.19.2.50:9100"
	    index => "nginx" }
	}

创建docker-compose配置文件

	vim /root/elk/docker-compose.yml
	elasticsearch:
	  image: elasticsearch:5
	  command: elasticsearch
	  environment:
	    - "ES_JAVA_OPTS=-Xmx1g -Xms1g"
	  volumes:
	    - ./elasticsearch:/usr/share/elasticsearch/data
	    - ./es/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
	  ports:
	    - "9200:9200"
	    - "9300:9300"
	
	logstash:
	  image: logstash:latest
	  command: logstash -w 4 -f /etc/logstash/conf.d/logstash.conf
	  environment:
	    - LS_HEAP_SIZE=2048m
	  volumes:
	    - ./logstash/logstash.conf:/etc/logstash/conf.d/logstash.conf
	  ports:
	    - "20000:20000"
	
	kibana:
	  image: kibana:latest
	  volumes:
	    - ./kibana/kibana.yml:/etc/kibana/kibana.yml
	  ports:
	    - "5601:5601"
	
	kopf:
	  image: lmenezes/elasticsearch-kopf
	  ports:
	    - "80:80"
	  environment:
	    - KOPF_SERVER_NAME=kopf
	    - KOPF_ES_SERVERS=172.19.2.50:9100

## 4、启动docker-compose ##

	cd /root/elk
	docker-compose up
	docker-compose ps

# 三、172.19.2.51安装elasticsearch和nginx+filebeat #

## 1、安装docker-compose ##

	curl -L https://github.com/docker/compose/releases/download/1.3.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
	chmod +x /usr/local/bin/docker-compose
	vim /etc/profile
	export PATH="$PATH:/usr/local/bin"
	source /etc/profile
	echo $PATH

## 2、调整单进程的虚拟内存数 ##

	sysctl -a | grep vm.max_map_count
	sysctl -w vm.max_map_count=262144

## 3、创建配置文件目录和文件 ##

创建elasticsearch数据存储目录

	mkdir -pv /root/elk/elasticsearch

创建elasticsearch配置文件目录

	mkdir -pv /root/elk/es

创建elasticsearch配置文件

	vim /root/elk/es/elasticsearch.yml
	network.bind_host: 0.0.0.0
	network.host: 172.19.2.50
	cluster.name: es-cluster
	node.name: "es-node2"
	node.master: true
	discovery.zen.minimum_master_nodes: 1
	discovery.zen.ping.unicast.hosts:
	   -  172.19.2.51
	   -  172.19.2.50 
	   -  172.19.2.49

创建docker-compose配置文件

	vim /root/elk/docker-compose.yml
	elasticsearch:
	  image: elasticsearch:5
	  command: elasticsearch
	  environment:
	    - "ES_JAVA_OPTS=-Xmx1g -Xms1g"
	  volumes:
	    - ./elasticsearch:/usr/share/elasticsearch/data
	    - ./es/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
	  ports:
	    - "9200:9200"
	    - "9300:9300"

修改nginx配置文件（此nginx用来返带elasticsearch集群的9200端口至9100，即es集群的3台主机的9200端口都通过172.19.2.50:9200访问，同时我们采集此nginx的80端口访问日志）

	vim /etc/nginx/nginx.conf
	user  nginx;
	worker_processes  1;
	error_log  /var/log/nginx/error.log warn;
	pid        /var/run/nginx.pid;
	events {
	    worker_connections  1024;
	}
	http {
	    include       /etc/nginx/mime.types;
	    default_type  application/octet-stream;
	    log_format  logstash_json  '{ "@timestamp": "$time_local", '
	                               '"@fields": { '
	                               '"remote_addr": "$remote_addr", '
	                               '"remote_user": "$remote_user", '
	                               '"body_bytes_sent": "$body_bytes_sent", '
	                               '"request_time": "$request_time", '
	                               '"status": "$status", '
	                               '"request": "$request", '
	                               '"request_method": "$request_method", '
	                               '"http_referrer": "$http_referer", '
	                               '"body_bytes_sent":"$body_bytes_sent", '
	                               '"http_x_forwarded_for": "$http_x_forwarded_for", '
	                               '"http_user_agent": "$http_user_agent" } }';
	    access_log  /var/log/nginx/access.log  logstash_json;
	    sendfile        on;
	    keepalive_timeout  65;
	
	    upstream els {
	        server 172.19.2.49:9200 weight=1 max_fails=2 fail_timeout=1;
	        server 172.19.2.50:9200 weight=1 max_fails=2 fail_timeout=1;
	        server 172.19.2.51:9200 weight=1 max_fails=2 fail_timeout=1;
	        }
	
	    server {
	        listen       9100;
	        access_log  /var/log/nginx/accessels.log  logstash_json;
	
	        location / {
	            proxy_pass   http://els/;
	            proxy_set_header Host $host;
	            proxy_set_header X-Real-IP $remote_addr;
	            }
	        }
	
	    include /etc/nginx/conf.d/*.conf;
	}

## 4、安装和配置filebeat ##

	cd /root/
	curl -L -O https://download.elastic.co/beats/filebeat/filebeat-1.3.0-x86_64.rpm
	rpm -vi filebeat-1.3.0-x86_64.rpm
	vim /etc/filebeat/filebeat.yml
	filebeat:
	  prospectors:
	    -
	      paths:
	        - /var/log/nginx/access.log
	      input_type: log
	      multiline:
	        negate: true
	        match: after
	      tail_files: false
	  registry_file: /var/lib/filebeat/registry
	output:
	  logstash:
	    hosts: ["172.19.2.51:20000"]
	    worker: 4
	shipper:
	logging:
	  files:
	    rotateeverybytes: 10485760 # = 10MB

## 5、启动docker-compose，启动nginx，启动filebeat ##

	cd /root/elk
	docker-compose up
	service nginx start
	service filebeat start

# 四、172.19.2.49安装elasticsearch节点 #

## 1、安装docker-compose ##

	curl -L https://github.com/docker/compose/releases/download/1.3.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
	chmod +x /usr/local/bin/docker-compose
	vim /etc/profile
	export PATH="$PATH:/usr/local/bin"
	source /etc/profile
	echo $PATH

## 2、调整单进程的虚拟内存数 ##

	sysctl -a | grep vm.max_map_count
	sysctl -w vm.max_map_count=262144

## 3、创建配置文件目录和文件 ##

创建elasticsearch数据存储目录

	mkdir -pv /root/elk/elasticsearch

创建elasticsearch配置文件目录

	mkdir -pv /root/elk/es

创建elasticsearch配置文件

	vim /root/elk/es/elasticsearch.yml
	network.bind_host: 0.0.0.0
	network.host: 172.19.2.49
	cluster.name: es-cluster
	node.name: "es-node3"
	node.master: true
	discovery.zen.minimum_master_nodes: 1
	discovery.zen.ping.unicast.hosts:
	   -  172.19.2.51
	   -  172.19.2.50 
	   -  172.19.2.49

创建docker-compose配置文件

	vim /root/elk/docker-compose.yml
	elasticsearch:
	  image: elasticsearch:5
	  command: elasticsearch
	  environment:
	    - "ES_JAVA_OPTS=-Xmx1g -Xms1g"
	  volumes:
	    - ./elasticsearch:/usr/share/elasticsearch/data
	    - ./es/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
	  ports:
	    - "9200:9200"
	    - "9300:9300"

## 4、启动docker-compose ##

	cd /root/elk
	docker-compose up

#五、ELK插件访问地址#

## 1、kopf ##

http://172.19.2.51/#!/cluster

## 2、kibana ##

http://172.19.2.51:5601/

## 3、所有配置文件已上传git ##

https://github.com/xsllqs/Blogfile/tree/master/elk