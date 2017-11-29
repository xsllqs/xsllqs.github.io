---
date: 2017-11-01 03:12:24+08:00
layout: post
title: kubernetes中部署mysql集群并持久化存储
categories: linux
tags: docker harbor kubernetes k8s mysql集群
---


# 简介： #


参考：https://github.com/Yolean/kubernetes-mysql-cluster

环境

	主节点：172.19.2.50
	从节点：
	172.19.2.51
	172.19.2.140

部署完成后通过各节点的30336端口访问mysql

	账号root，密码abcd1234
	如：
	mysql -h 172.19.2.50 -P 30336 -uroot -pabcd1234

部署完成后通过galera可以让集群3个节点间的数据一致

容器内访问mysql时，可以通过所有k8s节点的30336端口访问，也可以使用k8s服务中的内部入口访问，如mysql.mysql:3306，dns会自动解析mysql.mysql到对应的服务集群

# 一、在主节点创建目录 #

	mkdir -pv /mysql_data/datadir-mariadb-0
	mkdir -pv /mysql_data/datadir-mariadb-1
	mkdir -pv /mysql_data/datadir-mariadb-2

# 二、修改部署文件 #

	cd /opt/kubernetes-mysql-cluster

命名空间部署文件

	vim 00namespace.yml
	---
	apiVersion: v1
	kind: Namespace
	metadata:
	  name: mysql

pvc部署文件

	vim 10pvc.yml
	---
	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: mysql-mariadb-0
	  namespace: mysql
	spec:
	  accessModes:
		- ReadWriteOnce		#这里为pvc的访问模式
	  resources:
		requests:
		  storage: 10Gi		#这里调整要挂载的pvc大小
	  selector:
		matchLabels:		#这里要和pv的标签对应
		  app: mariadb
		  podindex: "0"
	---
	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: mysql-mariadb-1
	  namespace: mysql
	spec:
	  accessModes:
		- ReadWriteOnce
	  resources:
		requests:
		  storage: 10Gi
	  selector:
		matchLabels:
		  app: mariadb
		  podindex: "1"
	---
	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: mysql-mariadb-2
	  namespace: mysql
	spec:
	  accessModes:
		- ReadWriteOnce
	  resources:
		requests:
		  storage: 10Gi
	  selector:
		matchLabels:
		  app: mariadb
		  podindex: "2"

mariadb服务文件

	vim 20mariadb-service.yml
	# the "Headless Service, used to control the network domain"
	---
	apiVersion: v1
	kind: Service
	metadata:
	  name: mariadb
	  namespace: mysql
	spec:
	  clusterIP: None
	  selector:
		app: mariadb
	  ports:
		- port: 3306
		  name: mysql
		- port: 4444
		  name: sst
		- port: 4567
		  name: replication
		- protocol: UDP
		  port: 4567
		  name: replicationudp
		- port: 4568
		  name: ist

mysql服务文件

	vim 30mysql-service.yml
	---
	apiVersion: v1
	kind: Service
	metadata:
	  name: mysql
	  namespace: mysql
	spec:
	  ports:
	  - port: 3306
		name: mysql
		targetPort: 3306
		nodePort: 30336		#这里为porxy映射端口
	  selector:
		app: mariadb
	  type: NodePort

载入配置文件

	vim 40configmap.sh
	#!/bin/bash
	DIR=`dirname "$BASH_SOURCE"`
	kubectl create configmap "conf-d" --from-file="$DIR/conf-d/" --namespace=mysql

输入mysql初始密码文件

	vim 41secret.sh
	#!/bin/bash
	echo -n Please enter mysql root password for upload to k8s secret:
	read -s rootpw
	echo
	kubectl create secret generic mysql-secret --namespace=mysql --from-literal=rootpw=$rootpw

部署有状态集群文件

	vim 50mariadb.yml
	apiVersion: apps/v1beta1
	kind: StatefulSet
	metadata:
	  name: mariadb
	  namespace: mysql
	spec:
	  serviceName: "mariadb"
	  replicas: 1		#这里是要启动的节点的数量
	  template:
		metadata:
		  labels:
			app: mariadb
		spec:
		  terminationGracePeriodSeconds: 10
		  containers:
			- name: mariadb
			  image: mariadb:10.1.22	#这里修改使用的镜像文件
			  ports:
				- containerPort: 3306
				  name: mysql
				- containerPort: 4444
				  name: sst
				- containerPort: 4567
				  name: replication
				- containerPort: 4567
				  protocol: UDP
				  name: replicationudp
				- containerPort: 4568
				  name: ist
			  env:
				- name: MYSQL_ROOT_PASSWORD
				  valueFrom:
					secretKeyRef:
					  name: mysql-secret
					  key: rootpw
				- name: MYSQL_INITDB_SKIP_TZINFO
				  value: "yes"
			  args:
				- --character-set-server=utf8mb4
				- --collation-server=utf8mb4_unicode_ci
				# Remove after first replicas=1 create
				- --wsrep-new-cluster	#这里在执行的时候代表会创建新集群，新增节点的时候要注释掉
			  volumeMounts:
				- name: mysql
				  mountPath: /var/lib/mysql
				- name: conf
				  mountPath: /etc/mysql/conf.d
				- name: initdb
				  mountPath: /docker-entrypoint-initdb.d
		  volumes:
			- name: conf
			  configMap:
				name: conf-d
			- name: initdb
			  emptyDir: {}
	  volumeClaimTemplates:
	  - metadata:
		  name: mysql
		spec:
		  accessModes: [ "ReadWriteOnce" ]
		  resources:
			requests:
			  storage: 10Gi

调整集群节点到3个的文件

	vim 70unbootstrap.sh
	#!/bin/bash
	DIR=`dirname "$BASH_SOURCE"`
	set -e
	set -x
	cp "$DIR/50mariadb.yml" "$DIR/50mariadb.yml.unbootstrap.yml"
	sed -i 's/replicas: 1/replicas: 3/' "$DIR/50mariadb.yml.unbootstrap.yml"
	sed -i 's/- --wsrep-new-cluster/#- --wsrep-new-cluster/' "$DIR/50mariadb.yml.unbootstrap.yml"
	kubectl apply -f "$DIR/50mariadb.yml.unbootstrap.yml"
	rm "$DIR/50mariadb.yml.unbootstrap.yml"

创建目录

	mkdir -pv bootstrap conf-d

创建pv脚本

	vim bootstrap/pv.sh
	#!/bin/bash
	echo "Note that in for example GKE a PetSet will have PersistentVolume(s) and PersistentVolumeClaim(s) created for it automatically"
	dir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && cd .. && pwd )"
	path="$dir/data"
	echo "Please enter a path where to store data during local testing: ($path)"
	read newpath
	[ -n "$newpath" ] && path=$newpath
	cat bootstrap/pv-template.yml | sed "s|/tmp/k8s-data|$path|" | kubectl create -f -

创建挂载pv文件

	vim bootstrap/pv-template.yml
	---
	apiVersion: v1
	kind: PersistentVolume
	metadata:
	  name: datadir-mariadb-0
	  labels:				#这里的标签要和pvc的matchLabels对应
		app: mariadb
		podindex: "0"
	spec:
	  accessModes:
	  - ReadWriteOnce		#这里是pv的访问模式，必须要与pvc相同
	  capacity:
		storage: 10Gi		#这里是要创建的pv的大小
	  hostPath:
		path: /mysql_data/datadir-mariadb-0		#这里为挂载到本地的路径
	---
	apiVersion: v1
	kind: PersistentVolume
	metadata:
	  name: datadir-mariadb-1
	  labels:
		app: mariadb
		podindex: "1"
	spec:
	  accessModes:
	  - ReadWriteOnce
	  capacity:
		storage: 10Gi
	  hostPath:
		path: /mysql_data/datadir-mariadb-1
	---
	apiVersion: v1
	kind: PersistentVolume
	metadata:
	  name: datadir-mariadb-2
	  labels:
		app: mariadb
		podindex: "2"
	spec:
	  accessModes:
	  - ReadWriteOnce
	  capacity:
		storage: 10Gi
	  hostPath:
		path: /mysql_data/datadir-mariadb-2

删除pv脚本

	vim bootstrap/rm.sh
	#!/bin/bash
	echo "Note that in for example GKE a PetSet will have PersistentVolume(s) and PersistentVolumeClaim(s) created for it automatically"
	dir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && cd .. && pwd )"
	path="$dir/data"
	echo "Please enter a path where to store data during local testing: ($path)"
	read newpath
	[ -n "$newpath" ] && path=$newpath
	cat bootstrap/pv-template.yml | sed "s|/tmp/k8s-data|$path|" | kubectl delete -f -

集群同步galera的配置文件

	vim conf-d/galera.cnf
	[server]
	[mysqld]
	[galera]
	wsrep_on=ON
	wsrep_provider="/usr/lib/galera/libgalera_smm.so"
	wsrep_cluster_address="gcomm://mariadb-0.mariadb,mariadb-1.mariadb,mariadb-2.mariadb"
	binlog_format=ROW
	default_storage_engine=InnoDB
	innodb_autoinc_lock_mode=2
	wsrep-sst-method=rsync
	bind-address=0.0.0.0
	[embedded]
	[mariadb]
	[mariadb-10.1]


# 三、在kubernetes上部署mariadb集群 #

kubernetes所有节点执行

	docker pull mariadb:10.1.22

主节点执行

	cd /opt/kubernetes-mysql-cluster
	sh bootstrap/pv.sh
	kubectl create -f 00namespace.yml
	kubectl create -f 10pvc.yml
	./40configmap.sh
	./41secret.sh
	设置数据库root密码为abcd1234
	kubectl create -f 20mariadb-service.yml
	kubectl create -f 30mysql-service.yml
	kubectl create -f 50mariadb.yml


# 四、增加集群节点至3个 #

	./70unbootstrap.sh

# 五、清理kubernetes上的mysql集群 #

主节点上执行

	cd /opt/kubernetes-mysql-cluster
	kubectl delete -f ./
	sh bootstrap/rm.sh

所有节点执行

	rm -rf /mysql_data/datadir-mariadb-0/*
	rm -rf /mysql_data/datadir-mariadb-1/*
	rm -rf /mysql_data/datadir-mariadb-2/*


