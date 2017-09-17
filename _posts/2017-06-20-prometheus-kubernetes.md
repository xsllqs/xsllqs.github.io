---
date: 2017-06-20 08:12:52+08:00
layout: post
title: prometheus监控kubernetes
categories: linux
tags: kubernetes prometheus
---


# 一、在kuber-master（172.19.2.49）上下载github上的prometheus项目 #

	mkdir -pv /home/lvqingshan/prometheus/
	git clone https://github.com/coreos/prometheus-operator.git

# 二、进入prometheus目录，修改映射端口 #

	cd /home/lvqingshan/prometheus/prometheus-operator/contrib/kube-prometheus

	#修改nodePort到kubernetes允许的范围
	vim manifests/prometheus/prometheus-k8s-service.yaml
		nodePort: 8990

	vim manifests/grafana/grafana-service.yaml
		nodePort: 8992

	vim manifests/alertmanager/alertmanager-service.yaml
		nodePort: 8993

# 三、部署prometheus #

	cd /home/lvqingshan/prometheus/prometheus-operator/contrib/kube-prometheus
	#注意因为prometheus的部署脚本用了相对路径，所以一定要进入克隆到本地的prometheus的prometheus-operator/contrib/kube-prometheus目录来执行prometheus的部署与移除

	#部署：
	hack/cluster-monitoring/deploy
	#移除:
	hack/cluster-monitoring/teardown

	#因为网络原因，下载images的速度会非常非常慢，至少2-4小时才会部署好
	#查看是否部署好用：
	kubectl get pods -n monitoring
	#如果结果全部为run则部署完成

# 四、部署完成后查看 #

进入kubernets-dashboard查看grafana对应的pod所在的主机

找到service中grafana的NodePort端口

浏览器中访问

http://172.19.2.51:8992
