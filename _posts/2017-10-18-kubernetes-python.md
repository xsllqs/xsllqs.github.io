---
date: 2017-10-19 01:15:24+08:00
layout: post
title: jenkins触发式自动构建python应用镜像并发布至kubernetes集群
categories: linux
tags: jenkins docker harbor kubernetes k8s python
---


# 一、制作Dockerfile文件 #

1.在172.19.2.51上部署

上传安装包至该目录并解压

	mkdir -pv /opt/git/obd
	cd /opt/git/obd
	tar zxvf flask.tar.gz

	vim Dockerfile
	FROM python:2.7
	RUN mkdir -pv /opt/flask
	ADD flask /opt/flask
	RUN pip install flask
	RUN pip install Flask-MYSQL
	EXPOSE 5000
	CMD ["python","/opt/flask/server.py"]

	ls /home/app/test/cmdb
	Dockerfile  flask  flask.tar.gz

2.测试dockerfile是否能正常工作

	docker build -t obd:v1 ./
	docker run -p 31500:5000 -idt obd:v1
	git add -A
	git commit
	git push -u origin master
	gitlab账号lvqingshan
	密码abcd1234


# 二、配置登录habor仓库（仓库为172.19.2.139） #

1.在192.168.13.45上配置仓库私钥

	mkdir -pv /etc/docker/certs.d/172.19.2.139/
	vim /etc/docker/certs.d/172.19.2.139/ca.crt
	-----BEGIN CERTIFICATE-----
	MIIDvjCCAqagAwIBAgIUQzFZBuFh7EZLOzWUYZ10QokL+BUwDQYJKoZIhvcNAQEL
	BQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0Jl
	aUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwpr
	dWJlcm5ldGVzMB4XDTE3MDcwNDA4NTMwMFoXDTIyMDcwMzA4NTMwMFowZTELMAkG
	A1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxDDAK
	BgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwprdWJlcm5ldGVz
	MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyWgHFV6Cnbgxcs7X7ujj
	APnnMmotzNnnTRhygJLCMpCZUaWYrdBkFE4T/HGpbYi1R5AykSPA7FCffFHpJIf8
	Gs5DAZHmpY/uRsLSrqeP7/D8sYlyCpggVUeQJviV/a8L7PkCyGq9DSiU/MUBg4CV
	Dw07OT46vFJH0lzTaZJNSz7E5QsekLyzRb61tZiBN0CJvSOxXy7wvdqK0610OEFM
	T6AN8WfafTH4qmKWulFBJN1LjHTSYfTZzCL6kfTSG1M3kqG0W4B2o2+TkNLVmC9n
	gEKdeh/yQmQWfraRkuWiCorJZGxte27xpjgu7u62sRyCm92xQRNgp5RiGHxP913+
	HQIDAQABo2YwZDAOBgNVHQ8BAf8EBAMCAQYwEgYDVR0TAQH/BAgwBgEB/wIBAjAd
	BgNVHQ4EFgQUDFiYOhMMWkuq93iNBoC1Udr9wLIwHwYDVR0jBBgwFoAUDFiYOhMM
	Wkuq93iNBoC1Udr9wLIwDQYJKoZIhvcNAQELBQADggEBADTAW0FPhfrJQ6oT/WBe
	iWTv6kCaFoSuWrIHiB9fzlOTUsicrYn6iBf+XzcuReZ6qBILghYGPWPpOmnap1dt
	8UVl0Shdj+hyMbHzxR0XzX12Ya78Lxe1GFg+63XbxNwOURssd9DalJixKcyj2BW6
	F6JG1aBQhdgGSBhsCDvG1zawqgZX/h4VWG55Kv752PYBrQOtUH8CS93NfeB5Q7bE
	FOuyvGVd1iO40JQLoFIkZuyxNh0okGjfmT66dia7g+bC0v1SCMiE/UJ9uvHvfPYe
	qLkSRjIHH7FH1lQ/AKqjl9qrpZe7lHplskQ/jynEWHcb60QRcAWPyd94OPrpLrTU
	64g=
	-----END CERTIFICATE-----

2.登录仓库

	docker login 172.19.2.139
	Username: admin
	Password: Cmcc@1ot

3.上传镜像测试

在habor上创建ops仓库后才能push

	docker tag obd:v1 192.168.13.45/ops/obd
	docker login -p admin -u Cmcc@1ot -e 172.19.2.139
	docker push 192.168.13.45/ops/obd

	
# 三、kubernets文件配置 #

在kubernetes主节点172.19.2.50上配置

	vim /opt/kube-obd/obd-rc1.yaml
	apiVersion: v1
	kind: ReplicationController
	metadata:
	  name: obd
	spec:
	  replicas: 2		#pod数量
	  selector:
		app: obd
	  template:
		metadata:
		  labels:
			app: obd
		spec:
		  containers:
		  - image: 172.19.2.139/ops/obd:latest		#使用的镜像
			name: obd
			resources:
			  limits:
				cpu: "2"		#分配给pod的CPU资源
				memory: 2Gi		#分配给pod的内存资源
			ports:
			- containerPort: 5000		#开放的端口

	vim /opt/kube-obd/obd-svc1.yaml
	apiVersion: v1
	kind: Service
	metadata:
	  name: obd
	spec:
	  ports:
	  - name: obd-svc
		port: 5000
		targetPort: 5000
		nodePort: 31500		#proxy映射出来的端口
	  selector:
		app: obd
	  type: NodePort		#映射端口类型


# 四、jenkins配置 #

1.General中配置参数化构建过程

	新增String Parameter

	名字：VERSION

	默认值：[空]

	描述：请输入版本号

![1](https://xsllqs.github.io/assets/2017-10-18-kubernetes-python1.png)


2.源码管理Git设置

	Repository URL 为http://172.19.2.140:18080/lijun/obd.git

![2](https://xsllqs.github.io/assets/2017-10-18-kubernetes-python2.png)

3.设置Gitlab出现变更自动触发构建

一分钟检测一次gitlab项目是否有变化

	*/1 * * * *

![3](https://xsllqs.github.io/assets/2017-10-18-kubernetes-python3.png)
![4](https://xsllqs.github.io/assets/2017-10-18-kubernetes-python4.png)

4.Execute shell设置

两种控制版本的方式，当自动触发构建或者版本号为空时使用时间戳作为版本，当填入版本号时使用填入的版本号

	imagesid=`docker images | grep -i obd | awk '{print $3}'| head -1`
	project=/var/lib/jenkins/jobs/odb-docker-build/workspace

	if [ -z "$VERSION" ];then
		VERSION=`date +%Y%m%d%H%M`
	fi

	echo $VERSION

	if [ -z "$imagesid" ];then
		echo $imagesid "is null"
	else
		docker rmi -f $imagesid 
	fi

	docker build -t obd:$VERSION $project


	docker tag obd:$VERSION 172.19.2.139/ops/obd:$VERSION
	docker tag obd:$VERSION 172.19.2.139/ops/obd:latest
	docker login -u admin -p Cmcc@1ot 172.19.2.139
	docker push 172.19.2.139/ops/obd:$VERSION
	docker push 172.19.2.139/ops/obd:latest


![5](https://xsllqs.github.io/assets/2017-10-18-kubernetes-python5.png)

5.ansible-playbook配置

	vim /home/app/ansible/playbooks/obd/obd.yaml
	- hosts: 172.19.2.50
	  remote_user: app
	  sudo: yes
	  tasks:
		- name: 关闭原有pod
		  shell: kubectl delete -f /opt/kube-obd
		  ignore_errors: yes
		- name: 启动新pod
		  shell: kubectl create -f /opt/kube-obd

![6](https://xsllqs.github.io/assets/2017-10-18-kubernetes-python6.png)