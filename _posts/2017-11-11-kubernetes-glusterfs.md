---
date: 2017-11-11 01:35:24+08:00
layout: post
title: kubernetes中挂载glusterfs并使用
categories: linux
tags: docker harbor helm kubernetes
---

# 一、所有k8s节点安装glusterfs客户端 #

安装客户端

	yum install -y glusterfs glusterfs-fuse

在hosts中加入所有gluster的节点
	
	vim /etc/hosts
	172.19.12.193  gluster-manager
	172.19.12.194  gluster-node1
	172.19.12.195  gluster-node2

# 二、在kubernetes主节点部署 #

新建名称空间

	vim portal-ns1.yaml
	piVersion: v1
	kind: Namespace
	metadata:
	  name: gcgj-portal

新建endpoints

	cd /opt/glusterfs
	curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/volumes/glusterfs/glusterfs-endpoints.json
	vim glusterfs-endpoints.json
	{
	  "kind": "Endpoints",
	  "apiVersion": "v1",
	  "metadata": {
		"name": "glusterfs-cluster",
		"namespace": "gcgj-portal"	#如果后面要调用的pod有ns则一定要写ns
	  },
	  "subsets": [
		{
		  "addresses": [
			{
			  "ip": "172.19.12.193"
			}
		  ],
		  "ports": [
			{
			  "port": 1990	#这个端口自己随便写
			}
		  ]
		}
	  ]
	}

	kubectl apply -f glusterfs-endpoints.json
	kubectl get ep

新建服务

	curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/volumes/glusterfs/glusterfs-service.json
	vim glusterfs-service.json
	{
	  "kind": "Service",
	  "apiVersion": "v1",
	  "metadata": {
		"name": "glusterfs-cluster",
		"namespace": "gcgj-portal"
	  },
	  "spec": {
		"ports": [
		  {"port": 1990}
		]
	  }
	}

	kubectl apply -f glusterfs-service.json
	kubectl get svc

新建glusterfs的pod

	curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/examples/volumes/glusterfs/glusterfs-pod.json
	vim glusterfs-pod.json
	{
		"apiVersion": "v1",
		"kind": "Pod",
		"metadata": {
			"name": "glusterfs",
			"namespace": "gcgj-portal"
		},
		"spec": {
			"containers": [
				{
					"name": "glusterfs",
					"image": "nginx",
					"volumeMounts": [
						{
							"mountPath": "/mnt/glusterfs",	#自定义本地挂载glusterfs的目录
							"name": "glusterfsvol"
						}
					]
				}
			],
			"volumes": [
				{
					"name": "glusterfsvol",
					"glusterfs": {
						"endpoints": "glusterfs-cluster",
						"path": "models",
						"readOnly": true
					}
				}
			]
		}
	}


	kubectl apply -f glusterfs-pod.json
	kubectl get pods

创建pv

	vim glusterfs-pv.yaml
	apiVersion: v1
	kind: PersistentVolume
	metadata:
	  name: gluster-dev-volume
	spec:
	  capacity:
		storage: 8Gi	#pv申请的容量大小
	  accessModes:
		- ReadWriteMany
	  glusterfs:
		endpoints: "glusterfs-cluster"
		path: "models"
		readOnly: false

	kubectl apply -f glusterfs-pv.yaml
	kubectl get pv

创建pvc

	vim glusterfs-pvc.yaml
	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: glusterfs-gcgj
	  namespace: gcgj-portal
	spec:
	  accessModes:
		- ReadWriteMany
	  resources:
		requests:
		  storage: 8Gi

	kubectl apply -f glusterfs-pvc.yaml
	kubectl get pvc

新建应用，测试能否正常挂载

	cd /opt/kube-gcgj/portal-test
	vim portal-rc1.yaml
	apiVersion: v1
	kind: ReplicationController
	metadata:
	  name: gcgj-portal
	  namespace: gcgj-portal
	spec:
	  replicas: 1
	  selector:
		app: portal
	  template:
		metadata:
		  labels:
			app: portal
		spec:
		  containers:
		  - image: 172.19.2.139/gcgj/portal:latest
			name: portal
			resources:
			  limits:
				cpu: "1"
				memory: 2Gi
			ports:
			- containerPort: 8080
			volumeMounts:
			- mountPath: /usr/local/tomcat/logs		#需要挂载的目录
			  name: gcgj-portal-log			#这里的名字和下面的volumes的name要一致
		  volumes:
		  - name: gcgj-portal-log
			persistentVolumeClaim:
			  claimName: glusterfs-gcgj		#这里为pvc的名字

	vim portal-svc1.yaml
	apiVersion: v1
	kind: Service
	metadata:
	  name: gcgj-portal
	  namespace: gcgj-portal
	spec:
	  ports:
	  - name: portal-svc
		port: 8080
		targetPort: 8080
		nodePort: 30082
	  selector:
		app: portal
	  type: NodePort

	kubectl create -f /opt/kube-gcgj/portal-test

应用启动后到gluster集群对应的目录中查看是否有新日志生成