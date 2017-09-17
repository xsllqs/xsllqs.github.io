---
date: 2017-07-15 11:12:52+08:00
layout: post
title: ELK集群搭建和filebeat安装配置
categories: linux
tags: elasticsearch ELK logstash filebeat kibana cerebro
---

访问elasticsearch集群：http://172.19.2.141:9100

访问kibana：http://172.19.2.141:5601

访问cerebro：http://172.19.2.50:1234

# 一、安装配置elasticsearch #

	#在["172.19.2.49", "172.19.2.50", "172.19.2.51", "172.19.2.140", "172.19.2.141"]上安装elasticsearch

	#环境配置：在5台测试环境的主机上安装jdk1.8
	yum install java-1.8.0-openjdk.x86_64

	#在5台测试环境的主机上部署elasticsearch5.2
	cd /home/app/
	wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.2.0.rpm
	rpm -ivh elasticsearch-5.2.0.rpm

	#在172.19.2.49配置
	vim /etc/elasticsearch/elasticsearch.yml
	cluster.name: ELK_logs
	node.name: es-node1
	http.port: 9200
	network.host: 172.19.2.49
	discovery.zen.ping.unicast.hosts: ["172.19.2.49", "172.19.2.50", "172.19.2.51", "172.19.2.140", "172.19.2.141"]
	discovery.zen.minimum_master_nodes: 3

	#在172.19.2.50配置
	vim /etc/elasticsearch/elasticsearch.yml
	cluster.name: ELK_logs
	node.name: es-node2
	http.port: 9200
	network.host: 172.19.2.50
	discovery.zen.ping.unicast.hosts: ["172.19.2.49", "172.19.2.50", "172.19.2.51", "172.19.2.140", "172.19.2.141"]
	discovery.zen.minimum_master_nodes: 3

	#在172.19.2.51配置
	vim /etc/elasticsearch/elasticsearch.yml
	cluster.name: ELK_logs
	node.name: es-node3
	http.port: 9200
	network.host: 172.19.2.51
	discovery.zen.ping.unicast.hosts: ["172.19.2.49", "172.19.2.50", "172.19.2.51", "172.19.2.140", "172.19.2.141"]
	discovery.zen.minimum_master_nodes: 3

	#在172.19.2.140配置
	vim /etc/elasticsearch/elasticsearch.yml
	cluster.name: ELK_logs
	node.name: es-node4
	http.port: 9200
	network.host: 172.19.2.140
	discovery.zen.ping.unicast.hosts: ["172.19.2.49", "172.19.2.50", "172.19.2.51", "172.19.2.140", "172.19.2.141"]
	discovery.zen.minimum_master_nodes: 3

	#在172.19.2.141配置
	vim /etc/elasticsearch/elasticsearch.yml
	cluster.name: ELK_logs
	node.name: es-node5
	http.port: 9200
	network.host: 172.19.2.141
	discovery.zen.ping.unicast.hosts: ["172.19.2.49", "172.19.2.50", "172.19.2.51", "172.19.2.140", "172.19.2.141"]
	discovery.zen.minimum_master_nodes: 3

	#在5台测试环境的主机上分别执行
	service elasticsearch start
	netstat -tnlp | grep 9200

	#在172.19.2.141上部署nginx反代elasticsearch集群中的3台性能消耗低的主机
	elasticsearch集群被反代至了172.19.2.141的9100端口上
	yum install nginx

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
		access_log      off;
		sendfile        on;
		keepalive_timeout  65;

		upstream elk {
			server 172.19.2.50:9200 weight=1 max_fails=2 fail_timeout=1;
			server 172.19.2.51:9200 weight=1 max_fails=2 fail_timeout=1;
			server 172.19.2.140:9200 weight=2 max_fails=2 fail_timeout=1;
			}

		server {
			listen       9100;
			access_log  /var/log/nginx/accesse.log  logstash_json;

			location / {
				proxy_pass   http://elk/;
				proxy_set_header Host $host;
				proxy_set_header X-Real-IP $remote_addr;
				}
			}

		server {
			listen       80;
			access_log  /var/log/nginx/k8saccesse.log  logstash_json;

			location / {
				proxy_pass   http://172.19.2.49:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/;
				proxy_set_header Host $host;
				proxy_set_header X-Real-IP $remote_addr;
				}
			}


		include /etc/nginx/conf.d/*.conf;
	}

	service nginx start


# 二、配置部署kibana #

在172.19.2.141部署kibana

	mkdir -pv /home/lvqingshan/
	cd /home/lvqingshan/
	wget https://artifacts.elastic.co/downloads/kibana/kibana-5.2.0-x86_64.rpm
	rpm -ivh kibana-5.2.0-x86_64.rpm

	vim /etc/kibana/kibana.yml
	server.host: "172.19.2.141"
	elasticsearch.url: "http://172.19.2.141:9100"

	service kibana start

启动后访问

http://172.19.2.141:5601

# 三、配置部署cerebro #

作用：替代kopf使用，用来查看集群状态和索引状态

官网位置：

https://github.com/lmenezes/cerebro

	#在172.19.2.50上部署cerebro
	cd /home/lvqingshan/
	wget https://github.com/lmenezes/cerebro/releases/download/v0.6.5/cerebro-0.6.5.tgz
	tar xvf cerebro-0.6.5.tgz
	cd cerebro-0.6.5
	#修改配置文件，只需要host填上本机的es地址和name填上es集群名
	vim /home/lvqingshan/cerebro-0.6.5/conf/application.conf
	secret = "ki:s:[[@=Ag?QI`W2jMwkY:eqvrJ]JqoJyi2axj3ZvOv^/KavOT4ViJSv?6YY4[N"
	basePath = "/"
	rest.history.size = 50 // defaults to 50 if not specified
	data.path = "./cerebro.db"
	auth = {
	}
	hosts = [
	  {
		host = "http://172.19.2.50:9200"
		name = "ELK_logs"
	  },
	]

	nohup /home/lvqingshan/cerebro-0.6.5/bin/cerebro -Dhttp.port=1234 -Dhttp.address=172.19.2.50 &

cerebro访问地址：http://172.19.2.50:1234


# 四、配置logstash和filebeat #

其中5.2版本是发送给elk的，1.3版本是发送给Graylog

在192.168.38.50和192.168.38.51上部署

	mkdir -pv /root/elklog
	cd /root/elklog/
	wget http://172.19.2.139/filebeat-1.3.0-x86_64.tar.gz
	wget http://172.19.2.139/filebeat-5.2.0-linux-x86_64.tar.gz
	tar xvf filebeat-1.3.0-x86_64.tar.gz
	tar xvf filebeat-5.2.0-linux-x86_64.tar.gz
	mv filebeat-1.3.0-x86_64 filebeat-1.3
	mv filebeat-5.2.0-linux-x86_64 filebeat-5.2

	vim /root/elklog/filebeat-5.2/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /home/app/nettyLogs/netty.log
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "netty-%{+yyyy.MM.dd}"

	vim /root/elklog/filebeat-1.3/filebeat.yml
	filebeat:
	  prospectors:
		-
		  paths:
			- /home/app/nettyLogs/netty.log
		  input_type: log
	output:
	  logstash:
		hosts: ["172.19.2.139:5000"]
	shipper:
	  name: "192.168.38.50"
	logging:
	  files:
		rotateeverybytes: 10485760 # = 10MB

	nohup /root/elklog/filebeat-1.3/filebeat -c /root/elklog/filebeat-1.3/filebeat.yml &
	nohup /root/elklog/filebeat-5.2/filebeat -c /root/elklog/filebeat-5.2/filebeat.yml &

在192.168.38.52和192.168.38.53上部署

	mkdir -pv /root/elklog
	cd /root/elklog/
	wget http://172.19.2.139/filebeat-1.3.0-x86_64.tar.gz
	wget http://172.19.2.139/filebeat-5.2.0-linux-x86_64.tar.gz
	tar xvf filebeat-1.3.0-x86_64.tar.gz
	tar xvf filebeat-5.2.0-linux-x86_64.tar.gz
	mv filebeat-1.3.0-x86_64 filebeat-1.3
	mv filebeat-5.2.0-linux-x86_64 filebeat-5.2

	vim /root/elklog/filebeat-5.2/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/route-war/logs/platform-log/root_*.log
	- input_type: log
	  paths:
		- /usr/local/route-war/logs/platform-log/route_war_*.log
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "route-war-%{+yyyy.MM.dd}"

	vim /root/elklog/filebeat-1.3/filebeat.yml
	filebeat:
	  prospectors:
		-
		  paths:
			- /usr/local/route-war/logs/platform-log/root_*.log
		  input_type: log
		-
		  paths:
			- /usr/local/route-war/logs/platform-log/route_war_*.log
		  input_type: log
	output:
	  logstash:
		hosts: ["172.19.2.139:5001"]
	shipper:
	  name: "192.168.38.52"
	logging:
	  files:
		rotateeverybytes: 10485760 # = 10MB

	nohup /root/elklog/filebeat-1.3/filebeat -c /root/elklog/filebeat-1.3/filebeat.yml &
	nohup /root/elklog/filebeat-5.2/filebeat -c /root/elklog/filebeat-5.2/filebeat.yml &

在192.168.38.73和192.168.38.74上部署

	mkdir -pv /root/elklog
	cd /root/elklog/
	wget http://172.19.2.139/filebeat-1.3.0-x86_64.tar.gz
	wget http://172.19.2.139/filebeat-5.2.0-linux-x86_64.tar.gz
	tar xvf filebeat-1.3.0-x86_64.tar.gz
	tar xvf filebeat-5.2.0-linux-x86_64.tar.gz
	mv filebeat-1.3.0-x86_64 filebeat-1.3
	mv filebeat-5.2.0-linux-x86_64 filebeat-5.2

	vim /root/elklog/filebeat-5.2/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/kafkaservice/logs/kafkaservice.log.*
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "kafka-%{+yyyy.MM.dd}"


	vim /root/elklog/filebeat-1.3/filebeat.yml
	filebeat:
	  prospectors:
		-
		  paths:
			- /usr/local/kafkaservice/logs/kafkaservice.log.*
		  input_type: log
	output:
	  logstash:
		hosts: ["172.19.2.139:5002"]
	shipper:
	  name: "192.168.38.73"
	logging:
	  files:
		rotateeverybytes: 10485760 # = 10MB

	nohup /root/elklog/filebeat-1.3/filebeat -c /root/elklog/filebeat-1.3/filebeat.yml &
	nohup /root/elklog/filebeat-5.2/filebeat -c /root/elklog/filebeat-5.2/filebeat.yml &

在192.168.38.52上部署

	cd /root/elklog/
	cp -a filebeat-1.3 filebeat-5.2job
	cp -a filebeat-5.2 filebeat-5.2job

	vim /root/elklog/filebeat-5.2job/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/route-job/logs/route_job.log
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "route-job-%{+yyyy.MM.dd}"

	vim /root/elklog/filebeat-1.3job/filebeat.yml
	filebeat:
	  prospectors:
		-
		  paths:
			- /usr/local/route-job/logs/route_job.log
		  input_type: log
	output:
	  logstash:
		hosts: ["172.19.2.139:5003"]
	shipper:
	  name: "192.168.38.52"
	logging:
	  files:
		rotateeverybytes: 10485760 # = 10MB

	nohup /root/elklog/filebeat-1.3job/filebeat -c /root/elklog/filebeat-1.3job/filebeat.yml &
	nohup /root/elklog/filebeat-5.2job/filebeat -c /root/elklog/filebeat-5.2job/filebeat.yml &

在192.168.38.54和192.168.38.55上部署

	mkdir -pv /root/elklog
	cd /root/elklog/
	wget http://172.19.2.139/filebeat-1.3.0-x86_64.tar.gz
	wget http://172.19.2.139/filebeat-5.2.0-linux-x86_64.tar.gz
	tar xvf filebeat-1.3.0-x86_64.tar.gz
	tar xvf filebeat-5.2.0-linux-x86_64.tar.gz
	mv filebeat-1.3.0-x86_64 filebeat-1.3
	mv filebeat-5.2.0-linux-x86_64 filebeat-5.2

	vim /root/elklog/filebeat-5.2/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/router-center-api/logs/root_*.log
	- input_type: log
	  paths:
		- /usr/local/router-center-api/logs/router_center_api_*.log
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "router-center-api-%{+yyyy.MM.dd}"

	vim /root/elklog/filebeat-1.3/filebeat.yml
	filebeat:
	  prospectors:
		-
		  paths:
			- /usr/local/router-center-api/logs/root_*.log
		  input_type: log
		-
		  paths:
			- /usr/local/router-center-api/logs/router_center_api_*.log
		  input_type: log
	output:
	  logstash:
		hosts: ["172.19.2.139:5004"]
	shipper:
	  name: "192.168.38.54"
	logging:
	  files:
		rotateeverybytes: 10485760 # = 10MB

	nohup /root/elklog/filebeat-1.3/filebeat -c /root/elklog/filebeat-1.3/filebeat.yml &
	nohup /root/elklog/filebeat-5.2/filebeat -c /root/elklog/filebeat-5.2/filebeat.yml &

在192.168.38.56和192.168.38.57上部署

	mkdir -pv /root/elklog
	cd /root/elklog/
	wget http://172.19.2.139/filebeat-1.3.0-x86_64.tar.gz
	wget http://172.19.2.139/filebeat-5.2.0-linux-x86_64.tar.gz
	tar xvf filebeat-1.3.0-x86_64.tar.gz
	tar xvf filebeat-5.2.0-linux-x86_64.tar.gz
	mv filebeat-1.3.0-x86_64 filebeat-1.3
	mv filebeat-5.2.0-linux-x86_64 filebeat-5.2
	cp -a filebeat-5.2 filebeat-5.2redis

	vim /root/elklog/filebeat-5.2/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/RouterCenter/logs/router-center/root_*.log
	- input_type: log
	  paths:
		- /usr/local/RouterCenter/logs/router-center/routercenter_*.log
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "router-center-%{+yyyy.MM.dd}"

	vim /root/elklog/filebeat-1.3/filebeat.yml
	filebeat:
	  prospectors:
		-
		  paths:
			- /usr/local/RouterCenter/logs/router-center/redis_*.log
		  input_type: log
		-
		  paths:
			- /usr/local/RouterCenter/logs/router-center/root_*.log
		  input_type: log
		-   
		  paths:
			- /usr/local/RouterCenter/logs/router-center/routercenter_*.log
		  input_type: log
	output:
	  logstash:
		hosts: ["172.19.2.139:5005"]
	shipper:
	  name: "192.168.38.56"
	logging:
	  files:
		rotateeverybytes: 10485760 # = 10MB

	vim /root/elklog/filebeat-5.2redis/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/RouterCenter/logs/router-center/redis_*.log
	output.logstash:
	  hosts: ["172.19.2.51:5010"]

	nohup /root/elklog/filebeat-1.3/filebeat -c /root/elklog/filebeat-1.3/filebeat.yml &
	nohup /root/elklog/filebeat-5.2/filebeat -c /root/elklog/filebeat-5.2/filebeat.yml &
	nohup /root/elklog/filebeat-5.2redis/filebeat -c /root/elklog/filebeat-5.2redis/filebeat.yml &

对应的172.19.2.51的logstash配置

	mkdir -pv /logstash
	cd /logstash
	wget https://artifacts.elastic.co/downloads/logstash/logstash-5.2.0.tar.gz
	tar xvf logstash-5.2.0.tar.gz
	mv logstash-5.2.0 router-center-redis

	vim /logstash/router-center-redis/config/router-center-redis.conf
	input {
		beats {
			host => "172.19.2.51"
			port => "5010"
			codec => "json"
		}
	}
	output {
		elasticsearch {
			hosts => ["172.19.2.141:9100"]
			index => "routercenter-redis-%{+YYYY.MM.dd}" 
		}
	}

	nohup /logstash/router-center-redis/bin/logstash -f /logstash/router-center-redis/config/router-center-redis.conf

在192.168.38.59和192.168.38.60和192.168.38.61和192.168.38.62上部署

	mkdir -pv /root/elklog
	cd /root/elklog/
	wget http://172.19.2.139/filebeat-1.3.0-x86_64.tar.gz
	wget http://172.19.2.139/filebeat-5.2.0-linux-x86_64.tar.gz
	tar xvf filebeat-1.3.0-x86_64.tar.gz
	tar xvf filebeat-5.2.0-linux-x86_64.tar.gz
	mv filebeat-1.3.0-x86_64 filebeat-1.3
	mv filebeat-5.2.0-linux-x86_64 filebeat-5.2redis
	cp -a filebeat-5.2redis filebeat-5.2keepalived

	vim /root/elklog/filebeat-5.2redis/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/redis/logs/redis.log
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "redis-%{+yyyy.MM.dd}"

	vim /root/elklog/filebeat-5.2keepalived/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /usr/local/keepalived/logs/redis-state.log
	output.elasticsearch:
	  hosts: ["172.19.2.141:9100"]
	  index: "keepalived-%{+yyyy.MM.dd}"

	vim /root/elklog/filebeat-1.3/filebeat.yml
	filebeat:
	  prospectors:
		-
		  paths:
			- /usr/local/redis/logs/redis.log
		  input_type: log
		-
		  paths:
			- /usr/local/keepalived/logs/redis-state.log
		  input_type: log
	output:
	  logstash:
		hosts: ["172.19.2.139:5006"]
	shipper:
	  name: "192.168.38.59"
	logging:
	  files:
		rotateeverybytes: 10485760 # = 10MB

	nohup /root/elklog/filebeat-1.3/filebeat -c /root/elklog/filebeat-1.3/filebeat.yml &
	nohup /root/elklog/filebeat-5.2redis/filebeat -c /root/elklog/filebeat-5.2redis/filebeat.yml &
	nohup /root/elklog/filebeat-5.2keepalived/filebeat -c /root/elklog/filebeat-5.2keepalived/filebeat.yml &

在192.168.37.196和192.168.37.197和192.168.37.198上部署

	cd /home/app/
	wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.2.0-x86_64.rpm
	rpm -ivh filebeat-5.2.0-x86_64.rpm

	vim /etc/filebeat/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /apps/soft/tomcat/logs/catalina.out
	output.logstash:
	  hosts: ["172.19.2.51:5007"]

	service filebeat start

对应的172.19.2.51的logstash配置

	cd /logstash
	wget https://artifacts.elastic.co/downloads/logstash/logstash-5.2.0.tar.gz
	tar xvf logstash-5.2.0.tar.gz
	mv logstash-5.2.0 router-old-logstash

	vim /logstash/router-old-logstash/config/router_old_tomcat.conf
	input {
		beats {
			host => "172.19.2.51"
			port => "5007"
			codec => "json"
		}
	}

	output {
		elasticsearch {
			hosts => ["172.19.2.141:9100"]
			index => "router_old_tomcat-%{+YYYY.MM.dd}" 
		}
	}

	nohup /logstash/router-old-logstash/bin/logstash -f /logstash/router-old-logstash/config/router_old_tomcat.conf &

在192.168.37.234和192.168.37.235和192.168.37.236上部署

	cd /home/app/
	wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.2.0-x86_64.rpm
	rpm -ivh filebeat-5.2.0-x86_64.rpm

	vim /etc/filebeat/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /var/log/nginx/access.log
	output.logstash:
	  hosts: ["172.19.2.51:5008"]

	service filebeat start

对应的172.19.2.51的logstash配置

	cd /logstash/router-center-redis

	vim /logstash/router-center-redis/config/oldnginx.conf
	input {
		beats {
			host => "172.19.2.51"
			port => "5008"
			codec => "json"
		}
	}

	output {
		elasticsearch {
			hosts => ["172.19.2.141:9100"]
			index => "oldnginx-%{+YYYY.MM.dd}" 
		}
	}

	nohup /logstash/router-center-redis/bin/logstash -f /logstash/router-center-redis/config/oldnginx.conf &

在192.168.38.65和192.168.38.68和192.168.38.69上部署

	cd /home/app/
	wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.2.0-x86_64.rpm
	rpm -ivh filebeat-5.2.0-x86_64.rpm

	vim /etc/filebeat/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /var/lib/mysql/slowsql.sql
	  multiline.pattern: ^\[
	  multiline.negate: true
	  multiline.match: after
	output.logstash:
	  hosts: ["172.19.2.51:5045"]

	service filebeat start

对应的172.19.2.51的logstash配置

	cd /logstash
	wget https://artifacts.elastic.co/downloads/logstash/logstash-5.2.0.tar.gz
	tar xvf logstash-5.2.0.tar.gz
	mv logstash-5.2.0 router-new-logstash

	vim /logstash/router-new-logstash/config/new-mysql-slow.conf
	input {
	  beats {
		port => 5045
	 }
	}
	filter {
	 grok {
	  match => { "message" => "(?m)^#\s+User@Host:\s+%{DATA:user[host]}\s+@\s+\[%{IPV4:ipaddr}\](\s+Id:\s+(?:[+-]?(?:[0-9]+)))?\n#\s+Query_time:\s+%{BASE16FLOAT:query_time}\s+Lock_time:\s+%{BASE16FLOAT:lock_time}\s+Rows_sent:\s+(?:[+-]?(?:[0-9]+))\s+Rows_examined:\s+(?:[+-]?(?:[0-9]+))\n(?<database>(.*?)\s+(.*?);\n)?SET\s+timestamp\=(?:[+-]?(?:[0-9]+));\n(?<sql_command>.*?);$" }
	 }
	 if "_grokparsefailure" in [tags] {
	  drop {}
	 }
	  mutate {
		convert => ["query_time","float"]
	 }
	}
	output {
	 elasticsearch {
	  hosts => ["172.19.2.141:9100"]
	  index => "new-mysql-slow-%{+YYYY.MM.dd}"
	 }
	}

	nohup /logstash/router-new-logstash/bin/logstash -f /logstash/router-new-logstash/config/new-mysql-slow.conf &

在192.168.37.230上部署

	cd /home/app/
	wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.2.0-x86_64.rpm
	rpm -ivh filebeat-5.2.0-x86_64.rpm

	vim /etc/filebeat/filebeat.yml
	filebeat.prospectors:
	- input_type: log
	  paths:
		- /apps/data/mysql/slow-query.log
	  multiline.pattern: ^\[
	  multiline.negate: true
	  multiline.match: after
	output.logstash:
	  hosts: ["172.19.2.51:5044"]

对应的172.19.2.51的logstash配置

	cd /logstash/router-old-logstash

	vim /logstash/router-old-logstash/config/mysql37230.yml
	input {
	  beats {
		port => 5044
	 }
	}

	filter {
	 grok {
	  match => { "message" => "(?m)^#\s+User@Host:\s+%{DATA:user[host]}\s+@\s+\[%{IPV4:ipaddr}\](\s+Id:\s+(?:[+-]?(?:[0-9]+)))?\n#\s+Query_time:\s+%{BASE16FLOAT:query_time}\s+Lock_time:\s+%{BASE16FLOAT:lock_time}\s+Rows_sent:\s+(?:[+-]?(?:[0-9]+))\s+Rows_examined:\s+(?:[+-]?(?:[0-9]+))\n(?<database>(.*?)\s+(.*?);\n)?SET\s+timestamp\=(?:[+-]?(?:[0-9]+));\n(?<sql_command>.*?);$" }
	 }
	 if "_grokparsefailure" in [tags] {
	  drop {}
	 }
	  mutate {
		convert => ["query_time","float"]
	 }
	}

	output {
	 elasticsearch {
	  hosts => ["172.19.2.141:9100"]
	  index => "old-mysql-slow-%{+YYYY.MM.dd}"
	 }
	}

	nohup /logstash/router-old-logstash/bin/logstash -f /logstash/router-old-logstash/config/mysql37230.yml &



