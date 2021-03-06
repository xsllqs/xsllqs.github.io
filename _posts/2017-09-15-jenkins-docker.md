---
date: 2017-09-15 10:55:24+08:00
layout: post
title: jenkins触发式自动构建docker镜像上传至harbor并发布
categories: linux
tags: jenkins docker harbor
---


# 一、制作Dockerfile文件 #

1.在172.19.2.51上部署

	mkdir -pv /opt/git
	git clone http://172.19.2.140:18080/lvqingshan/gcgj.git
	cd /opt/git/gcgj
	scp app@172.19.2.1:/home/app/portal-tomcat/webapps/portal.war ./
	scp app@192.168.37.34:/home/app/portal-tomcat/conf/server.xml ./

	vim Dockerfile
	FROM tomcat:7.0.77-jre8
	ADD server.xml /usr/local/tomcat/conf
	RUN rm -rf /usr/local/tomcat/webapps/*
	COPY portal.war /usr/local/tomcat/webapps/ROOT.war
	EXPOSE 8080
	CMD ["/usr/local/tomcat/bin/catalina.sh","run"]

2.测试dockerfile是否能正常工作

	docker build -t gcgj/portal .
	docker run -p 38080:8080 -idt gcgj/portal:latest
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

在habor上创建gcgj仓库后才能push

	docker tag gcgj/portal:latest 192.168.13.45/gcgj/portal
	docker login -p admin -u Cmcc@1ot -e 172.19.2.139
	docker push 192.168.13.45/gcgj/portal


# 三、jenkins配置 #

1.General中配置参数化构建过程

	新增String Parameter

	名字：VERSION

	默认值：[空]

	描述：请输入版本号

![1](https://xsllqs.github.io/assets/2017-09-15-jenkins1.png)


2.源码管理Git设置

	Repository URL 为http://172.19.2.140:18080/lvqingshan/gcgj.git

![2](https://xsllqs.github.io/assets/2017-09-15-jenkins2.png)

3.设置Gitlab出现变更自动触发构建

一分钟检测一次gitlab项目是否有变化

	*/1 * * * *

![3](https://xsllqs.github.io/assets/2017-09-15-jenkins3.png)

4.Execute shell设置

两种控制版本的方式，当自动触发构建或者版本号为空时使用时间戳作为版本，当填入版本号时使用填入的版本号

	imagesid=`docker images | grep -i gcgj | awk '{print $3}'| head -1`
	project=/var/lib/jenkins/jobs/build-docker-router-portal/workspace

	if [ -z "$VERSION" ];then
		VERSION=`date +%Y%m%d%H%M`
	fi

	echo $VERSION

	if docker ps -a|grep -i gcgj;then
	   docker rm -f gcgj
	fi

	if [ -z "$imagesid" ];then
		echo $imagesid "is null"
	else
		docker rmi -f $imagesid 
	fi

	docker build -t gcgj/portal:$VERSION $project

	docker run -p 38080:8080 -idt --name gcgj gcgj/portal:$VERSION


	docker tag gcgj/portal:$VERSION 172.19.2.139/gcgj/portal:$VERSION
	docker login -u admin -p Cmcc@1ot 172.19.2.139
	docker push 172.19.2.139/gcgj/portal:$VERSION

![4](https://xsllqs.github.io/assets/2017-09-15-jenkins4.png)