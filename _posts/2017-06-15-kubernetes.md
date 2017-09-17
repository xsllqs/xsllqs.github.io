---
date: 2017-06-15 11:12:52+08:00
layout: post
title: kubernetes��Ⱥ����
categories: linux
tags: kubernetes
---


�ο��ĵ���https://github.com/opsnull/follow-me-install-kubernetes-cluster

��װ����
172.19.2.49(kube-apiserver,kube-controller-manager,kube-dns,kube-proxy,kubectl,etcd)
172.19.2.50(kubectl,etcd,kube-proxy)
172.19.2.51(kubectl,etcd,kube-proxy)

# һ������ CA ֤�����Կ������ǰ���û������� #

172.19.2.49�Ͻ��в���

	mkdir -pv /root/local/bin
	vim /root/local/bin/environment.sh
	#!/usr/bin/bash

	export PATH=/root/local/bin:$PATH

	# TLS Bootstrapping ʹ�õ� Token������ʹ������ head -c 16 /dev/urandom | od -An -t x | tr -d ' ' ����
	BOOTSTRAP_TOKEN="11d74483444fb57f6a1cc114ed715949"

	# ���ʹ�� ����δ�õ����� ������������κ� Pod ����

	# �������� (Service CIDR��������ǰ·�ɲ��ɴ�����Ⱥ��ʹ��IP:Port�ɴ�
	SERVICE_CIDR="10.254.0.0/16"

	# POD ���� (Cluster CIDR��������ǰ·�ɲ��ɴ**�����**·�ɿɴ�(flanneld��֤)
	CLUSTER_CIDR="172.30.0.0/16"

	# ����˿ڷ�Χ (NodePort Range)
	export NODE_PORT_RANGE="8400-9000"

	# etcd ��Ⱥ�����ַ�б�
	export ETCD_ENDPOINTS="https://172.19.2.49:2379,https://172.19.2.50:2379,https://172.19.2.51:2379"

	# flanneld ��������ǰ׺
	export FLANNEL_ETCD_PREFIX="/kubernetes/network"

	# kubernetes ���� IP (һ���� SERVICE_CIDR �е�һ��IP)
	export CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

	# ��Ⱥ DNS ���� IP (�� SERVICE_CIDR ��Ԥ����)
	export CLUSTER_DNS_SVC_IP="10.254.0.2"

	# ��Ⱥ DNS ����
	export CLUSTER_DNS_DOMAIN="cluster.local."

	# ��ǰ����Ļ�������(��㶨�壬ֻҪ�����ֲ�ͬ��������)
	export NODE_NAME=etcd-host0 
	# ��ǰ����Ļ��� IP
	export NODE_IP=172.19.2.49
	# etcd ��Ⱥ���л��� IP
	export NODE_IPS="172.19.2.49 172.19.2.50 172.19.2.51"
	# etcd ��Ⱥ��ͨ�ŵ�IP�Ͷ˿� 
	export ETCD_NODES=etcd-host0=https://172.19.2.49:2380,etcd-host1=https://172.19.2.50:2380,etcd-host2=https://172.19.2.51:2380

	# �滻Ϊ kubernetes maste ��Ⱥ��һ���� IP
	export MASTER_IP=172.19.2.49
	export KUBE_APISERVER="https://${MASTER_IP}:6443"

	scp /root/local/bin/environment.sh app@172.19.2.50:/home/app
	scp /root/local/bin/environment.sh app@172.19.2.51:/home/app

172.19.2.50�Ļ�����������

	vim /home/app/environment.sh
	#!/usr/bin/bash

	export PATH=/root/local/bin:$PATH

	# TLS Bootstrapping ʹ�õ� Token������ʹ������ head -c 16 /dev/urandom | od -An -t x | tr -d ' ' ����
	BOOTSTRAP_TOKEN="11d74483444fb57f6a1cc114ed715949"

	# ���ʹ�� ����δ�õ����� ������������κ� Pod ����

	# �������� (Service CIDR��������ǰ·�ɲ��ɴ�����Ⱥ��ʹ��IP:Port�ɴ�
	SERVICE_CIDR="10.254.0.0/16"

	# POD ���� (Cluster CIDR��������ǰ·�ɲ��ɴ**�����**·�ɿɴ�(flanneld��֤)
	CLUSTER_CIDR="172.30.0.0/16"

	# ����˿ڷ�Χ (NodePort Range)
	export NODE_PORT_RANGE="8400-9000"

	# etcd ��Ⱥ�����ַ�б�
	export ETCD_ENDPOINTS="https://172.19.2.49:2379,https://172.19.2.50:2379,https://172.19.2.51:2379"

	# flanneld ��������ǰ׺
	export FLANNEL_ETCD_PREFIX="/kubernetes/network"

	# kubernetes ���� IP (һ���� SERVICE_CIDR �е�һ��IP)
	export CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

	# ��Ⱥ DNS ���� IP (�� SERVICE_CIDR ��Ԥ����)
	export CLUSTER_DNS_SVC_IP="10.254.0.2"

	# ��Ⱥ DNS ����
	export CLUSTER_DNS_DOMAIN="cluster.local."

	# ��ǰ����Ļ�������(��㶨�壬ֻҪ�����ֲ�ͬ��������)
	export NODE_NAME=etcd-host1
	# ��ǰ����Ļ��� IP
	export NODE_IP=172.19.2.50
	# etcd ��Ⱥ���л��� IP
	export NODE_IPS="172.19.2.49 172.19.2.50 172.19.2.51"
	# etcd ��Ⱥ��ͨ�ŵ�IP�Ͷ˿� 
	export ETCD_NODES=etcd-host0=https://172.19.2.49:2380,etcd-host1=https://172.19.2.50:2380,etcd-host2=https://172.19.2.51:2380
	# �滻Ϊ kubernetes maste ��Ⱥ��һ���� IP
	export MASTER_IP=172.19.2.49
	export KUBE_APISERVER="https://${MASTER_IP}:6443"

172.19.2.51�Ļ�����������

	vim /home/app/environment.sh
	#!/usr/bin/bash

	export PATH=/root/local/bin:$PATH

	# TLS Bootstrapping ʹ�õ� Token������ʹ������ head -c 16 /dev/urandom | od -An -t x | tr -d ' ' ����
	BOOTSTRAP_TOKEN="11d74483444fb57f6a1cc114ed715949"

	# ���ʹ�� ����δ�õ����� ������������κ� Pod ����

	# �������� (Service CIDR��������ǰ·�ɲ��ɴ�����Ⱥ��ʹ��IP:Port�ɴ�
	SERVICE_CIDR="10.254.0.0/16"

	# POD ���� (Cluster CIDR��������ǰ·�ɲ��ɴ**�����**·�ɿɴ�(flanneld��֤)
	CLUSTER_CIDR="172.30.0.0/16"

	# ����˿ڷ�Χ (NodePort Range)
	export NODE_PORT_RANGE="8400-9000"

	# etcd ��Ⱥ�����ַ�б�
	export ETCD_ENDPOINTS="https://172.19.2.49:2379,https://172.19.2.50:2379,https://172.19.2.51:2379"

	# flanneld ��������ǰ׺
	export FLANNEL_ETCD_PREFIX="/kubernetes/network"

	# kubernetes ���� IP (һ���� SERVICE_CIDR �е�һ��IP)
	export CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

	# ��Ⱥ DNS ���� IP (�� SERVICE_CIDR ��Ԥ����)
	export CLUSTER_DNS_SVC_IP="10.254.0.2"

	# ��Ⱥ DNS ����
	export CLUSTER_DNS_DOMAIN="cluster.local."

	# ��ǰ����Ļ�������(��㶨�壬ֻҪ�����ֲ�ͬ��������)
	export NODE_NAME=etcd-host2 
	# ��ǰ����Ļ��� IP
	export NODE_IP=172.19.2.51
	# etcd ��Ⱥ���л��� IP
	export NODE_IPS="172.19.2.49 172.19.2.50 172.19.2.51"
	# etcd ��Ⱥ��ͨ�ŵ�IP�Ͷ˿� 
	export ETCD_NODES=etcd-host0=https://172.19.2.49:2380,etcd-host1=https://172.19.2.50:2380,etcd-host2=https://172.19.2.51:2380

	# �滻Ϊ kubernetes maste ��Ⱥ��һ���� IP
	export MASTER_IP=172.19.2.49
	export KUBE_APISERVER="https://${MASTER_IP}:6443"


172.19.2.49��172.19.2.50��172.19.2.51�϶�ִ��

	mv /home/app/environment.sh /root/local/bin/
	chown root:root /root/local/bin/environment.sh 
	chmod 777 /root/local/bin/environment.sh
	source /root/local/bin/environment.sh

	wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
	chmod +x cfssl_linux-amd64
	cp cfssl_linux-amd64 /root/local/bin/cfssl

	wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
	chmod +x cfssljson_linux-amd64
	cp cfssljson_linux-amd64 /root/local/bin/cfssljson

	wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
	chmod +x cfssl-certinfo_linux-amd64
	cp cfssl-certinfo_linux-amd64 /root/local/bin/cfssl-certinfo

	export PATH=/root/local/bin:$PATH

172.19.2.49��ִ�в���

	mkdir ssl
	cd ssl
	cfssl print-defaults config > config.json
	cfssl print-defaults csr > csr.json

	cat > ca-config.json << EOF
	{
	  "signing": {
		"default": {
		  "expiry": "8760h"
		},
		"profiles": {
		  "kubernetes": {
			"usages": [
				"signing",
				"key encipherment",
				"server auth",
				"client auth"
			],
			"expiry": "8760h"
		  }
		}
	  }
	}
	EOF

	cat > ca-csr.json << EOF
	{
	  "CN": "kubernetes",
	  "key": {
		"algo": "rsa",
		"size": 2048
	  },
	  "names": [
		{
		  "C": "CN",
		  "ST": "BeiJing",
		  "L": "BeiJing",
		  "O": "k8s",
		  "OU": "System"
		}
	  ]
	}
	EOF

	cfssl gencert -initca ca-csr.json | cfssljson -bare ca

	ls ca*
	ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem

	mkdir -pv /etc/kubernetes/ssl
	cp ca* /etc/kubernetes/ssl
	scp ca* root@172.19.2.50:/home/app/ca
	scp ca* root@172.19.2.51:/home/app/ca

172.19.2.50��172.19.2.51��ִ�в���

	chown -R root:root /home/app/ca
	mkdir -pv /etc/kubernetes/ssl
	cp /home/app/ca/ca* /etc/kubernetes/ssl


# ��������߿���etcd��Ⱥ #

172.19.2.49��172.19.2.50��172.19.2.51�϶�ִ��

	source /root/local/bin/environment.sh
	wget https://github.com/coreos/etcd/releases/download/v3.1.6/etcd-v3.1.6-linux-amd64.tar.gz
	tar -xvf etcd-v3.1.6-linux-amd64.tar.gz
	cp etcd-v3.1.6-linux-amd64/etcd* /root/local/bin

	cat > etcd-csr.json <<EOF
	{
	  "CN": "etcd",
	  "hosts": [
		"127.0.0.1",
		"${NODE_IP}"
	  ],
	  "key": {
		"algo": "rsa",
		"size": 2048
	  },
	  "names": [
		{
		  "C": "CN",
		  "ST": "BeiJing",
		  "L": "BeiJing",
		  "O": "k8s",
		  "OU": "System"
		}
	  ]
	}
	EOF

	export PATH=/root/local/bin:$PATH
	source /root/local/bin/environment.sh

	cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
	  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
	  -config=/etc/kubernetes/ssl/ca-config.json \
	  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

	ls etcd*
	etcd.csr  etcd-csr.json  etcd-key.pem etcd.pem

	mkdir -p /etc/etcd/ssl
	mv etcd*.pem /etc/etcd/ssl
	rm etcd.csr  etcd-csr.json
	mkdir -p /var/lib/etcd

	cat > etcd.service <<EOF
	[Unit]
	Description=Etcd Server
	After=network.target
	After=network-online.target
	Wants=network-online.target
	Documentation=https://github.com/coreos

	[Service]
	Type=notify
	WorkingDirectory=/var/lib/etcd/
	ExecStart=/root/local/bin/etcd \\
	  --name=${NODE_NAME} \\
	  --cert-file=/etc/etcd/ssl/etcd.pem \\
	  --key-file=/etc/etcd/ssl/etcd-key.pem \\
	  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
	  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
	  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
	  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
	  --initial-advertise-peer-urls=https://${NODE_IP}:2380 \\
	  --listen-peer-urls=https://${NODE_IP}:2380 \\
	  --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \\
	  --advertise-client-urls=https://${NODE_IP}:2379 \\
	  --initial-cluster-token=etcd-cluster-0 \\
	  --initial-cluster=${ETCD_NODES} \\
	  --initial-cluster-state=new \\
	  --data-dir=/var/lib/etcd
	Restart=on-failure
	RestartSec=5
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
	EOF

	mv etcd.service /etc/systemd/system/
	systemctl daemon-reload
	systemctl enable etcd
	systemctl start etcd
	systemctl status etcd

��172.19.2.49����֤��Ⱥ

	for ip in ${NODE_IPS}; do
	  ETCDCTL_API=3 /root/local/bin/etcdctl \
	  --endpoints=https://${ip}:2379  \
	  --cacert=/etc/kubernetes/ssl/ca.pem \
	  --cert=/etc/etcd/ssl/etcd.pem \
	  --key=/etc/etcd/ssl/etcd-key.pem \
	  endpoint health; done


# ��������Kubectl�����й��� #
  
172.19.2.49��172.19.2.50��172.19.2.51�϶�ִ��

	vim /root/local/bin/environment.sh
	# �滻Ϊkubernetes��Ⱥmastr����IP
	export MASTER_IP=172.19.2.49
	export KUBE_APISERVER="https://${MASTER_IP}:6443"

	source /root/local/bin/environment.sh

172.19.2.49�϶�ִ��

	wget https://dl.k8s.io/v1.6.2/kubernetes-client-linux-amd64.tar.gz
	tar -xzvf kubernetes-client-linux-amd64.tar.gz
	cp kubernetes/client/bin/kube* /root/local/bin/
	chmod a+x /root/local/bin/kube*

	cat > admin-csr.json << EOF
	{
	  "CN": "admin",
	  "hosts": [],
	  "key": {
		"algo": "rsa",
		"size": 2048
	  },
	  "names": [
		{
		  "C": "CN",
		  "ST": "BeiJing",
		  "L": "BeiJing",
		  "O": "system:masters",
		  "OU": "System"
		}
	  ]
	}
	EOF

	cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
	  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
	  -config=/etc/kubernetes/ssl/ca-config.json \
	  -profile=kubernetes admin-csr.json | cfssljson -bare admin
	  
	ls admin*
	admin.csr  admin-csr.json  admin-key.pem  admin.pem

	mv admin*.pem /etc/kubernetes/ssl/
	rm admin.csr admin-csr.json

	# ���ü�Ⱥ����
	kubectl config set-cluster kubernetes \
	  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
	  --embed-certs=true \
	  --server=${KUBE_APISERVER}

	# ���ÿͻ�����֤����
	kubectl config set-credentials admin \
	  --client-certificate=/etc/kubernetes/ssl/admin.pem \
	  --embed-certs=true \
	  --client-key=/etc/kubernetes/ssl/admin-key.pem

	# ���������Ĳ���
	kubectl config set-context kubernetes \
	  --cluster=kubernetes \
	  --user=admin
	  
	# ����Ĭ��������
	kubectl config use-context kubernetes

	cat ~/.kube/config

172.19.2.50��172.19.2.51��ִ��

	scp kubernetes-client-linux-amd64.tar.gz app@172.19.2.50:/home/app
	scp kubernetes-client-linux-amd64.tar.gz app@172.19.2.51:/home/app
	mv /home/app/kubernetes-client-linux-amd64.tar.gz /home/lvqingshan
	chown root:root kubernetes-client-linux-amd64.tar.gz
	tar -xzvf kubernetes-client-linux-amd64.tar.gz
	cp kubernetes/client/bin/kube* /root/local/bin/
	chmod a+x /root/local/bin/kube*
	mkdir ~/.kube/

172.19.2.49�϶�ִ��

	scp ~/.kube/config root@172.19.2.50:/home/app
	scp ~/.kube/config root@172.19.2.51:/home/app

172.19.2.50��172.19.2.51�϶�ִ��

	mv /home/app/config ~/.kube/
	chown root:root ~/.kube/config


# �ġ�����Flannel���� #

172.19.2.49�϶�ִ��

	cat > flanneld-csr.json <<EOF
	{
	  "CN": "flanneld",
	  "hosts": [],
	  "key": {
		"algo": "rsa",
		"size": 2048
	  },
	  "names": [
		{
		  "C": "CN",
		  "ST": "BeiJing",
		  "L": "BeiJing",
		  "O": "k8s",
		  "OU": "System"
		}
	  ]
	}
	EOF

	cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
	  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
	  -config=/etc/kubernetes/ssl/ca-config.json \
	  -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
	  
	ls flanneld*
	flanneld.csr  flanneld-csr.json  flanneld-key.pem  flanneld.pem

	scp flanneld* root@172.19.2.50:/home/app/
	scp flanneld* root@172.19.2.51:/home/app/

172.19.2.50��172.19.2.51�϶�ִ��

	mv /home/app/flanneld* .
	mv /home/app/flanneld* .


172.19.2.49��172.19.2.50��172.19.2.51�϶�ִ��

	mkdir -p /etc/flanneld/ssl
	mv flanneld*.pem /etc/flanneld/ssl
	rm flanneld.csr  flanneld-csr.json

172.19.2.49��ִ��һ�Σ�ֻ��master��ִ��һ�Σ������ڵ㲻ִ�У�

	/root/local/bin/etcdctl \
	  --endpoints=${ETCD_ENDPOINTS} \
	  --ca-file=/etc/kubernetes/ssl/ca.pem \
	  --cert-file=/etc/flanneld/ssl/flanneld.pem \
	  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
	  set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"'${CLUSTER_CIDR}'", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'



	mkdir flannel
	wget https://github.com/coreos/flannel/releases/download/v0.7.1/flannel-v0.7.1-linux-amd64.tar.gz
	tar -xzvf flannel-v0.7.1-linux-amd64.tar.gz -C flannel
	cp flannel/{flanneld,mk-docker-opts.sh} /root/local/bin


	cat > flanneld.service << EOF
	[Unit]
	Description=Flanneld overlay address etcd agent
	After=network.target
	After=network-online.target
	Wants=network-online.target
	After=etcd.service
	Before=docker.service

	[Service]
	Type=notify
	ExecStart=/root/local/bin/flanneld \\
	  -etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
	  -etcd-certfile=/etc/flanneld/ssl/flanneld.pem \\
	  -etcd-keyfile=/etc/flanneld/ssl/flanneld-key.pem \\
	  -etcd-endpoints=${ETCD_ENDPOINTS} \\
	  -etcd-prefix=${FLANNEL_ETCD_PREFIX}
	ExecStartPost=/root/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
	Restart=on-failure

	[Install]
	WantedBy=multi-user.target
	RequiredBy=docker.service
	EOF

	cp flanneld.service /etc/systemd/system/
	systemctl daemon-reload
	systemctl enable flanneld
	systemctl start flanneld
	systemctl status flanneld
	journalctl  -u flanneld |grep 'Lease acquired'
	ifconfig flannel.1

	# �鿴��Ⱥ Pod ����(/16)
	/root/local/bin/etcdctl \
	   --endpoints=${ETCD_ENDPOINTS} \
	   --ca-file=/etc/kubernetes/ssl/ca.pem \
	   --cert-file=/etc/flanneld/ssl/flanneld.pem \
	   --key-file=/etc/flanneld/ssl/flanneld-key.pem \
	   get ${FLANNEL_ETCD_PREFIX}/config
	�������
	{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}

	# �鿴�ѷ���� Pod �������б�(/24)
	/root/local/bin/etcdctl \
	   --endpoints=${ETCD_ENDPOINTS} \
	   --ca-file=/etc/kubernetes/ssl/ca.pem \
	   --cert-file=/etc/flanneld/ssl/flanneld.pem \
	   --key-file=/etc/flanneld/ssl/flanneld-key.pem \
	   ls ${FLANNEL_ETCD_PREFIX}/subnets
	�������
	/kubernetes/network/subnets/172.30.27.0-24

	# �鿴ĳһ Pod ���ζ�Ӧ�� flanneld ���̼����� IP ���������
	/root/local/bin/etcdctl \
	   --endpoints=${ETCD_ENDPOINTS} \
	   --ca-file=/etc/kubernetes/ssl/ca.pem \
	   --cert-file=/etc/flanneld/ssl/flanneld.pem \
	   --key-file=/etc/flanneld/ssl/flanneld-key.pem \
	   get ${FLANNEL_ETCD_PREFIX}/subnets/172.30.27.0-24
	�������
	{"PublicIP":"172.19.2.49","BackendType":"vxlan","BackendData":{"VtepMAC":"9a:7b:7e:6a:2e:0b"}}

172.19.2.49�϶�ִ��

	/root/local/bin/etcdctl \
	  --endpoints=${ETCD_ENDPOINTS} \
	  --ca-file=/etc/kubernetes/ssl/ca.pem \
	  --cert-file=/etc/flanneld/ssl/flanneld.pem \
	  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
	  ls ${FLANNEL_ETCD_PREFIX}/subnets

�������

	/kubernetes/network/subnets/172.30.27.0-24
	/kubernetes/network/subnets/172.30.22.0-24
	/kubernetes/network/subnets/172.30.38.0-24

�ֱ�ping���µ�ַ��ע���Լ�ping�Լ�ping��ͨ

	172.30.27.1
	172.30.22.1
	172.30.38.1


# �塢����master�ڵ� #

kubernetes master �ڵ�����������
kube-apiserver
kube-scheduler
kube-controller-manager

172.19.2.49��ִ��

	wget https://github.com/kubernetes/kubernetes/releases/download/v1.6.2/kubernetes.tar.gz
	tar -xzvf kubernetes.tar.gz
	cd kubernetes
	./cluster/get-kube-binaries.sh
	wget https://dl.k8s.io/v1.6.2/kubernetes-server-linux-amd64.tar.gz
	tar -xzvf kubernetes-server-linux-amd64.tar.gz
	cd kubernetes
	tar -xzvf  kubernetes-src.tar.gz
	cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /root/local/bin/
	cd ../..

	cat > kubernetes-csr.json <<EOF
	{
	  "CN": "kubernetes",
	  "hosts": [
		"127.0.0.1",
		"${MASTER_IP}",
		"${CLUSTER_KUBERNETES_SVC_IP}",
		"kubernetes",
		"kubernetes.default",
		"kubernetes.default.svc",
		"kubernetes.default.svc.cluster",
		"kubernetes.default.svc.cluster.local"
	  ],
	  "key": {
		"algo": "rsa",
		"size": 2048
	  },
	  "names": [
		{
		  "C": "CN",
		  "ST": "BeiJing",
		  "L": "BeiJing",
		  "O": "k8s",
		  "OU": "System"
		}
	  ]
	}
	EOF

	cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
	  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
	  -config=/etc/kubernetes/ssl/ca-config.json \
	  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

	ls kubernetes*
	kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
	mkdir -p /etc/kubernetes/ssl/
	mv kubernetes*.pem /etc/kubernetes/ssl/
	rm kubernetes.csr  kubernetes-csr.json

	cat > token.csv <<EOF
	${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
	EOF

	mv token.csv /etc/kubernetes/

	cat  > kube-apiserver.service <<EOF
	[Unit]
	Description=Kubernetes API Server
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=network.target

	[Service]
	ExecStart=/root/local/bin/kube-apiserver \\
	  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
	  --advertise-address=${MASTER_IP} \\
	  --bind-address=${MASTER_IP} \\
	  --insecure-bind-address=${MASTER_IP} \\
	  --authorization-mode=RBAC \\
	  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \\
	  --kubelet-https=true \\
	  --experimental-bootstrap-token-auth \\
	  --token-auth-file=/etc/kubernetes/token.csv \\
	  --service-cluster-ip-range=${SERVICE_CIDR} \\
	  --service-node-port-range=${NODE_PORT_RANGE} \\
	  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \\
	  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
	  --client-ca-file=/etc/kubernetes/ssl/ca.pem \\
	  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
	  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
	  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \\
	  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \\
	  --etcd-servers=${ETCD_ENDPOINTS} \\
	  --enable-swagger-ui=true \\
	  --allow-privileged=true \\
	  --apiserver-count=3 \\
	  --audit-log-maxage=30 \\
	  --audit-log-maxbackup=3 \\
	  --audit-log-maxsize=100 \\
	  --audit-log-path=/var/lib/audit.log \\
	  --event-ttl=1h \\
	  --v=2
	Restart=on-failure
	RestartSec=5
	Type=notify
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
	EOF

	cp kube-apiserver.service /etc/systemd/system/
	systemctl daemon-reload
	systemctl enable kube-apiserver
	systemctl start kube-apiserver
	systemctl status kube-apiserver


	cat > kube-controller-manager.service <<EOF
	[Unit]
	Description=Kubernetes Controller Manager
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes

	[Service]
	ExecStart=/root/local/bin/kube-controller-manager \\
	  --address=127.0.0.1 \\
	  --master=http://${MASTER_IP}:8080 \\
	  --allocate-node-cidrs=true \\
	  --service-cluster-ip-range=${SERVICE_CIDR} \\
	  --cluster-cidr=${CLUSTER_CIDR} \\
	  --cluster-name=kubernetes \\
	  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \\
	  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \\
	  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \\
	  --root-ca-file=/etc/kubernetes/ssl/ca.pem \\
	  --leader-elect=true \\
	  --v=2
	Restart=on-failure
	RestartSec=5

	[Install]
	WantedBy=multi-user.target
	EOF



	cp kube-controller-manager.service /etc/systemd/system/
	systemctl daemon-reload
	systemctl enable kube-controller-manager
	systemctl start kube-controller-manager
	systemctl status kube-controller-manager




	cat > kube-scheduler.service <<EOF
	[Unit]
	Description=Kubernetes Scheduler
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes

	[Service]
	ExecStart=/root/local/bin/kube-scheduler \\
	  --address=127.0.0.1 \\
	  --master=http://${MASTER_IP}:8080 \\
	  --leader-elect=true \\
	  --v=2
	Restart=on-failure
	RestartSec=5

	[Install]
	WantedBy=multi-user.target
	EOF

	cp kube-scheduler.service /etc/systemd/system/
	systemctl daemon-reload
	systemctl enable kube-scheduler
	systemctl start kube-scheduler
	systemctl status kube-scheduler

��֤�ڵ㽡��״��

	kubectl get componentstatuses


# ��������Node�ڵ� #

kubernetes Node �ڵ�������������
flanneld
docker
kubelet
kube-proxy

172.19.2.49��172.19.2.50��172.19.2.51�϶�ִ��

	#��װdocker
	yum install docker-ce
	rpm -qa | grep docker
	docker-ce-selinux-17.03.1.ce-1.el7.centos.noarch
	docker-ce-17.03.1.ce-1.el7.centos.x86_64

	#�޸�docker�����ļ����������ļ��м���һ��
	vim /etc/systemd/system/docker.service
	[Service]
	Type=notify
	Environment=GOTRACEBACK=crash
	EnvironmentFile=-/run/flannel/docker

	systemctl daemon-reload
	systemctl enable docker
	systemctl start docker
	docker version
	kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
	wget https://dl.k8s.io/v1.6.2/kubernetes-server-linux-amd64.tar.gz
	tar -xzvf kubernetes-server-linux-amd64.tar.gz
	cd kubernetes
	tar -xzvf  kubernetes-src.tar.gz
	cp -r ./server/bin/{kube-proxy,kubelet} /root/local/bin/

	# ���ü�Ⱥ����
	kubectl config set-cluster kubernetes \
	  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
	  --embed-certs=true \
	  --server=${KUBE_APISERVER} \
	  --kubeconfig=bootstrap.kubeconfig
	# ���ÿͻ�����֤����
	kubectl config set-credentials kubelet-bootstrap \
	  --token=${BOOTSTRAP_TOKEN} \
	  --kubeconfig=bootstrap.kubeconfig
	# ���������Ĳ���
	kubectl config set-context default \
	  --cluster=kubernetes \
	  --user=kubelet-bootstrap \
	  --kubeconfig=bootstrap.kubeconfig
	# ����Ĭ��������
	kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
	mv bootstrap.kubeconfig /etc/kubernetes/

	mkdir /var/lib/kubelet
	cat > kubelet.service <<EOF
	[Unit]
	Description=Kubernetes Kubelet
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=docker.service
	Requires=docker.service

	[Service]
	WorkingDirectory=/var/lib/kubelet
	ExecStart=/root/local/bin/kubelet \\
	  --address=${NODE_IP} \\
	  --hostname-override=${NODE_IP} \\
	  --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest \\
	  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
	  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
	  --require-kubeconfig \\
	  --cert-dir=/etc/kubernetes/ssl \\
	  --cluster_dns=${CLUSTER_DNS_SVC_IP} \\
	  --cluster_domain=${CLUSTER_DNS_DOMAIN} \\
	  --hairpin-mode promiscuous-bridge \\
	  --allow-privileged=true \\
	  --serialize-image-pulls=false \\
	  --logtostderr=true \\
	  --v=2
	Restart=on-failure
	RestartSec=5

	[Install]
	WantedBy=multi-user.target
	EOF

	cp kubelet.service /etc/systemd/system/kubelet.service
	systemctl daemon-reload
	systemctl enable kubelet
	systemctl start kubelet
	systemctl status kubelet
	journalctl -e -u kubelet

172.19.2.49��ִ��

	kubectl get csr
	#ע��approve��Ϊkubelet�ڵ��NAME���˴�Ϊͨ���ڵ����֤
	kubectl certificate approve csr-1w6sj
	kubectl get csr
	kubectl get nodes

	ls -l /etc/kubernetes/kubelet.kubeconfig
	ls -l /etc/kubernetes/ssl/kubelet*

	cat > kube-proxy-csr.json << EOF
	{
	  "CN": "system:kube-proxy",
	  "hosts": [],
	  "key": {
		"algo": "rsa",
		"size": 2048
	  },
	  "names": [
		{
		  "C": "CN",
		  "ST": "BeiJing",
		  "L": "BeiJing",
		  "O": "k8s",
		  "OU": "System"
		}
	  ]
	}
	EOF

	cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
	  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
	  -config=/etc/kubernetes/ssl/ca-config.json \
	  -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy

	ls kube-proxy*
	cp kube-proxy*.pem /etc/kubernetes/ssl/
	rm kube-proxy.csr  kube-proxy-csr.json

	scp kube-proxy*.pem root@172.19.2.51:/home/lvqingshan
	scp kube-proxy*.pem root@172.19.2.51:/home/lvqingshan

172.19.2.50��172.19.2.51�϶�ִ��

	mv /home/lvqingshan/kube-proxy*.pem /etc/kubernetes/ssl/

172.19.2.49��172.19.2.50��172.19.2.51�϶�ִ��

	# ���ü�Ⱥ����
	kubectl config set-cluster kubernetes \
	  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
	  --embed-certs=true \
	  --server=${KUBE_APISERVER} \
	  --kubeconfig=kube-proxy.kubeconfig
	# ���ÿͻ�����֤����
	kubectl config set-credentials kube-proxy \
	  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
	  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
	  --embed-certs=true \
	  --kubeconfig=kube-proxy.kubeconfig
	# ���������Ĳ���
	kubectl config set-context default \
	  --cluster=kubernetes \
	  --user=kube-proxy \
	  --kubeconfig=kube-proxy.kubeconfig
	# ����Ĭ��������
	kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
	mv kube-proxy.kubeconfig /etc/kubernetes/

	mkdir -p /var/lib/kube-proxy

	cat > kube-proxy.service <<EOF
	[Unit]
	Description=Kubernetes Kube-Proxy Server
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=network.target

	[Service]
	WorkingDirectory=/var/lib/kube-proxy
	ExecStart=/root/local/bin/kube-proxy \\
	  --bind-address=${NODE_IP} \\
	  --hostname-override=${NODE_IP} \\
	  --cluster-cidr=${SERVICE_CIDR} \\
	  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
	  --logtostderr=true \\
	  --v=2
	Restart=on-failure
	RestartSec=5
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
	EOF

	cp kube-proxy.service /etc/systemd/system/
	systemctl daemon-reload
	systemctl enable kube-proxy
	systemctl start kube-proxy
	systemctl status kube-proxy

172.19.2.49��ִ��
	cat > nginx-ds.yml << EOF
	apiVersion: v1
	kind: Service
	metadata:
	  name: nginx-ds
	  labels:
		app: nginx-ds
	spec:
	  type: NodePort
	  selector:
		app: nginx-ds
	  ports:
	  - name: http
		port: 80
		targetPort: 80

	---

	apiVersion: extensions/v1beta1
	kind: DaemonSet
	metadata:
	  name: nginx-ds
	  labels:
		addonmanager.kubernetes.io/mode: Reconcile
	spec:
	  template:
		metadata:
		  labels:
			app: nginx-ds
		spec:
		  containers:
		  - name: my-nginx
			image: nginx:1.7.9
			ports:
			- containerPort: 80
	EOF

	kubectl create -f nginx-ds.yml
	kubectl get nodes

	kubectl get pods  -o wide|grep nginx-ds
	nginx-ds-4cbd3   1/1       Running   0          1m        172.17.0.2   172.19.2.51
	nginx-ds-f0217   1/1       Running   0          1m        172.17.0.2   172.19.2.50

	kubectl get svc |grep nginx-ds
	nginx-ds     10.254.194.173   <nodes>       80:8542/TCP   1m

	curl 10.254.194.173


# �ߡ�����DNS��� #

172.19.2.49��ִ��

	mkdir -pv /home/lvqingshan/dns
	cd /home/lvqingshan/dns

	wget https://github.com/opsnull/follow-me-install-kubernetes-cluster/raw/master/manifests/kubedns/kubedns-cm.yaml
	wget https://github.com/opsnull/follow-me-install-kubernetes-cluster/raw/master/manifests/kubedns/kubedns-controller.yaml
	wget https://github.com/opsnull/follow-me-install-kubernetes-cluster/raw/master/manifests/kubedns/kubedns-sa.yaml
	wget https://github.com/opsnull/follow-me-install-kubernetes-cluster/raw/master/manifests/kubedns/kubedns-svc.yaml

	kubectl get clusterrolebindings system:kube-dns -o yaml
	apiVersion: rbac.authorization.k8s.io/v1beta1
	kind: ClusterRoleBinding
	metadata:
	  annotations:
		rbac.authorization.kubernetes.io/autoupdate: "true"
	  creationTimestamp: 2017-05-23T07:18:07Z
	  labels:
		kubernetes.io/bootstrapping: rbac-defaults
	  name: system:kube-dns
	  resourceVersion: "56"
	  selfLink: /apis/rbac.authorization.k8s.io/v1beta1/clusterrolebindingssystem%3Akube-dns
	  uid: f8284130-3f87-11e7-964f-005056bfceaa
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: system:kube-dns
	subjects:
	- kind: ServiceAccount
	  name: kube-dns
	  namespace: kube-system

	pwd
	/home/lvqingshan/dns

	ls *
	kubedns-cm.yaml  kubedns-controller.yaml  kubedns-sa.yaml  kubedns-svc.yaml

	kubectl create -f .

	cat > my-nginx.yaml << EOF
	apiVersion: extensions/v1beta1
	kind: Deployment
	metadata:
	  name: my-nginx
	spec:
	  replicas: 2
	  template:
		metadata:
		  labels:
			run: my-nginx
		spec:
		  containers:
		  - name: my-nginx
			image: nginx:1.7.9
			ports:
			- containerPort: 80
	EOF


	kubectl create -f my-nginx.yaml
	kubectl expose deploy my-nginx
	kubectl get services --all-namespaces |grep my-nginx
	default       my-nginx     10.254.57.162   <none>        80/TCP          14s

	cat > pod-nginx.yaml << EOF
	apiVersion: v1
	kind: Pod
	metadata:
	  name: nginx
	spec:
	  containers:
	  - name: nginx
		image: nginx:1.7.9
		ports:
		- containerPort: 80
	EOF

	kubectl create -f pod-nginx.yaml
	kubectl exec  nginx -i -t -- /bin/bash
	cat /etc/resolv.conf
	exit


# �ˡ�����Dashboard��� #

172.19.2.49��ִ��

	mkdir dashboard
	cd dashboard
	wget https://github.com/opsnull/follow-me-install-kubernetes-cluster/raw/master/manifests/dashboard/dashboard-controller.yaml
	wget https://github.com/opsnull/follow-me-install-kubernetes-cluster/raw/master/manifests/dashboard/dashboard-rbac.yaml
	wget https://github.com/opsnull/follow-me-install-kubernetes-cluster/raw/master/manifests/dashboard/dashboard-service.yaml

	pwd
	/home/lvqingshan/dashboard
	ls *.yaml
	dashboard-controller.yaml  dashboard-rbac.yaml  dashboard-service.yaml

	#����apiserver��ַ
	vim dashboard-controller.yaml
			ports:
			- containerPort: 9090
			livenessProbe:
			  httpGet:
				path: /
				port: 9090
			  initialDelaySeconds: 30
			  timeoutSeconds: 30
			args:
			- --apiserver-host=http://172.19.2.49:8080
	#����nodePort�˿�
	vim dashboard-service.yaml
	spec:
	  type: NodePort
	  selector:
		k8s-app: kubernetes-dashboard
	  ports:
	  - port: 80
		targetPort: 9090
		nodePort: 8484

	kubectl create -f .

	kubectl get services kubernetes-dashboard -n kube-system
	NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)       AGE
	kubernetes-dashboard   10.254.86.190   <nodes>       80:8861/TCP   21s

	kubectl get deployment kubernetes-dashboard  -n kube-system
	NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
	kubernetes-dashboard   1         1         1            1           57s

	kubectl get pods  -n kube-system | grep dashboard
	kubernetes-dashboard-3677875397-8lhp3   1/1       Running   0          1m

	#ͨ�� kubectl proxy ���� dashboard
	kubectl proxy --address='172.19.2.49' --port=8086 --accept-hosts='^*$'

	kubectl cluster-info
	Kubernetes master is running at https://172.19.2.49:6443
	KubeDNS is running at https://172.19.2.49:6443/api/v1/proxy/namespaces/kube-system/services/kube-dns
	kubernetes-dashboard is running at https://172.19.2.49:6443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard

	#ͨ�� kube-apiserver ����dashboard
	����������������µ�ַ����:
	http://172.19.2.49:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard


# �š�����harbor˽�вֿ� #

172.19.2.49��ִ��

	source /root/local/bin/environment.sh

	cd /root/harbor
	cat > harbor-csr.json <<EOF
	{
	  "CN": "harbor",
	  "hosts": [
		"$NODE_IP"
	  ],
	  "key": {
		"algo": "rsa",
		"size": 2048
	  },
	  "names": [
		{
		  "C": "CN",
		  "ST": "BeiJing",
		  "L": "BeiJing",
		  "O": "k8s",
		  "OU": "System"
		}
	  ]
	}
	EOF

	cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
	  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
	  -config=/etc/kubernetes/ssl/ca-config.json \
	  -profile=kubernetes harbor-csr.json | cfssljson -bare harbor

	ls harbor*
	harbor.csr  harbor-csr.json  harbor-key.pem harbor.pem

	mkdir -p /etc/harbor/ssl
	mv harbor*.pem /etc/harbor/ssl
	rm harbor.csr  harbor-csr.json

	export PATH=/usr/local/bin/:$PATH
	mkdir -p /etc/docker/certs.d/172.19.2.49
	cp /etc/kubernetes/ssl/ca.pem /etc/docker/certs.d/172.19.2.49/ca.crt

	vim /etc/sysconfig/docker
	OPTIONS='--selinux-enabled --insecure-registry=172.19.2.49'

	yum install ca-certificates

	vim /root/harbor/harbor.cfg
	hostname = 172.19.2.49
	ui_url_protocol = https
	ssl_cert = /etc/harbor/ssl/harbor.pem
	ssl_cert_key = /etc/harbor/ssl/harbor-key.pem
	#��¼harbor���˺�����
	harbor_admin_password = Harbor123456

	./install.sh

	#ֹͣ������
	docker-compose down -v
	docker-compose up -d

	#��¼172.19.2.49
	�˺ţ�admin
	���룺Harbor123456


	#�������¼�ֿ�url
	https://172.19.2.49/

	#�ϴ����ؾ���˽�вֿ�
	docker login 172.19.2.49
	docker tag quay.io/pires/docker-elasticsearch-kubernetes:5.4.0 172.19.2.49/library/docker-elasticsearch-kubernetes:5.4
	docker push 172.19.2.49/library/docker-elasticsearch-kubernetes:5.4

172.19.2.50��172.19.2.51�϶�ִ��

	mkdir -pv /etc/docker/certs.d/172.19.2.49/

	vim /etc/sysconfig/docker
	OPTIONS='--selinux-enabled --insecure-registry=172.19.2.49'

	yum install ca-certificates

172.19.2.49�ϴ�֤��

	scp /etc/docker/certs.d/172.19.2.49/ca.crt root@172.19.2.50:/etc/docker/certs.d/172.19.2.49/
	scp /etc/docker/certs.d/172.19.2.49/ca.crt root@172.19.2.51:/etc/docker/certs.d/172.19.2.49/

172.19.2.50��172.19.2.51��¼harbor�ֿ�
	docker login 172.19.2.49


# �š�С���� #

��Ϊ����calico�ķ�ʽ�������Ⱥ���磬���ɾ����װcalico������tunl0����

	modprobe -r ipip
