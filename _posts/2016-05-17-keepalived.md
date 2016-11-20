---
date: 2016-05-17 11:47:39+08:00
layout: post
title: keepalived实现lvs高可用并负载均衡lamp
categories: linux
tags: keepalived lvs lamp 高可用 负载均衡
---

一、安装lamp

1、安装httpd（172.16.23.211）

	[root@cs1 ~]# yum install -y httpd

2、安装php（172.16.23.211）

	[root@cs1 ~]# yum install -y php

3、安装php-mysql（172.16.23.211）

	[root@cs1 ~]# yum install -y php-mysql

4、安装mariadb（172.16.23.211 CentOS7）

	[root@cs1 ~]# yum install -y mariadb-server

5、配置MPM模型

这里我启用的是event模型

	[root@cs1 ~]# cd /etc/httpd/conf.modules.d/
	[root@cs1 conf.modules.d]# vim 00-mpm.conf
	#注释掉prefork，开起event
	#LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
	LoadModule mpm_event_module modules/mod_mpm_event.so

MPM：多路处理模块

prefork：是多进程模型，每个进程响应一个请求；

worker：是多进程多线程模型，一个主进程生成多个子进程，每个子进程负责生个多个线程，每个线程响应一个请求；

event：事件驱动模型，每个线程响应n个请求；

6、配置fast-cgi模块

查看模块是否存在，注意我安装的是httpd2.4

	[root@cs1 conf.modules.d]# vim /etc/httpd/conf.modules.d/00-proxy.conf 
	LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
查看模块是否加载

	[root@cs1 conf]# vim /etc/httpd/conf/httpd.conf 
	Include conf.modules.d/*.conf

7、修改httpd配置文件

	[root@cs1 conf]# vim /etc/httpd/conf/httpd.conf 
	ServerRoot "/etc/httpd"    #服务器根目录位置配置文件中没有使用绝对路径的地方，都认为是在该目录下
	
	Listen 80    #监听在80端口
	
	Include conf.modules.d/*.conf    #加载/etc/httd/conf.modules.d/下的.conf文件，所有的模块都在其中
	
	User apache    #访问httpd是进程使用的用户和组
	Group apache
	
	ServerAdmin root@localhost    #管理员邮箱
	
	ServerName cs1.xinfeng.com:80    #主机名
	
	<Directory />    #限制用户的目录访问权限
	    AllowOverride none
	    Require all denied
	</Directory>
	
	DocumentRoot "/var/www/html"    #url对应的根目录，这里cs1.xinfeng.com对应的就是这个目录
	
	<Directory "/var/www">
	    AllowOverride None
	    # Allow open access:
	    Require all granted    #all granted表示可无条件访问该目录
	</Directory>
	
	<Directory "/var/www/html">    #用于设定在该目录中哪些特性可用。默认这里有个Indexes选项，作用是当浏览器访问该目录如果该目录下没有默认网页(如index.html)，那么此时就会返回该目录下的文件名列表，所以建议取消掉
	    Options none
	    AllowOverride None
	    Require all granted
	</Directory>
	
	<IfModule dir_module>    #对指定的模块进行处理，
	    DirectoryIndex index.php index.html
	</IfModule>
	
	<Files ".ht*">    #任意目录下，文件名符合.ht*的文件都会被禁止访问。
	    Require all denied
	</Files>
	
	ErrorLog "logs/error_log"    #错误日志所在位置/etc/httpd/logs/error_log 
	
	LogLevel warn    #错误日志级别
	
	<IfModule log_config_module>
	    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
	    LogFormat "%h %l %u %t \"%r\" %>s %b" common
	    <IfModule logio_module>
	      # You need to enable mod_logio.c to use %I and %O
	      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio
	    </IfModule>
	    CustomLog "logs/access_log" combined    #访问日志的格式和纪录位置
	</IfModule>
	
	<IfModule alias_module>    #ScriptAlias会将URL路径映射到指定目录，并且让该目录具有CGI脚本执行权限（因此CGI脚本都可放置在该目录下）。
	    ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"
	</IfModule>
	
	<Directory "/var/www/cgi-bin">    #用于设定在该目录中哪些特性可用
	    AllowOverride None
	    Options None
	    Require all granted
	</Directory>
	
	<IfModule mime_module>    #关于mime模块的设置
	    TypesConfig /etc/mime.types
	    AddType application/x-compress .Z
	    AddType application/x-gzip .gz .tgz
	    AddType text/html .shtml
	    AddOutputFilter INCLUDES .shtml
	    
	    AddType application/x-httpd-php  .php    #让apache能识别php格式的页面
	    AddType application/x-httpd-php-source  .phps
	</IfModule>
	
	
	
	AddDefaultCharset UTF-8    #支持的编码格式为UTF-8
	
	<IfModule mime_magic_module>    
	    MIMEMagicFile conf/magic
	</IfModule>
	
	EnableSendfile on    #允许Apache使用系统核心支持的sendfile来传送文件给客户端 
	
	ProxyRequests Off    #关闭正向代理
	ProxyPassMatch ^/(.*\.php)$ fcgi://127.0.0.1:9000/var/www/html/$1    #把以.php结尾的文件请求发送到php-fpm进程
	
	
	
	IncludeOptional conf.d/*.conf    #在/etc/httpd/conf.d目录下以.conf结尾的配置文件也会被读取
	
	[root@cs1 ~]# httpd -t    #检查语法
	[root@cs1 conf.modules.d]# vim /var/www/html/index.php    #创建一个php文件
	<?php
	        phpinfo();
	?>
	[root@cs1 conf.modules.d]# systemctl start httpd

8、安装配置php-fpm

	[root@cs1 ~]# yum install php-fpm -y
	[root@cs1 ~]# vim /etc/php-fpm.d/www.conf 
	listen = 127.0.0.1:9000    #确保监听在9000端口
	listen.allowed_clients = 127.0.0.1
	[root@cs1 ~]# systemctl start php-fpm
	[root@cs1 conf.modules.d]# getenforce    #确保selinux关闭
	Disabled
	[root@cs1 conf.modules.d]# iptables -F    #清空防火墙规则
	[root@cs1 conf.modules.d]# iptables -L

9、安装配置phpMyAdmin

	[root@cs1 ~]# yum install -y phpMyAdmin
	[root@cs1 ~]# yum install php-mbstring -y
	[root@cs1 libraries]# vim /usr/share/phpMyAdmin/libraries/config.default.php
	#编辑配置文件
	$cfg['PmaAbsoluteUri'] = 'http://172.16.23.211/phpMyAdmin/';  #这里要填入phpMyAdmin所在的路径，这里也可以写成'http://cs1.xinfeng.com/phpMyAdmin/'
	
	[root@cs1 html]# vim /etc/httpd/conf.d/phpMyAdmin.conf 
	#修改一下几行
	<Directory /usr/share/phpMyAdmin/>
	   AddDefaultCharset UTF-8
	
	   <IfModule mod_authz_core.c>
	     # Apache 2.4
	     <RequireAny>
	#       Require ip 127.0.0.1
	#       Require ip ::1
	        Require all granted
	<Directory /usr/share/phpMyAdmin/setup/>
	   <IfModule mod_authz_core.c>
	     # Apache 2.4
	     <RequireAny>
	#       Require ip 127.0.0.1
	#       Require ip ::1
	        Require all granted
	
	        
	[root@cs1 ~]# vim /etc/phpMyAdmin/config.inc.php 
	$cfg['blowfish_secret'] = '1342758687478692';    #这里必须要给一个随机数
	
	[root@cs1 html]# ln -s /usr/share/phpMyAdmin /var/www/html/
	#这是将phpMyAdmin链接至httpd的根目录
	
	[root@cs1 ~]# systemctl restart php-fpm
	[root@cs1 ~]# systemctl restart httpd

进入http://172.16.23.211/phpMyAdmin/    测试能不能打开

![1](https://xsllqs.github.io/assets/2016-05-17-keepalived1.png)

进入http://172.16.23.211/index.php    测试能不能打开

![2](https://xsllqs.github.io/assets/2016-05-17-keepalived2.png)

10、配置mysql

	[root@cs1 ~]# systemctl start mariadb
	[root@cs1 ~]# mysql
	MariaDB [(none)]> create database php;    创建一个叫php的数据库
	Query OK, 1 row affected (0.01 sec)
	MariaDB [(none)]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| php                |
	| test               |
	+--------------------+
	MariaDB [(none)]> grant all privileges on php.* to xxoo@'%' identified by '123';    #创建一个xxoo用户密码为123，授权给php库，授权范围为全网
	Query OK, 0 rows affected (0.01 sec)
	MariaDB [(none)]> grant all privileges on php.* to xxoo@localhost identified by '123';    #授权范围本地
	Query OK, 0 rows affected (0.00 sec)
	MariaDB [(none)]> flush privileges;    #刷新权限
	Query OK, 0 rows affected (0.00 sec)

11、重启httpd,php-fpm，mariadb进入phpMyadmin测试

	[root@cs1 ~]# service php-fpm restart
	[root@cs1 ~]# service httpd restart
	[root@cs1 ~]# service mariadb restart

![3](https://xsllqs.github.io/assets/2016-05-17-keepalived3.png)

![4](https://xsllqs.github.io/assets/2016-05-17-keepalived4.png)

二、基于lamp安装wordpress

1、安装httpd（172.16.23.213）

	[root@cs2 ~]# yum install httpd -y

2、安装php（172.16.23.213）

	[root@cs2 ~]# yum install php -y

3、安装php-mysql（172.16.23.213）

	[root@cs1 ~]# yum install -y php-mysql

4、安装mariadb（172.16.23.213 CentOS7）

	[root@cs1 ~]# yum install -y mariadb-server

5、安装php-fpm

	[root@cs1 ~]# yum install php-fpm -y

6、安装phpMyAdmin

	[root@cs1 ~]# yum install -y phpMyAdmin
	[root@cs1 ~]# yum install php-mbstring -y

7、配置和上面的lamp相同，不创建index.php

8、下载安装配置wordpress

	[root@cs2 ~]# wget 
	[root@cs2 ~]# tar xvf latest.tar.gz  
	[root@cs2 ~]# ls
	anaconda-ks.cfg  latest.tar.gz  wordpress
	[root@cs2 ~]# chown root:root /root/wordpress    #改权限
	[root@cs2 ~]# chown root:root /root/wordpress/*
	[root@cs2 html]# cp -a /root/wordpress/* /var/www/html/    #将所有文件都复制到documentroot下
	[root@cs2 html]# vim wp-config-sample.php    #修改配置文件
	#我直接使用了php数据库，你也可以根据需要自己创建
	define('DB_NAME', 'php');
	
	#数据库用户名xxoo
	define('DB_USER', 'xxoo');
	
	#数据库密码123
	define('DB_PASSWORD', '123');
	
	#数据库位置，这里我安装的是本地，也可以指向其他有数据库的地址
	define('DB_HOST', '127.0.0.1');
	
	[root@cs2 html]# cp wp-config-sample.php wp-config.php
	[root@cs2 html]# service httpd restart
	[root@cs2 html]# service php-fpm restart

9、安装wordpress

在phpmyadmin中给wordpress创建一个数据库，这里我创建的数据库是之前在mysql中创建的php，并且授权给了用户xxoo的

![5](https://xsllqs.github.io/assets/2016-05-17-keepalived5.png)

![6](https://xsllqs.github.io/assets/2016-05-17-keepalived6.png)

![7](https://xsllqs.github.io/assets/2016-05-17-keepalived7.png)

![8](https://xsllqs.github.io/assets/2016-05-17-keepalived8.png)

![9](https://xsllqs.github.io/assets/2016-05-17-keepalived9.png)

![10](https://xsllqs.github.io/assets/2016-05-17-keepalived10.png)

![11](https://xsllqs.github.io/assets/2016-05-17-keepalived11.png)

![12](https://xsllqs.github.io/assets/2016-05-17-keepalived12.png)

三、基于lamp安装DiscuzX

1、安装httpd（172.16.23.215）

	[root@cs2 ~]# yum install httpd -y

2、安装php（172.16.23.215）

	[root@cs2 ~]# yum install php -y

3、安装php-mysql（172.16.23.215）

	[root@cs1 ~]# yum install -y php-mysql

4、安装mariadb（172.16.23.215 CentOS7）

	[root@cs1 ~]# yum install -y mariadb-server

5、安装php-fpm

	[root@cs1 ~]# yum install php-fpm -y

6、安装phpMyAdmin

	[root@cs1 ~]# yum install -y phpMyAdmin
	[root@cs1 ~]# yum install php-mbstring -y

7、配置方法和lamp一样，不创建index.php

8、下载解压配置DiscuzX

	[root@cs3 ~]# wget 
	[root@cs3 ~]# ls
	anaconda-ks.cfg  Discuz_X3.2_SC_UTF8.zip
	[root@cs3 ~]# mkdir Discuz
	[root@cs3 ~]# unzip -d /root/Discuz/ Discuz_X3.2_SC_UTF8.zip
	[root@cs3 ~]# cp -a /root/Discuz/* /var/www/html/
	[root@cs3 html]# chmod -R 777 /var/www/html/upload/*

9、进入首页进行配置，注意url

![13](https://xsllqs.github.io/assets/2016-05-17-keepalived13.png)

![14](https://xsllqs.github.io/assets/2016-05-17-keepalived14.png)

![15](https://xsllqs.github.io/assets/2016-05-17-keepalived15.png)

![16](https://xsllqs.github.io/assets/2016-05-17-keepalived16.png)

![17](https://xsllqs.github.io/assets/2016-05-17-keepalived17.png)

![18](https://xsllqs.github.io/assets/2016-05-17-keepalived18.png)

![19](https://xsllqs.github.io/assets/2016-05-17-keepalived19.png)

这里因为后续我要用lvs做负载均衡，所以需要把documentroot改一下

	[root@cs3 html]# vim /etc/httpd/conf/httpd.conf 
	DocumentRoot "/var/www/html/upload"
	<Directory "/var/www/html/upload">
	ProxyPassMatch ^/(.*\.php)$ fcgi://127.0.0.1:9000/var/www/html/upload/$1
	
	[root@cs3 html]# httpd -t
	Syntax OK
	
	[root@cs3 html]# rm phpmyadmin
	[root@cs3 html]# ln -s /usr/share/phpMyAdmin/ /var/www/html/upload/phpmyadmin
	[root@cs3 html]# service httpd restart
	[root@cs3 html]# service php-fpm restart
	[root@cs3 html]# service mariadb restart

改了之后的效果

![20](https://xsllqs.github.io/assets/2016-05-17-keepalived20.png)

![21](https://xsllqs.github.io/assets/2016-05-17-keepalived21.png)

![22](https://xsllqs.github.io/assets/2016-05-17-keepalived22.png)

四、keepalive实现lvs-dr

![23](https://xsllqs.github.io/assets/2016-05-17-keepalived23.png)

1、配置phpinfp（192.168.1.107）

	#让服务器忽略来自客户端计算机的ARP广播请求，防止服务器回答来自客户端查找VIP的ARP广播
	#接口可根据实际情况来定义，我这里用的本地回环接口
	[root@cs1 ~]# vim set.sh
	#!/bin/bash
	case $1 in
	start)
	        echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
	        echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	        echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
	        echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
	        ;;
	stop)
	        echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
	        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	        echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
	        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
	        ;;
	esac
	[root@cs1 ~]# bash set.sh start
	#将脚本传给另外wordpress和Discuz
	[root@cs1 ~]# scp set.sh 192.168.1.114:/root/
	[root@cs1 ~]# scp set.sh 192.168.1.113:/root/
	#修改lo接口的ip地址为VIP
	[root@cs1 ~]# ifconfig lo:0 192.168.1.33/32 broadcast 192.168.1.33 up
	#添加路由规则
	[root@cs1 ~]# route add -host 192.168.1.33 dev lo:0

2、配置wordpress(192.168.1.114)

	[root@cs2 ~]# bash set.sh start
	#修改lo接口的ip地址为VIP
	[root@cs2 ~]# ifconfig lo:0 192.168.1.33/32 broadcast 192.168.1.33 up
	#添加路由规则
	[root@cs2 ~]# route add -host 192.168.1.33 dev lo:0

3、配置Discuz（192.168.1.113）

	[root@cs3 ~]# bash set.sh start
	#修改lo接口的ip地址为VIP
	[root@cs3 ~]# ifconfig lo:0 192.168.1.33/32 broadcast 192.168.1.33 up
	#添加路由规则
	[root@cs3 ~]# route add -host 192.168.1.33 dev lo:0

4、配置director1（192.168.1.112）

	#安装ipvsadm工具
	[root@lvs1 ~]# yum install -y ipvsadm
	
	#安装配置keepalived
	[root@lvs1 ~]# yum install keepalived
	[root@lvs1 ~]# cp /etc/keepalived/keepalived.conf{,.bak}
	[root@lvs1 ~]# vim /etc/keepalived/keepalived.conf
	! Configuration File for keepalived
	
	global_defs {
	   notification_email {
	        root@localhost    #修改邮箱地址
	   }
	   notification_email_from admin@localhost
	   smtp_server 127.0.0.1    #修改smtp地址
	   smtp_connect_timeout 30
	   router_id LVS_DEVEL
	}
	
	vrrp_instance VI_1 {
	    state MASTER    #设为主服务器
	    interface eth0    #端口为et0
	    virtual_router_id 51    #虚拟路由id为51
	    priority 100    #优先级为100
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 68978103    #给以个随机数
	    }
	    virtual_ipaddress {
	        192.168.1.33/32    #VIP地址
	    }
	}
	
	virtual_server 192.168.1.33 80 {    #定义VIP
	    delay_loop 6
	    lb_algo rr    #lvs算法为rr
	    lb_kind DR    #lvs模式为DR
	    nat_mask 255.255.255.255    #子网掩码
	    protocol TCP
	
	    real_server 192.168.1.107 80 {    #phpinfo的地址
	        weight 1    #权重为1
	        TCP_CHECK {   #使用HTTP方式测试
	            connect_timeout 3    
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80    #检测80端口
	        }
	    }
	
	
	    real_server 192.168.1.114 80 {    #wordpress的地址
	        weight 1
	        TCP_CHECK { 
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	
	    real_server 192.168.1.113 80 {    #discuz的地址
	        weight 1
	        TCP_CHECK { 
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	}
	
	#将编辑好的keepalived.conf传给lvs2（192.168.1.111）
	[root@lvs1 ~]# scp /etc/keepalived/keepalived.conf 192.168.1.111:/etc/keepalived/
	#启动keepalived
	[root@lvs1 ~]# service keepalived start

5、配置director2（192.168.1.111）

	#安装keepalived
	[root@lvs2 ~]# yum install keepalived
	#编辑刚才从director1传来的配置文件中的2项即可
	[root@lvs2 ~]# vim /etc/keepalived/keepalived.conf
	vrrp_instance VI_1 {
	    state BACKUP    #这里改为BACKUP
	    interface eth0
	    virtual_router_id 51
	    priority 99    #这里将优先级改为99
	#启动keepalived
	[root@lvs2 ~]# service keepalived start

6、使用tcpdump抓包查看

	#使用tcpdump抓包查看是否成功
	[root@lvs1 ~]# tcpdump -i eth0 -nn host 192.168.1.111
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
	05:57:17.255964 ARP, Request who-has 192.168.1.111 tell 192.168.1.113, length 46
	05:57:17.255975 ARP, Reply 192.168.1.111 is-at 00:0c:29:63:cc:d8, length 46
	05:57:18.243218 IP 192.168.1.111.38941 > 192.168.1.113.80: Flags [S], seq 468244056, win 14600, options [mss 1460,sackOK,TS val 17008837 ecr 0,nop,wscale 5], length 0
	05:57:18.243230 IP 192.168.1.113.80 > 192.168.1.111.38941: Flags [S.], seq 137965257, ack 468244057, win 28960, options [mss 1460,sackOK,TS val 45829659 ecr 17008837,nop,wscale 6], length 0
	05:57:18.243417 IP 192.168.1.111.38941 > 192.168.1.113.80: Flags [.], ack 1, win 457, options [nop,nop,TS val 17008838 ecr 45829659], length 0
	05:57:18.243503 IP 192.168.1.111.38941 > 192.168.1.113.80: Flags [R.], seq 1, ack 1, win 457, options [nop,nop,TS val 17008838 ecr 45829659], length 0
	05:57:20.563328 IP 192.168.1.111.44763 > 192.168.1.107.80: Flags [S], seq 1861279836, win 14600, options [mss 1460,sackOK,TS val 17011157 ecr 0,nop,wscale 5], length 0
	05:57:20.563338 IP 192.168.1.107.80 > 192.168.1.111.44763: Flags [S.], seq 1361632953, ack 1861279837, win 28960, options [mss 1460,sackOK,TS val 45255624 ecr 17011157,nop,wscale 6], length 0
	05:57:20.563500 IP 192.168.1.111.44763 > 192.168.1.107.80: Flags [.], ack 1, win 457, options [nop,nop,TS val 17011158 ecr 45255624], length 0
	05:57:20.563504 IP 192.168.1.111.44763 > 192.168.1.107.80: Flags [R.], seq 1, ack 1, win 457, options [nop,nop,TS val 17011158 ecr 45255624], length 0
	05:57:20.917067 IP 192.168.1.111.57732 > 192.168.1.114.80: Flags [S], seq 950098347, win 14600, options [mss 1460,sackOK,TS val 17011511 ecr 0,nop,wscale 5], length 0
	05:57:20.917506 IP 192.168.1.114.80 > 192.168.1.111.57732: Flags [S.], seq 145530752, ack 950098348, win 28960, options [mss 1460,sackOK,TS val 45926618 ecr 17011511,nop,wscale 6], length 0
	05:57:20.918642 IP 192.168.1.111.57732 > 192.168.1.114.80: Flags [.], ack 1, win 457, options [nop,nop,TS val 17011512 ecr 45926618], length 0
	05:57:20.918650 IP 192.168.1.111.57732 > 192.168.1.114.80: Flags [R.], seq 1, ack 1, win 457, options [nop,nop,TS val 17011513 ecr 45926618], length 0
	#查看ip是否配置成功
	[root@lvs1 ~]# ip addr
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    link/ether 00:0c:29:52:50:3e brd ff:ff:ff:ff:ff:ff
	    inet 192.168.1.112/24 brd 192.168.1.255 scope global eth0
	    inet 192.168.1.33/32 scope global eth0
	    inet6 fe80::20c:29ff:fe52:503e/64 scope link 
	       valid_lft forever preferred_lft forever
	#用ipvsadm查看规则是否添加成功
	[root@lvs1 ~]# ipvsadm -L -n
	IP Virtual Server version 1.2.1 (size=4096)
	Prot LocalAddress:Port Scheduler Flags
	  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
	TCP  192.168.1.33:80 rr
	  -> 192.168.1.107:80             Route   1      0          0         
	  -> 192.168.1.113:80             Route   1      0          0         
	  -> 192.168.1.114:80             Route   1      0          0
	#我停掉192.168.1.112这台lvs1的keepalived服务实验一下
	[root@lvs1 ~]# service keepalived stop
	#进入192.168.1.111查看
	[root@lvs2 ~]# ip addr
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    link/ether 00:0c:29:63:cc:d8 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.1.111/24 brd 192.168.1.255 scope global eth0
	    inet 192.168.1.33/32 scope global eth0
	    inet6 fe80::20c:29ff:fe63:ccd8/64 scope link 
	       valid_lft forever preferred_lft forever
	You have new mail in /var/spool/mail/root
	
	#可以看到地址成功切换出去了

7、分别停到phpinfo、wordpress、discuz的httpd服务测试一下

停掉192.168.1.107和192.168.1.113的httpd

![24](https://xsllqs.github.io/assets/2016-05-17-keepalived24.png)

停掉192.168.1.114和192.168.1.113的httpd

![25](https://xsllqs.github.io/assets/2016-05-17-keepalived25.png)

停掉192.168.1.107和192.168.1.114的httpd

![26](https://xsllqs.github.io/assets/2016-05-17-keepalived26.png)

测试成功

五、keepalive实现lvs-nat

![27](https://xsllqs.github.io/assets/2016-05-17-keepalived27.png)

1、配置phpinfo（172.16.23.211）

	#配置网关
	[root@cs1 ~]# route add default gw 172.16.23.10
	[root@cs1 ~]# bash set.sh start

2、配置wordpress（172.16.23.213）

	#配置网关
	[root@cs2 ~]# route add default gw 172.16.23.10
	[root@cs2 ~]# bash set.sh start

3、配置discuz（172.16.23.215）

	#配置网关
	[root@cs3 ~]# route add default gw 172.16.23.10
	[root@cs3 ~]# bash set.sh start

4、配置director1(172.16.25.24)

	#打开路由转发
	[root@lvs1 ~]# echo "1">/proc/sys/net/ipv4/ip_forward
	[root@lvs1 ~]# vim /etc/sysctl.conf
	net.ipv4.ip_forward = 1
	[root@lvs1 ~]# sysctl -p
	#开起第二个网卡
	[root@lvs1 ~]#ifconfig eth1 up
	#配置keepalived
	[root@lvs1 ~]# vim /etc/keepalived/keepalived.conf
	! Configuration File for keepalived
	
	global_defs {
	   notification_email {
	        root@localhost
	   }
	   notification_email_from admin@localhost
	   smtp_server 127.0.0.1
	   smtp_connect_timeout 30
	   router_id LVS_1
	}
	vrrp_sync_group VG_1 {    #注意这里将DIP的别名和VIP定义为一个组，这样才能使两个地址同进退
	    group {
	        VI_1
	        VI_2
	        }
	}
	vrrp_instance VI_1 {    #这里定义VIP
	    state MASTER
	    interface eth1
	    virtual_router_id 53
	    priority 100
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 68978103
	    }
	    virtual_ipaddress {
	        172.16.23.33
	    }
	}
	vrrp_instance VI_2 {    #这里定义DIP别名
	    state MASTER
	    interface eth0
	    virtual_router_id 63
	    priority 100
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 68978103
	    }
	    virtual_ipaddress {
	        172.16.23.10
	    }
	}
	virtual_server 172.16.23.33 80 {
	    delay_loop 6
	    lb_algo wrr
	    lb_kind NAT
	    nat_mask 255.255.255.255
	    protocol TCP
	
	    real_server 172.16.23.211 80 {
	        weight 1
	        TCP_CHECK {
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	
	
	    real_server 172.16.23.213 80 {
	        weight 1
	        TCP_CHECK {
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	
	    real_server 172.16.23.215 80 {
	        weight 1
	        TCP_CHECK {
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	}
	#启动keepadlived
	[root@lvs1 ~]# service keepalived start

5、配置director2(172.16.25.83)

	[root@lvs2 ~]# echo "1">/proc/sys/net/ipv4/ip_forward
	[root@lvs2 ~]# vim /etc/sysctl.conf
	net.ipv4.ip_forward = 1
	[root@lvs2 ~]# sysctl -p
	[root@lvs2 ~]#ifconfig eth2 up
	[root@lvs2 ~]# vim /etc/keepalived/keepalived.conf
	! Configuration File for keepalived
	
	global_defs {
	   notification_email {
	        root@localhost
	   }
	   notification_email_from admin@localhost
	   smtp_server 127.0.0.1
	   smtp_connect_timeout 30
	   router_id LVS_2
	}
	vrrp_sync_group VG_1 {
	    group {
	        VI_1
	        VI_2
	        }
	}
	vrrp_instance VI_1 {
	    state BACKUP
	    interface eth2
	    virtual_router_id 53
	    priority 99
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 68978103
	    }
	    virtual_ipaddress {
	        172.16.23.33
	    }
	}
	vrrp_instance VI_2 {
	    state BACKUP
	    interface eth0
	    virtual_router_id 63
	    priority 99
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 68978103
	    }
	    virtual_ipaddress {
	        172.16.23.10
	    }
	}
	virtual_server 172.16.23.33 80 {
	    delay_loop 6
	    lb_algo wrr
	    lb_kind NAT
	    nat_mask 255.255.255.255
	    protocol TCP
	
	    real_server 172.16.23.211 80 {
	        weight 1
	        TCP_CHECK {
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	
	
	    real_server 172.16.23.213 80 {
	        weight 1
	        TCP_CHECK {
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	
	    real_server 172.16.23.215 80 {
	        weight 1
	        TCP_CHECK {
	            connect_timeout 3
	            nb_get_retry 3
	            delay_before_retry 3
	            connect_port 80
	        }
	    }
	}
	
	                                            
	#启动keepalived
	[root@lvs2 ~]# service keepalived start

6、查看是否配置成功

	[root@lvs2 ~]# ipvsadm -L -n
	IP Virtual Server version 1.2.1 (size=4096)
	Prot LocalAddress:Port Scheduler Flags
	  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
	TCP  172.16.23.33:80 wrr
	  -> 172.16.23.211:80             Masq    1      0          0         
	  -> 172.16.23.213:80             Masq    1      0          0         
	  -> 172.16.23.215:80             Masq    1      0          0
	[root@lvs1 ~]# ip addr
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    link/ether 00:0c:29:52:50:3e brd ff:ff:ff:ff:ff:ff
	    inet 172.16.25.24/16 brd 172.16.255.255 scope global eth0
	    inet 172.16.23.33/32 scope global eth0
	    inet6 fe80::20c:29ff:fe52:503e/64 scope link 
	       valid_lft forever preferred_lft forever
	3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    link/ether 00:0c:29:52:50:48 brd ff:ff:ff:ff:ff:ff
	    inet 172.16.23.10/32 scope global eth1
	    inet6 fe80::20c:29ff:fe52:5048/64 scope link 
	       valid_lft forever preferred_lft forever
	[root@lvs2 ~]# ip addr
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    link/ether 00:0c:29:63:cc:d8 brd ff:ff:ff:ff:ff:ff
	    inet 172.16.25.83/16 brd 172.16.255.255 scope global eth0
	    inet 172.16.23.33/32 scope global eth0
	    inet6 fe80::20c:29ff:fe63:ccd8/64 scope link 
	       valid_lft forever preferred_lft forever
	3: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
	    link/ether 00:0c:29:63:cc:e2 brd ff:ff:ff:ff:ff:ff
	    inet 172.16.23.10/32 scope global eth2
	    inet6 fe80::20c:29ff:fe63:cce2/64 scope link 
	       valid_lft forever preferred_lft forever
	#可以看到已经可以自由转换了

六、keepalive实现lvs-tun

![28](https://xsllqs.github.io/assets/2016-05-17-keepalived28.png)

    lvs-tun特点：
        不修改请求报文的ip首部，而是通过在原有的ip首部（cip<-->vip）之外，再封装一个ip首部(dip<-->rip)
            (1) RIP, DIP, VIP全得是公网地址
            (2) RS的网关的不能指向DIP
            (3) 请求报文必须经由director调度，但响应报文必须不能经由director
            (4) 不支持端口映射
            (5) RS的OS必须支持隧道功能

七、keepalive实现lvs-fullnat

![29](https://xsllqs.github.io/assets/2016-05-17-keepalived29.png)

    lvs-fullnat特点：
        director通过同时修改请求报文的目标地址和源地址进行转发
            (1) VIP是公网地址；RIP和DIP是私网地址，二者无须在同一网络中
            (2) RS接收到的请求报文的源地址为DIP，因此要响应给DIP
            (3) 请求报文和响应报文都必须经由Director
            (4) 支持端口映射机制
            (5) RS可以使用任意OS

八、lvs调度算法

    lvs调度算法分为两类，一类为静态算法，一类为动态算法。
        静态算法：根据算法本身进行调度
            RR：轮询
            WRR：加权的轮询
            SH：实现session保持的机制；将来自于同一个IP的请求始终调度至同一RS
            DH：将对同一个目标的请求始终发往同一个RS
        动态算法：根据算法及各RS的当前负载状态进行调度
            LC：最少连接数，那台连接数最少就调度哪台
            WLC：加权最少连接数
            SED：最短期望延迟
            NQ：SED算法的改进；
            LBLC：动态的DH算法；
            LBLCR：带复制功能的LBLC算法；

九、tcpdump的使用

tcpdump是一款抓包工具，用来监听指定网络接口的数据包流向

直接使用tcpdump会监听第一个网络接口的数据流向

    选项：
        -nn：直接以IP和端口号显示，而非主机名与服务名称
        -i ：后面接要监听的网络端口，例如eth0,lo等
        -w ：将监听的数据包结果储存下来，后面文件名
        -c ：监听的数据包数量，如果不接这个参数，tcpdump会持续不断的监听，直到输入ctrl+c为止
        -A ：数据包的内容以ASCII码显示，通常用来捉取网页数据包
        -e ：用mac地址来显示数据包
        -q ：仅列出较为简短的数据包结果，每一行的内容比较精简
        -X ：可以列出十六进制以及ASCII码的数据包内容，对于监听数据包内容很有用
        -r ：将之前存好的数据包文件读出来

    关键字：
        第一种是要监听的目标类型的关键字，主要包括host，net，port，如果不指定默认是host
        第二种是确定传输方向的关键字，主要包括src（来源），dst（目标）
        第三种是协议的关键字，主要包括fddi，ip，arp，rarp，tcp，udp
        其他重要的关键字：gateway， broadcast，less， greater，
        三种逻辑运算：
            非：可以用not也可以用 ! 
            与：可以用and也可以用&&
            或：用or

例：

	#tcpdump -i eth0 -nn host 192.168.1.111
	分析之前使用的这个命令
	监听主机192.168.1.111的th0网卡所流过的所有数据包，显示数据包的ip和端口

十、总结

1、创建基于lamp的RS服务器

2、在DR服务器上用keepalived配置ipvs规则

keepalived是用来实现lvs高可用的，而lvs是用来实现RS服务器负载均衡的

3、利用tcpdump抓包来查看keepalived下lvs服务器的数据包的流向


