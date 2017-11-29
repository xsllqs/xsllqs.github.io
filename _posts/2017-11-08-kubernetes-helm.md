---
date: 2017-11-08 01:35:24+08:00
layout: post
title: kubernetes部署helm
categories: linux
tags: docker harbor helm kubernetes
---

# 环境 #

参考：https://docs.helm.sh/using_helm/#installing-helm

需要镜像

	gcr.io/kubernetes-helm/tiller:v2.7.0

重命名一个tag

	docker pull xsllqs/kubernetes-helm:v2.7.0
	docker tag xsllqs/kubernetes-helm:v2.7.0 gcr.io/kubernetes-helm/tiller:v2.7.0

上传到私有仓库

	docker tag xsllqs/kubernetes-helm:v2.7.0 172.19.2.139/xsllqs/kubernetes-helm/tiller:v2.7.0
	docker push 172.19.2.139/xsllqs/kubernetes-helm/tiller:v2.7.0

每个node都下载并重命名该节点

	docker pull 172.19.2.139/xsllqs/kubernetes-helm/tiller:v2.7.0
	docker tag 172.19.2.139/xsllqs/kubernetes-helm/tiller:v2.7.0 gcr.io/kubernetes-helm/tiller:v2.7.0

# 一、安装helm客户端和tiller #

修改环境变量

	vim /etc/bashrc
	export PATH="$PATH:/usr/local/bin"
	vim /etc/profile
	export PATH="$PATH:/usr/local/bin"
	source /etc/profile

RBAC授权

	kubectl create serviceaccount --namespace kube-system tiller
	kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
	kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

部署

	cd /opt/helm
	curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
	chmod 700 get_helm.sh
	./get_helm.sh
	helm init --tiller-namespace=kube-system

查看tiller是否安装成功

	kubectl get pods --namespace kube-system

测试Client和Server是否连接正常

	helm version

卸载

	kubectl delete deployment tiller-deploy --namespace kube-system

web-UI安装（本人未部署）

	helm install stable/nginx-ingress
	helm install stable/nginx-ingress --set controller.hostNetwork=true
	helm repo add monocular https://kubernetes-helm.github.io/monocular
	helm install monocular/monocular

添加国内可用仓库

	helm repo add opsgoodness http://charts.opsgoodness.com

# 二、应用的安装删除 #

安装redis应用

	helm install stable/redis-ha --version 0.2.3

如果上面执行不了就直接执行以下内容

	helm install https://kubernetes-charts.storage.googleapis.com/redis-ha-0.2.3.tgz

访问redis

	redis-cli -h torrid-tuatara-redis-ha.default.svc.cluster.local

安装kafka应用

	helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com
	helm install incubator/kafka --version 0.2.1

或者

	helm install https://kubernetes-charts-incubator.storage.googleapis.com/kafka-0.2.1.tgz

删除部署的应用

	helm ls
	helm delete {name}