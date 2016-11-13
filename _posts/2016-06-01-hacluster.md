---
date: 2016-06-01 18:04:24+08:00
layout: post
title: Centos7上利用corosync+pacemaker+crmsh构建高可用集群
categories: linux
tags: HA Cluster corosync pacemaker crmsh 高可用 集群
---

# 一、高可用集群框架 #

![1](https://xsllqs.github.io/assets/2016-06-01-hacluster1.png)

资源类型：

    primitive(native)：表示主资源
    group：表示组资源，组资源里包含多个主资源
    clone：表示克隆资源
    masterslave：表示主从资源

资源约束方式：

    位置约束：定义资源对节点的倾向性
    排序约束：定义资源彼此能否运行在同一节点的倾向性
    顺序约束：多个资源启动顺序的依赖关系

HA集群常用的工作模型：

    AP：两节点，activepassive，工作于主备模型
    AA：两节点，activeactive，工作于主主模型
    N-M：NM，N个节点，M个服务，假设每个节点运行一个服务，活动节点数为N，备用节点数为N-M

在集群分裂(split-brain)时需要使用到资源隔离，有两种隔离级别：

    STONITH：节点级别的隔离，通过断开一个节点的电源或者重新启动节点
    fencing：资源级别的隔离，类似通过向交换机发出隔离信号，特意让数据无法通过此接口
    当集群分裂，即分裂后的一个集群的法定票数小于总票数一半时采取对资源的控制策略，

# 二、在centos7上建立Ha cluster #

centos7(corosync v2 + pacemaker)

集群的全生命周期管理工具：

    pcs agent(pcsd)
    crmsh agentless (pssh)

## 1、集群配置前提 ##

时间同步，基于当前正在使用的主机名互相访问，是否会用到仲裁设备

	#更改主机名，两台主机的都要修改（192.168.1.114为ns2.xinfeng.com）（192.168.1.113为ns3.xinfeng.com）
	[root@ns2 ~]# hostnamectl set-hostname ns2.xinfeng.com
	[root@ns2 ~]# uname -n
	ns2.xinfeng.com
	[root@ns2 ~]# vim etchosts
	192.168.1.114   ns2.xinfeng.com
	192.168.1.113   ns3.xinfeng.com
	#同步时间
	[root@ns2 ~]# ntpdate s1a.time.edu.cn
	[root@ns2 ~]# ssh 192.168.1.113 'date';date
	The authenticity of host '192.168.1.113 (192.168.1.113)' can't be established.
	ECDSA key fingerprint is 09f9398c354dba2d134f3c9cb15854ec.
	Are you sure you want to continue connecting (yesno) yes
	Warning Permanently added '192.168.1.113' (ECDSA) to the list of known hosts.
	root@192.168.1.113's password 
	2016年 05月 28日 星期六 131807 CST
	2016年 05月 28日 星期六 131807 CST

## 2、安装pcs并启动集群 ##

	#192.168.1.113
	[root@ns3 ~]# yum install pcs
	#192.168.1.114
	[root@ns2 ~]# yum install pcs
	#用ansible开起服务并使服务开机运行
	[root@ns2 ~]# vim etcansiblehosts
	[ha]
	192.168.1.114
	192.168.1.113
	[root@ns2 ~]# ansible ha -m service -a 'name=pcsd state=started enabled=yes'
	192.168.1.113  SUCCESS = {
	    changed false, 
	    enabled true, 
	    name pcsd, 
	    state started
	}
	192.168.1.114  SUCCESS = {
	    changed true, 
	    enabled true, 
	    name pcsd, 
	    state started
	}
	#用ansible查看服务是否启动
	[root@ns2 ~]# ansible ha -m shell -a 'systemctl status pcsd'
	192.168.1.114  SUCCESS  rc=0 
	● pcsd.service - PCS GUI and remote configuration interface
	   Loaded loaded (usrlibsystemdsystempcsd.service; enabled; vendor preset disabled)
	   Active active (running) since 六 2016-05-28 133619 CST; 2min 32s ago
	 Main PID 2736 (pcsd)
	   CGroup system.slicepcsd.service
	           ├─2736 binsh usrlibpcsdpcsd start
	           ├─2740 binbash -c ulimit -S -c 0 devnull 2&1 ; usrbinruby -Iusrlibpcsd usrlibpcsdssl.rb
	           └─2741 usrbinruby -Iusrlibpcsd usrlibpcsdssl.rb
	
	5月 28 133616 ns2.xinfeng.com systemd[1] Starting PCS GUI and remote configuration interface...
	5月 28 133619 ns2.xinfeng.com systemd[1] Started PCS GUI and remote configuration interface.
	
	192.168.1.113  SUCCESS  rc=0 
	● pcsd.service - PCS GUI and remote configuration interface
	   Loaded loaded (usrlibsystemdsystempcsd.service; enabled; vendor preset disabled)
	   Active active (running) since 六 2016-05-28 133526 CST; 3min 24s ago
	 Main PID 2620 (pcsd)
	   CGroup system.slicepcsd.service
	           ├─2620 binsh usrlibpcsdpcsd start
	           ├─2624 binbash -c ulimit -S -c 0 devnull 2&1 ; usrbinruby -Iusrlibpcsd usrlibpcsdssl.rb
	           └─2625 usrbinruby -Iusrlibpcsd usrlibpcsdssl.rb
	
	5月 28 133524 ns3.xinfeng.com systemd[1] Starting PCS GUI and remote configuration interface...
	5月 28 133526 ns3.xinfeng.com systemd[1] Started PCS GUI and remote configuration interface.
	#给hacluster用户增加密码
	[root@ns2 ~]# ansible ha -m shell -a 'echo 123  passwd --stdin hacluster'
	192.168.1.113  SUCCESS  rc=0 
	更改用户 hacluster 的密码 。
	passwd：所有的身份验证令牌已经成功更新。
	
	192.168.1.114  SUCCESS  rc=0 
	更改用户 hacluster 的密码 。
	passwd：所有的身份验证令牌已经成功更新。
	#认证节点身份，用户名和密码为上面设置的hacluster和123，注意iptables规则，否则会出现无法联系的情况
	[root@ns2 ~]# pcs cluster auth ns2.xinfeng.com ns3.xinfeng.com
	Username hacluster
	Password 
	ns2.xinfeng.com Authorized
	ns3.xinfeng.com Authorized
	#配置集群，集群名字为xinfengcluster，集群中有2个节点
	[root@ns2 ~]# pcs cluster setup --name xinfengcluster ns2.xinfeng.com ns3.xinfeng.com
	Shutting down pacemakercorosync services...
	Redirecting to binsystemctl stop  pacemaker.service
	Redirecting to binsystemctl stop  corosync.service
	Killing any remaining services...
	Removing all cluster configuration files...
	ns2.xinfeng.com Succeeded
	ns3.xinfeng.com Succeeded
	Synchronizing pcsd certificates on nodes ns2.xinfeng.com, ns3.xinfeng.com...
	ns2.xinfeng.com Success
	ns3.xinfeng.com Success
	
	Restaring pcsd on the nodes in order to reload the certificates...
	ns2.xinfeng.com Success
	ns3.xinfeng.com Success
	#查看下配置文件
	[root@ns2 ~]# cat etccorosynccorosync.conf
	totem {        #集群的信息
	    version 2    #版本
	    secauth off    #安全功能是否开起
	    cluster_name xinfengcluster    #集群名称
	    transport udpu    #传输协议udpu也可以设置为udp
	}
	
	nodelist {    #集群中的所有节点
	    node {
	        ring0_addr ns2.xinfeng.com
	        nodeid 1    #节点ID
	    }
	
	    node {
	        ring0_addr ns3.xinfeng.com
	        nodeid 2
	    }
	}
	
	quorum {    #仲裁投票
	    provider corosync_votequorum    #投票系统
	    two_node 1    #是否为2节点集群
	}
	
	logging {    #日志
	    to_logfile yes    #是否记录日志
	    logfile varlogclustercorosync.log    #日志文件位置
	    to_syslog yes    #是否记录系统日志
	}
	#启动集群
	[root@ns2 ~]# pcs cluster start --all
	ns3.xinfeng.com Starting Cluster...
	ns2.xinfeng.com Starting Cluster...
	#查看ns2.xinfeng.com节点是否启动
	[root@ns2 ~]# corosync-cfgtool -s
	Printing ring status.
	Local node ID 1
	RING ID 0
		id	= 192.168.1.114
		status	= ring 0 active with no faults
	#查看ns3.xinfeng.com节点是否启动
	[root@ns3 ~]# corosync-cfgtool -s
	Printing ring status.
	Local node ID 2
	RING ID 0
		id	= 192.168.1.113
		status	= ring 0 active with no faults
	#查看集群信息
	[root@ns2 ~]# corosync-cmapctl  grep members
	runtime.totem.pg.mrp.srp.members.1.config_version (u64) = 0
	runtime.totem.pg.mrp.srp.members.1.ip (str) = r(0) ip(192.168.1.114) 
	runtime.totem.pg.mrp.srp.members.1.join_count (u32) = 1
	runtime.totem.pg.mrp.srp.members.1.status (str) = joined
	runtime.totem.pg.mrp.srp.members.2.config_version (u64) = 0
	runtime.totem.pg.mrp.srp.members.2.ip (str) = r(0) ip(192.168.1.113) 
	runtime.totem.pg.mrp.srp.members.2.join_count (u32) = 1
	runtime.totem.pg.mrp.srp.members.2.status (str) = joined
	[root@ns2 ~]# pcs status
	Cluster name xinfengcluster
	WARNING no stonith devices and stonith-enabled is not false
	Last updated Sat May 28 143823 2016		Last change Sat May 28 143315 2016 by hacluster via crmd on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	#DC为全局仲裁节点
	2 nodes and 0 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	
	PCSD Status
	  ns2.xinfeng.com Online
	  ns3.xinfeng.com Online
	
	Daemon Status
	  corosync activedisabled
	  pacemaker activedisabled
	  pcsd activeenabled

## 3、使用crmsh配置集群 ##

安装opensuse上的yum源

centos6

	cd etcyum.repos.d   
	wget httpdownload.opensuse.orgrepositoriesnetworkha-clusteringStableCentOS_CentOS-6networkha-clusteringStable.repo
	cd    
	yum -y install crmsh

centos7

	cd etcyum.repos.d   
	wget httpdownload.opensuse.orgrepositoriesnetworkha-clusteringStableCentOS_CentOS-7networkha-clusteringStable.repo
	cd    
	yum -y install crmsh

在其中一台主机上配置crmsh（192.1681.1.114）

	#显示当前的集群状态
	[root@ns2 yum.repos.d]# crm status
	Last updated Sat May 28 165252 2016		Last change Sat May 28 143315 2016 by hacluster via crmd on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 0 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources

## 4、两个节点上分别装上httpd ##

	[root@ns2 ~]# ansible ha -m shell -a 'yum install httpd -y'
	[root@ns2 ~]# echo h1ns2.xinfeng.comh1  varwwwhtmlindex.html
	[root@ns3 ~]# echo h1ns3.xinfeng.comh1  varwwwhtmlindex.html
	#测试能否正常启动，页面是否正常
	[root@ns2 ~]# ansible ha -m service -a 'name=httpd state=started enabled=yes'
	#centos6必须关闭服务，关闭开机启动，之后服务会变成资源，所以请确保服务不启动也不开机启动
	[root@ns2 ~]# ansible ha -m service -a 'name=httpd state=stopped enabled=no'
	#centos7必须关闭服务，开起开机启动
	[root@ns2 ~]# ansible ha -m service -a 'name=httpd state=stopped enabled=yes'

## 5、配置集群 ##

VIP为192.168.1.91，服务是httpd，将VIP和httpd作为资源来进行配置

	[root@ns2 ~]# crm
	crm(live)# ra    #进入资源代理
	crm(live)ra# classes    #查看可以代理的资源类型
	lsb
	ocf  .isolation heartbeat openstack pacemaker
	service
	stonith
	systemd
	crm(live)ra# list systemd    #查看systemd类型可代理的服务，其中有httpd
	NetworkManager                    NetworkManager-wait-online        auditd
	brandbot                          corosync                          cpupower
	crond                             dbus                              display-manager
	dm-event                          dracut-shutdown                   ebtables
	emergency                         exim                              firewalld
	getty@tty1                        httpd                             ip6tables
	iptables                          irqbalance                        kdump
	kmod-static-nodes                 ldconfig                          libvirtd
	lvm2-activation                   lvm2-lvmetad                      lvm2-lvmpolld
	lvm2-monitor                      lvm2-pvscan@82                   microcode
	network                           pacemaker                         pcsd
	plymouth-quit                     plymouth-quit-wait                plymouth-read-write
	plymouth-start                    polkit                            postfix
	rc-local                          rescue                            rhel-autorelabel
	rhel-autorelabel-mark             rhel-configure                    rhel-dmesg
	rhel-import-state                 rhel-loadmodules                  rhel-readonly
	rsyslog                           sendmail                          sshd
	sshd-keygen                       syslog                            systemd-ask-password-console
	systemd-ask-password-plymouth     systemd-ask-password-wall         systemd-binfmt
	systemd-firstboot                 systemd-fsck-root                 systemd-hwdb-update
	systemd-initctl                   systemd-journal-catalog-update    systemd-journal-flush
	systemd-journald                  systemd-logind                    systemd-machine-id-commit
	systemd-modules-load              systemd-random-seed               systemd-random-seed-load
	systemd-readahead-collect         systemd-readahead-done            systemd-readahead-replay
	systemd-reboot                    systemd-remount-fs                systemd-shutdownd
	systemd-sysctl                    systemd-sysusers                  systemd-tmpfiles-clean
	systemd-tmpfiles-setup            systemd-tmpfiles-setup-dev        systemd-udev-trigger
	systemd-udevd                     systemd-update-done               systemd-update-utmp
	systemd-update-utmp-runlevel      systemd-user-sessions             systemd-vconsole-setup
	tuned                             wpa_supplicant
	crm(live)ra# cd
	crm(live)# configure
	#配置资源，资源名为webip，ip为192.168.1.91
	crm(live)configure# primitive webip ocfheartbeatIPaddr params ip=192.168.1.91
	crm(live)configure# show
	node 1 ns2.xinfeng.com
	node 2 ns3.xinfeng.com
	primitive webip IPaddr 
		params ip=192.168.1.91
	property cib-bootstrap-options 
		have-watchdog=false 
		dc-version=1.1.13-10.el7_2.2-44eb2dd 
		cluster-infrastructure=corosync 
		cluster-name=xinfengcluster
	crm(live)configure# verify    #校验，因为没有隔离设备所以报错
	ERROR error unpack_resources	Resource start-up disabled since no STONITH resources have been defined
	   error unpack_resources	Either configure some or disable STONITH with the stonith-enabled option
	   error unpack_resources	NOTE Clusters with shared data need STONITH to ensure data integrity
	Errors found during check config not valid
	crm(live)configure# property stonith-enabled=false    #关闭隔离设备的设置
	crm(live)configure# verify    #再次校验
	crm(live)configure# commit    #提交，是配置生效
	crm(live)configure# cd
	crm(live)# status
	Last updated Sat May 28 174146 2016		Last change Sat May 28 174131 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 1 resource configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns2.xinfeng.com    #VIP已经启动在了ns2上
	 crm(live)# quit
	bye
	[root@ns2 ~]# ip addr    #验证一下VIP是否已经配置到网卡上
	1 lo LOOPBACK,UP,LOWER_UP mtu 65536 qdisc noqueue state UNKNOWN 
	    linkloopback 000000000000 brd 000000000000
	    inet 127.0.0.18 scope host lo
	       valid_lft forever preferred_lft forever
	    inet6 1128 scope host 
	       valid_lft forever preferred_lft forever
	2 eno16777728 BROADCAST,MULTICAST,UP,LOWER_UP mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    linkether 000c299157d1 brd ffffffffffff
	    inet 192.168.1.11424 brd 192.168.1.255 scope global dynamic eno16777728
	       valid_lft 5918sec preferred_lft 5918sec
	    inet 192.168.1.9124 brd 192.168.1.255 scope global secondary eno16777728
	       valid_lft forever preferred_lft forever
	    inet6 fe8020c29fffe9157d164 scope link 
	       valid_lft forever preferred_lft forever
	#将当前节点切换为备用节点
	[root@ns2 ~]# crm
	crm(live)# node
	crm(live)node# standby
	crm(live)node# cd 
	crm(live)# status
	Last updated Sat May 28 174541 2016		Last change Sat May 28 174534 2016 by root via crm_attribute on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 1 resource configured
	
	Node ns2.xinfeng.com standby
	Online [ ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns3.xinfeng.com
	#资源重新上线
	crm(live)# node online
	crm(live)# status
	Last updated Sat May 28 174640 2016		Last change Sat May 28 174637 2016 by root via crm_attribute on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 1 resource configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns3.xinfeng.com
	#配置httpd资源，资源名为httpd
	[root@ns2 ~]# crm
	crm(live)# configure
	crm(live)configure# primitive webser systemdhttpd 
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# cd
	crm(live)# status
	Last updated Sat May 28 175015 2016		Last change Sat May 28 174956 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 2 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns3.xinfeng.com
	 webser	(systemdhttpd)	Started ns2.xinfeng.com
	#将两个资源放在webhttp组中，资源启动顺序是webip，webser
	crm(live)# configure
	crm(live)configure# group webhttp webip webser
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# cd
	crm(live)# status
	Last updated Sat May 28 175248 2016		Last change Sat May 28 175241 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 2 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 Resource Group webhttp
	     webip	(ocfheartbeatIPaddr)	Started ns3.xinfeng.com
	     webser	(systemdhttpd)	Started ns3.xinfeng.com

由于是2个节点，会存在法定票数不足导致的资源不转移的情况，解决此问题的方法有四种：

	1、可以增加一个ping node节点。
	2、可以增加一个仲裁磁盘。
	3、让集群中的节点数成奇数个。
	4、直接忽略当集群没有法定票数时直接忽略。

这里我用的是第四种方式

	[root@ns2 ~]# crm
	crm(live)# configure
	crm(live)configure# property no-quorum-policy=ignore
	crm(live)configure# verify
	crm(live)configure# commit

这样做还不够，因为没有对资源进行监控，所以资源出现问题依然不会转移

现在只能测试下资源是否在ns3.xinfeng.com上启动了

![2](https://xsllqs.github.io/assets/2016-06-01-hacluster2.png)

要对资源进行监控需要在全局下命令primitive定义资源时一同定义，因此先把之前定义的资源删掉后重新定义

	[root@ns2 ~]# crm
	crm(live)# resource
	crm(live)resource# show
	 Resource Group webhttp
	     webip	(ocfheartbeatIPaddr)	Started
	     webser	(systemdhttpd)	Started
	crm(live)resource# stop webhttp    #停掉所有资源
	crm(live)resource# show
	 Resource Group webhttp
	     webip	(ocfheartbeatIPaddr)	(target-roleStopped) Stopped
	     webser	(systemdhttpd)	(target-roleStopped) Stopped
	crm(live)configure# edit    #编辑资源定义配置文件
	
	node 1 ns2.xinfeng.com 
	        attributes standby=off
	node 2 ns3.xinfeng.com
	primitive webip IPaddr             #删除
	        params ip=192.168.1.91        #删除
	primitive webser systemdhttpd        #删除
	group webhttp webip webser         #删除
	        meta target-role=Stopped        #删除
	property cib-bootstrap-options 
	        have-watchdog=false 
	        dc-version=1.1.13-10.el7_2.2-44eb2dd 
	        cluster-infrastructure=corosync 
	        cluster-name=xinfengcluster 
	        stonith-enabled=false 
	        no-quorum-policy=ignore
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# cd
	crm(live)# status
	Last updated Sat May 28 233201 2016		Last change Sat May 28 233149 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 0 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources

## 6、重新定义带有监控的资源 ##

	crm(live)# configure
	#每60秒监控一次，超时时长为20秒，时间不能小于建议时长，否则会报错
	crm(live)configure# primitive webip ocfIPaddr params ip=192.168.1.91 op monitor timeout=20s interval=60s
	crm(live)configure# primitive webser systemdhttpd op monitor timeout=20s interval=60s
	crm(live)configure# group webhttp webip webser
	crm(live)configure# property no-quorum-policy=ignore
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# cd
	crm(live)# status
	Last updated Sat May 28 234103 2016		Last change Sat May 28 234036 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns2.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 2 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 Resource Group webhttp
	     webip	(ocfheartbeatIPaddr)	Started ns2.xinfeng.com
	     webser	(systemdhttpd)	Started ns2.xinfeng.com

测试一下，将服务停掉，20秒后服务又自动会启动

	[root@ns2 ~]# ip addr
	1 lo LOOPBACK,UP,LOWER_UP mtu 65536 qdisc noqueue state UNKNOWN 
	    linkloopback 000000000000 brd 000000000000
	    inet 127.0.0.18 scope host lo
	       valid_lft forever preferred_lft forever
	    inet6 1128 scope host 
	       valid_lft forever preferred_lft forever
	2 eno16777728 BROADCAST,MULTICAST,UP,LOWER_UP mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    linkether 000c299157d1 brd ffffffffffff
	    inet 192.168.1.11424 brd 192.168.1.255 scope global dynamic eno16777728
	       valid_lft 6374sec preferred_lft 6374sec
	    inet 192.168.1.9124 brd 192.168.1.255 scope global secondary eno16777728
	       valid_lft forever preferred_lft forever
	    inet6 fe8020c29fffe9157d164 scope link 
	       valid_lft forever preferred_lft forever
	[root@ns2 ~]# service httpd stop
	Redirecting to binsystemctl stop  httpd.service

![3](https://xsllqs.github.io/assets/2016-06-01-hacluster3.png)

编辑配置文件，随便乱加几行让服务不能启动

	[root@ns2 ~]# vim etchttpdconfhttpd.conf 
	[root@ns2 ~]# service httpd stop
	Redirecting to binsystemctl stop  httpd.service

服务成功切换到ns3上

![4](https://xsllqs.github.io/assets/2016-06-01-hacluster4.png)

## 7、【注意】当重新恢复httpd服务后记得清除资源的错误信息，否则无法启动资源 ##

	crm(live)# resource
	crm(live)resource# cleanup webser    #清楚webser之前的错误信息
	Cleaning up webser on ns2.xinfeng.com, removing fail-count-webser
	Cleaning up webser on ns3.xinfeng.com, removing fail-count-webser
	Waiting for 2 replies from the CRMd.. OK
	crm(live)resource# show
	 webip	(ocfheartbeatIPaddr)	Started
	 webser	(systemdhttpd)	Started

## 8、定义资源约束 ##

删除组资源

	[root@ns2 ~]# crm
	crm(live)# configure
	crm(live)configure# delete webhttp    #删除webhttp组
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# cd
	crm(live)# status
	Last updated Sun May 29 092257 2016		Last change Sun May 29 092248 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns3.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 2 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns2.xinfeng.com
	 webser	(systemdhttpd)	Started ns3.xinfeng.com

排列约束

	[root@ns2 ~]# crm
	crm(live)# configure
	crm(live)configure# colocation webser_with_webip inf webser webip    #定义webser和webip两个资源必须在一起
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# cd
	crm(live)# status
	Last updated Sun May 29 094036 2016		Last change Sun May 29 094028 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns3.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 2 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns2.xinfeng.com    #可以看到两个资源都在ns2上启动了
	 webser	(systemdhttpd)	Started ns2.xinfeng.com

顺序约束

	crm(live)# configure
	#webip先于webser启动，强制的先启动webip在启动webser
	crm(live)configure# order webip_before_webser mandatory webip webser
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# show xml    #查看之前详细定义

位置约束

	#先看下当前的位置
	crm(live)# status
	Last updated Sun May 29 094842 2016		Last change Sun May 29 094435 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns3.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 2 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns2.xinfeng.com
	 webser	(systemdhttpd)	Started ns2.xinfeng.com
	#定义位置约束让资源更倾向于ns3上
	crm(live)# configure
	定义一个webservice的位置约束在节点2上，资源webip对ns3的倾向性是100
	crm(live)configure# location webservice_pref_node2 webip 100 ns3.xinfeng.com
	crm(live)configure# verify
	crm(live)configure# commit
	crm(live)configure# cd
	crm(live)# status
	Last updated Sun May 29 101212 2016		Last change Sun May 29 101204 2016 by root via cibadmin on ns2.xinfeng.com
	Stack corosync
	Current DC ns3.xinfeng.com (version 1.1.13-10.el7_2.2-44eb2dd) - partition with quorum
	2 nodes and 2 resources configured
	
	Online [ ns2.xinfeng.com ns3.xinfeng.com ]
	
	Full list of resources
	
	 webip	(ocfheartbeatIPaddr)	Started ns3.xinfeng.com    #很明显，资源都转移到了ns3上了
	 webser	(systemdhttpd)	Started ns3.xinfeng.com

# 三、总结 #

1、当重新恢复资源的服务后一定记得清除资源的错误信息，否则无法启动资源

2、在利用corosync+pacemaker且是两个节点实现高可用时，需要注意的是要设置全局属性把stonith设备关闭，忽略法定票数不大于一半的机制

3、注意selinux和iptables对服务的影响

4、注意节点相互用etchosts来解析

5、节点时间一定要保持同步

6、节点相互间进行无密钥通信

















