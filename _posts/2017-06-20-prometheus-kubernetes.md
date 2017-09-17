---
date: 2017-06-20 11:12:52+08:00
layout: post
title: prometheus���kubernetes
categories: linux
tags: kubernetes prometheus
---

# һ����kuber-master��172.19.2.49��������github�ϵ�prometheus��Ŀ #

	mkdir -pv /home/lvqingshan/prometheus/
	git clone https://github.com/coreos/prometheus-operator.git

# ��������prometheusĿ¼���޸�ӳ��˿� #

	cd /home/lvqingshan/prometheus/prometheus-operator/contrib/kube-prometheus

	#�޸�nodePort��kubernetes����ķ�Χ
	vim manifests/prometheus/prometheus-k8s-service.yaml
		nodePort: 8990

	vim manifests/grafana/grafana-service.yaml
		nodePort: 8992

	vim manifests/alertmanager/alertmanager-service.yaml
		nodePort: 8993

# ��������prometheus #

	cd /home/lvqingshan/prometheus/prometheus-operator/contrib/kube-prometheus
	#ע����Ϊprometheus�Ĳ���ű��������·��������һ��Ҫ�����¡�����ص�prometheus��prometheus-operator/contrib/kube-prometheusĿ¼��ִ��prometheus�Ĳ������Ƴ�

	#����
	hack/cluster-monitoring/deploy
	#�Ƴ�:
	hack/cluster-monitoring/teardown

	#��Ϊ����ԭ������images���ٶȻ�ǳ��ǳ���������2-4Сʱ�ŻᲿ���
	#�鿴�Ƿ�����ã�
	kubectl get pods -n monitoring
	#������ȫ��Ϊrun�������

# �ġ�������ɺ�鿴 #

����kubernets-dashboard�鿴grafana��Ӧ��pod���ڵ�����

�ҵ�service��grafana��NodePort�˿�

������з���

http://172.19.2.51:8992
