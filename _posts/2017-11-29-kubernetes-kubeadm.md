---
date: 2017-11-29 02:25:24+08:00
layout: post
title: 使用kubeadm安装kubernetes
categories: linux
tags: docker harbor kubernetes k8s kubeadm
---


# 简介： #

kubernetes集群结构和安装环境

	主节点：
	172.19.2.50
	主节点包含组件：kubectl、Kube-Proxy、kube-dns、etcd、kube-apiserver、kube-controller-manager、kube-scheduler、calico-node、calico-policy-controller、calico-etcd

	从节点：
	172.19.2.51
	172.19.2.140
	从节点包含组件：kubernetes-dashboard、calico-node、kube-proxy

	最后kubernetes-dashboard访问地址：
	http://172.19.2.50:30099/

	kubectl:用于操作kubernetes集群的命令行接口
	Kube-Proxy:用于暴露容器内的端口,kubernetes master会从与定义的端口范围内请求一个端口号（默认范围：30000-32767）
	kube-dns：用于提供容器内的DNS服务
	etcd：用于共享配置和服务发现的分布式，一致性的KV存储系统
	kube-apiserver：相当于是k8s集群的一个入口,不论通过kubectl还是使用remote api直接控制,都要经过apiserver
	kube-controller-manager：承担了master的主要功能,比如交互,管理node,pod,replication,service,namespace等
	kube-scheduler：根据特定的调度算法将pod调度到指定的工作节点（minion）上，这一过程也叫绑定（bind）
	calico：基于BGP协议的虚拟网络工具，这里主要用于容器内网络通信
	kubernetes-dashboard：官方提供的用户管理Kubernets集群可视化工具

参考文档：

https://mritd.me/2016/10/29/set-up-kubernetes-cluster-by-kubeadm/

http://www.cnblogs.com/liangDream/p/7358847.html


# 一、所有节点安装kubeadm #

清理系统中残留的kubernetes文件

	rm -r -f /etc/kubernetes /var/lib/kubelet /var/lib/etcd
	rm -rf $HOME/.kube

部署阿里云的kubernetes仓库

	vim /etc/yum.repos.d/kubernetes.repo
	[kube]
	name=Kube
	baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
	enabled=1
	gpgcheck=0

安装kubeadm

	yum install kubeadm

安装完后必须有以下几个组件

	rpm -qa | grep kube
	kubernetes-cni-0.5.1-0.x86_64
	kubelet-1.7.5-0.x86_64
	kubectl-1.7.5-0.x86_64
	kubeadm-1.7.5-0.x86_64

docker使用的是docker-ce

新增docker-ce仓库

	vim /etc/yum.repos.d/docker-ce.repo
	[docker-ce-stable]
	name=Docker CE Stable - $basearch
	baseurl=https://download.docker.com/linux/centos/7/$basearch/stable
	enabled=1
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-stable-debuginfo]
	name=Docker CE Stable - Debuginfo $basearch
	baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/stable
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-stable-source]
	name=Docker CE Stable - Sources
	baseurl=https://download.docker.com/linux/centos/7/source/stable
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-edge]
	name=Docker CE Edge - $basearch
	baseurl=https://download.docker.com/linux/centos/7/$basearch/edge
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-edge-debuginfo]
	name=Docker CE Edge - Debuginfo $basearch
	baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/edge
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-edge-source]
	name=Docker CE Edge - Sources
	baseurl=https://download.docker.com/linux/centos/7/source/edge
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-test]
	name=Docker CE Test - $basearch
	baseurl=https://download.docker.com/linux/centos/7/$basearch/test
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-test-debuginfo]
	name=Docker CE Test - Debuginfo $basearch
	baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/test
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

	[docker-ce-test-source]
	name=Docker CE Test - Sources
	baseurl=https://download.docker.com/linux/centos/7/source/test
	enabled=0
	gpgcheck=1
	gpgkey=https://download.docker.com/linux/centos/gpg

安装docker-ce

	yum install docker-ce


# 二、部署镜像到所有节点 #

所需镜像，因为是谷歌镜像需要翻墙，可以先把镜像构建到dockerhub上再下载到本地仓库

	gcr.io/google_containers/etcd-amd64:3.0.17
	gcr.io/google_containers/kube-apiserver-amd64:v1.7.6
	gcr.io/google_containers/kube-controller-manager-amd64:v1.7.6
	gcr.io/google_containers/kube-scheduler-amd64:v1.7.6
	quay.io/coreos/etcd:v3.1.10
	quay.io/calico/node:v2.4.1
	quay.io/calico/cni:v1.10.0
	quay.io/calico/kube-policy-controller:v0.7.0
	gcr.io/google_containers/pause-amd64:3.0
	gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.4
	gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.4
	gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.3
	gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.4
	gcr.io/google_containers/kube-proxy-amd64:v1.7.6

构建镜像过程参考

https://mritd.me/2016/10/29/set-up-kubernetes-cluster-by-kubeadm/

http://www.cnblogs.com/liangDream/p/7358847.html


复制已有的私有仓库密钥到本地

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

登出docker仓库，登录私有仓库

	docker logout
	docker login 172.19.2.139
	Username: admin
	Password: Cmcc@1ot

下载私有仓库中的镜像

	docker pull 172.19.2.139/xsllqs/etcd-amd64:3.0.17
	docker pull 172.19.2.139/xsllqs/kube-scheduler-amd64:v1.7.6
	docker pull 172.19.2.139/xsllqs/kube-apiserver-amd64:v1.7.6
	docker pull 172.19.2.139/xsllqs/kube-controller-manager-amd64:v1.7.6
	docker pull 172.19.2.139/xsllqs/etcd:v3.1.10
	docker pull 172.19.2.139/xsllqs/node:v2.4.1
	docker pull 172.19.2.139/xsllqs/cni:v1.10.0
	docker pull 172.19.2.139/xsllqs/kube-policy-controller:v0.7.0
	docker pull 172.19.2.139/xsllqs/pause-amd64:3.0
	docker pull 172.19.2.139/xsllqs/k8s-dns-kube-dns-amd64:1.14.4
	docker pull 172.19.2.139/xsllqs/k8s-dns-dnsmasq-nanny-amd64:1.14.4
	docker pull 172.19.2.139/xsllqs/kubernetes-dashboard-amd64:v1.6.3
	docker pull 172.19.2.139/xsllqs/k8s-dns-sidecar-amd64:1.14.4
	docker pull 172.19.2.139/xsllqs/kube-proxy-amd64:v1.7.6

重命名镜像

	docker tag 172.19.2.139/xsllqs/etcd-amd64:3.0.17 gcr.io/google_containers/etcd-amd64:3.0.17
	docker tag 172.19.2.139/xsllqs/kube-scheduler-amd64:v1.7.6 gcr.io/google_containers/kube-scheduler-amd64:v1.7.6
	docker tag 172.19.2.139/xsllqs/kube-apiserver-amd64:v1.7.6 gcr.io/google_containers/kube-apiserver-amd64:v1.7.6
	docker tag 172.19.2.139/xsllqs/kube-controller-manager-amd64:v1.7.6 gcr.io/google_containers/kube-controller-manager-amd64:v1.7.6
	docker tag 172.19.2.139/xsllqs/etcd:v3.1.10 quay.io/coreos/etcd:v3.1.10
	docker tag 172.19.2.139/xsllqs/node:v2.4.1 quay.io/calico/node:v2.4.1
	docker tag 172.19.2.139/xsllqs/cni:v1.10.0 quay.io/calico/cni:v1.10.0
	docker tag 172.19.2.139/xsllqs/kube-policy-controller:v0.7.0 quay.io/calico/kube-policy-controller:v0.7.0
	docker tag 172.19.2.139/xsllqs/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0
	docker tag 172.19.2.139/xsllqs/k8s-dns-kube-dns-amd64:1.14.4 gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.4
	docker tag 172.19.2.139/xsllqs/k8s-dns-dnsmasq-nanny-amd64:1.14.4 gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.4
	docker tag 172.19.2.139/xsllqs/kubernetes-dashboard-amd64:v1.6.3 gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.3
	docker tag 172.19.2.139/xsllqs/k8s-dns-sidecar-amd64:1.14.4 gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.4
	docker tag 172.19.2.139/xsllqs/kube-proxy-amd64:v1.7.6 gcr.io/google_containers/kube-proxy-amd64:v1.7.6


# 三、所有节点启动kubelet #

在hosts中加入所有主机名

直接启动kubelet会有这个报错

	journalctl -xe
	error: failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: "systemd" is different from docker cgroup driver: "cgrou

所以需要修改以下内容

	vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
	Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"

以下有就修改，没有则不管

	vim /etc/systemd/system/kubelet.service.d/99-kubelet-droplet.conf
	Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=registry.aliyuncs.com/archon/pause-amd64:3.0 --cgroup-driver=cgroupfs"

启动

	systemctl enable kubelet
	systemctl start kubelet

也许会启动失败，说找不到配置文件，但是不用管，因为kubeadm会给出配置文件并启动kubelet，建议执行了kubeadm init失败后启动kubelete


# 四、kubeadm部署master节点 #

master节点执行以下命令

	kubeadm reset
	echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
	kubeadm init --kubernetes-version=v1.7.6

会有以下内容出现

	[apiclient] All control plane components are healthy after 30.001912 seconds
	[token] Using token: cab485.49b7c0358a06ad35
	[apiconfig] Created RBAC rules
	[addons] Applied essential addon: kube-proxy
	[addons] Applied essential addon: kube-dns

	Your Kubernetes master has initialized successfully!

	To start using your cluster, you need to run (as a regular user):

	  mkdir -p $HOME/.kube
	  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	  sudo chown $(id -u):$(id -g) $HOME/.kube/config

	You should now deploy a pod network to the cluster.
	Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
	  http://kubernetes.io/docs/admin/addons/

	You can now join any number of machines by running the following on each node
	as root:
	
	  kubeadm join --token cab485.49b7c0358a06ad35 172.19.2.50:6443

请牢记以下内容，后期新增节点需要用到

	kubeadm join --token cab485.49b7c0358a06ad35 172.19.2.50:6443


按照要求执行

	mkdir -p $HOME/.kube
	cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	chown $(id -u):$(id -g) $HOME/.kube/config

部署calico网络，最好先部署kubernetes-dashboard再部署calico网络

	cd /home/lvqingshan/
	kubectl apply -f http://docs.projectcalico.org/v2.4/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml

查看名称空间

	kubectl get pods --all-namespaces


# 五、主节点安装kubernetes-dashboard #

下载kubernetes-dashboard对应的yaml文件

	cd /home/lvqingshan/
	wget https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml -O kubernetes-dashboard.yaml

修改该yaml文件，固定服务对外端口

	vim kubernetes-dashboard.yaml
	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  labels:
		k8s-app: kubernetes-dashboard
	  name: kubernetes-dashboard
	  namespace: kube-system
	---
	apiVersion: rbac.authorization.k8s.io/v1beta1
	kind: ClusterRoleBinding
	metadata:
	  name: kubernetes-dashboard
	  labels:
		k8s-app: kubernetes-dashboard
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: cluster-admin
	subjects:
	- kind: ServiceAccount
	  name: kubernetes-dashboard
	  namespace: kube-system
	---
	kind: Deployment
	apiVersion: extensions/v1beta1
	metadata:
	  labels:
		k8s-app: kubernetes-dashboard
	  name: kubernetes-dashboard
	  namespace: kube-system
	spec:
	  replicas: 1
	  revisionHistoryLimit: 10
	  selector:
		matchLabels:
		  k8s-app: kubernetes-dashboard
	  template:
		metadata:
		  labels:
			k8s-app: kubernetes-dashboard
		spec:
		  containers:
		  - name: kubernetes-dashboard
			image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.3
			ports:
			- containerPort: 9090
			  protocol: TCP
			args:
			  # Uncomment the following line to manually specify Kubernetes API server Host
			  # If not specified, Dashboard will attempt to auto discover the API server and connect
			  # to it. Uncomment only if the default does not work.
			  # - --apiserver-host=http://my-address:port
			livenessProbe:
			  httpGet:
				path: /
				port: 9090
			  initialDelaySeconds: 30
			  timeoutSeconds: 30
		  serviceAccountName: kubernetes-dashboard
		  # Comment the following tolerations if Dashboard must not be deployed on master
		  tolerations:
		  - key: node-role.kubernetes.io/master
			effect: NoSchedule
	---
	kind: Service
	apiVersion: v1
	metadata:
	  labels:
		k8s-app: kubernetes-dashboard
	  name: kubernetes-dashboard
	  namespace: kube-system
	spec:
	  type: NodePort
	  ports:
	  - port: 80
		targetPort: 9090
		nodePort: 30099		#新增这里，指定物理机上提供的端口
	  selector:
		k8s-app: kubernetes-dashboard
	  type: NodePort	#新增这里，指定类型

创建dashboard

	kubectl create -f kubernetes-dashboard.yaml


# 六、从节点加入集群 #

从节点修改/etc/systemd/system/kubelet.service.d/文件后执行

	systemctl enable kubelet
	systemctl start kubelet

	kubeadm join --token cab485.49b7c0358a06ad35 172.19.2.50:6443

查看dashboard的NodePort

	kubectl describe svc kubernetes-dashboard --namespace=kube-system

网页打开

http://172.19.2.50:30099/