---
date: 2016-08-13-ruby-rpm-ansible 23:06:31+08:00
layout: post
title: Centos6.5利用RubyGems的fpm制作zabbix_agent的rpm包，并使用ansible批量部署
categories: linux
tags: ruby gem fpm zabbix zabbix_agent rpm包制作 ansible
---
# 一、 搭建rpm包制作环境 #
安装gcc

	[root@lvs1 ~]# yum install gcc

安装make

	[root@lvs1 ~]# yum install make

安装ruby源（ruby版本必须要在1.9.3以上，centos自带的是1.8的版本，需要自己编译安装）

	[root@lvs1 ~]# yum install ruby rubygems ruby-devel

查看ruby源

	[root@lvs1 ~]# gem source list

添加国内源

	[root@lvs1 ~]# gem sources -a https://ruby.taobao.org/

移除国外源

	[root@lvs1 ~]# gem sources -r http://rubygems.org/

再次查看ruby源

	[root@lvs1 ~]# gem source list

升级ruby版本到最新

	[root@lvs1 ~]# gem update --system

安装fpm

	[root@lvs1 zlib]# gem install fpm

# 二、编译安装到本地 #

编译安装到本地

	[root@lvs1 ~]# tar -xzvf zabbix-3.0.4.gz
	[root@lvs1 zabbix-3.0.4]# cd zabbix-3.0.4
	[root@lvs1 zabbix-3.0.4]# ./configure --prefix=/opt/zabbix  --sysconfdir=/opt/zabbix  --enable-agent --disable-server --disable-proxy
	#--prefix为安装目录
	#--sysconfdir为配置文件目录
	#--enable-agent --disable-server --disable-proxy    安装agent不安装server和proxy，因为zabbix官方提供的源码包包含了所有组件，这里我们只需要agent所以不用全部安装
	[root@lvs1 zabbix-3.0.4]# make install

修改配置文件

	#可以用命令修改
	sed -i 's/^Server=.*$/Server=192.168.13.45/g' /opt/zabbix/zabbix_agentd.conf
	sed -i 's/^ServerActive=.*$/ServerActive=192.168.13.45:10051/g' /opt/zabbix/zabbix_agentd.conf
	sed -i 's/^LogFile=.*$/LogFile=\/opt\/zabbix\/logs\/zabbix_agentd.log/g' /opt/zabbix/zabbix_agentd.conf
	sed -i "s%^#HostnameItem=.*$%HostnameItem=system.hostname%g" /opt/zabbix/zabbix_agentd.conf
	sed -i "s%^#ListenIP=.*$%ListenIP=0.0.0.0%g" /opt/zabbix/zabbix_agentd.conf
	sed -i "s%^#ListenPort=.*$%ListenPort=10050%g" /opt/zabbix/zabbix_agentd.conf
	
	#也可以直接修改配置文件
	[root@lvs1 ~]# vim /opt/zabbix/zabbix_agentd.conf
	#zabbix_server的地址
	Server=192.168.13.45
	#主动上传给server的地址和端口
	ServerActive=192.168.13.45:10051
	#日志位置
	LogFile=/opt/zabbix/logs/zabbix_agentd.log
	#主机名取系统主机名
	HostnameItem=system.hostname
	#监听端口
	ListenPort=10050
	#监听地址
	ListenIP=0.0.0.0

复制编译包中对应系统的启动脚本到安装目录下

	[root@lvs1 core]# cp -a /root/zabbix-3.0.4/misc/init.d/fedora/core/zabbix_agentd /opt/zabbix/

修改启动脚本中安装目录的位置

	[root@lvs1 zabbix]# vim /opt/zabbix/zabbix_agentd
        BASEDIR=/opt/zabbix

# 三、创建安装后脚本和卸载后脚本 #

创建安装后执行脚本，在文件安装到本地后会做一些初始化操作

	[root@lvs1 ~]# vim /opt/install_after.sh
	#!/bin/bash
	#创建对应的用户和组以及日志目录，并给安装目录对应的权限
	groupadd zabbix
	useradd -g zabbix zabbix
	chown zabbix:zabbix /opt/zabbix
	mkdir -p /opt/zabbix/logs
	chown zabbix:zabbix /opt/zabbix/logs
	#这里把刚才复制的启动脚本链接到系统目录中
	ln -s /opt/zabbix/zabbix_agentd /etc/init.d/zabbix_agentd
	#判断是否有多个192.168网段的ip，因本人所在公司网络环境负责存在多网卡多ip情况，为防止出现问题，所以此脚本会把单网卡主机的监听ip改为本机，如果存在多个网卡是192.168网段则依然使用0.0.0.0
	ifip=$(ifconfig|sed -n '/inet addr/s/^[^:]*:\([0-9.]\{7,15\}\).*/\1/p' | grep '192.168.')
	ifwc=$(ifconfig|sed -n '/inet addr/s/^[^:]*:\([0-9.]\{7,15\}\).*/\1/p' | grep '192.168.'|wc -l)
	if [ $ifwc -gt 1 ];then
		echo $ifip
	elif [ $ifwc -eq 1 ];then
		sed -i "s%^ListenIP=0.0.0.0%ListenIP=$ifip%g" /opt/zabbix/zabbix_agentd.conf
	fi
	#启动agent
	service zabbix_agentd start
	#添加开机启动
	chkconfig --add zabbix_agentd
	chkconfig --level 35 zabbix_agentd on
	#添加iptables规则，允许对应端口通信，并保存规则
	iptables -I INPUT -m state --state new -m tcp -p tcp --dport 10050 -j ACCEPT
	iptables -I INPUT -m state --state new -m tcp -p tcp --dport 10051 -j ACCEPT
	/etc/init.d/iptables save
	exit 0

创建卸载后清理脚本，会清理安装目录和前面安装脚本添加的一些设置

	[root@lvs1 ~]# vim /opt/remove_after.sh
	#!/bin/bash
	service zabbix_agentd stop
	rm -rf /opt/zabbix
	rm -f /etc/init.d/zabbix_agentd
	userdel -r zabbix
	groupdel zabbix
	chkconfig --del zabbix_agentd
	chkconfig --level 35 zabbix_agentd off
	exit 0


整个rpm包安装后的目录结构

	opt
	├── install_after.sh
	├── remove_after.sh
	└── zabbix
	    ├── bin
	    │   ├── zabbix_get
	    │   └── zabbix_sender
	    ├── lib
	    ├── logs
	    │   └── zabbix_agentd.log
	    ├── sbin
	    │   └── zabbix_agentd
	    ├── share
	    │   └── man
	    │       ├── man1
	    │       │   ├── zabbix_get.1
	    │       │   └── zabbix_sender.1
	    │       └── man8
	    │           └── zabbix_agentd.8
	    ├── zabbix_agentd
	    ├── zabbix_agentd.conf
	    └── zabbix_agentd.conf.d

# 四、制作RPM包 #

	[root@lvs1 ~]# fpm -s dir -t rpm -n zabbix_agentd -v 3.0.4 -C / -p /root/ --post-install /opt/install_after.sh --post-uninstall /opt/remove_after.sh --no-rpm-sign /opt

	-s:指定源类型
	-t:指定目标类型，即想要制作为什么包
	-n:指定包的名字
	-v:指定包的版本号
	-C:指定打包的相对路径
	-d:指定依赖于哪些包
	-f:第二次包时目录下如果有同名安装包存在，则覆盖它
	-p:输出的安装包的目录，不想放在当前目录下就需要指定
	--post-install:软件包安装完成之后所要运行的脚本；同--offer-install
	--pre-install:软件包安装完成之前所要运行的脚本；同--before-install
	--post-uninstall:软件包卸载完成之后所要运行的脚本；同--offer-remove
	--pre-uninstall:软件包卸载完成之前所要运行的脚本；同—before-remove

注意：-C是相对目录，--no-rpm-sign才是安装目录

例如：-C / --no-rpm-sign /opt 则安装到/opt中，	再如：-C /tmp --no-rpm-sign /zabbix 则安装到/tmp/zabbix中



# 五、使用ansible批量部署 #

在hosts文件中加入分组和分组内主机，因为我公司没用密钥，所以这里我直接将账号密码写入了文件中，sudo的密码也写入了文件中，利用sudo切换到root权限，当然以下密码都是我乱写的。

	root@lv:~# vim /etc/ansible/hosts 
	[lvs]
	192.168.80.138 ansible_ssh_user=abc ansible_ssh_pass=!@#qwe ansible_sudo_pass=!@#qwe

用ifconfig命令测试下是否能正常使用，这里解释下-k命令，因为我公司sudo命令后是要输密码的，所以这里加了-k

	root@lv:~# ansible lvs -s -m command -a 'ifconfig'

将制作好的rpm复制给lvs组所有成员主机

	root@lv:~# ansible lvs -s -m copy -a 'src=/root/zabbix_agentd-3.0.4-1.x86_64.rpm dest=/root/'

给所有主机上的rpm包执行权限，其实不给也没影响

	root@lv:~# ansible lvs -s -m command -a 'chmod +x /root/zabbix_agentd-3.0.4-1.x86_64.rpm'

安装rpm包，因为我们设置的安装完启动，所以这部完成后就大功告成了

	root@lv:~# ansible lvs -s -m command -a 'rpm -ivh /root/zabbix_agentd-3.0.4-1.x86_64.rpm'
