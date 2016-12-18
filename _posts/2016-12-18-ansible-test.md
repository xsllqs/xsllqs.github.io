---
date: 2016-12-18 15:14:52+08:00
layout: post
title: 利用ansible-playbook从测试环境获取tomcat中java项目新版本发布到生产环境
categories: linux
tags: ansible yml 自动化版本发布 tomcat 升级 回滚 java
---

# 一、环境描述 #

安装有ansible的服务器：192.168.13.45

测试环境服务器：192.168.13.49

	/home/app/api-tomcat/webapps/api.war为测试环境新版本war包位置

生产环境服务器：192.168.13.51

	/home/app/api-tomcat/webapps/api.war为生产环境war包位置
	/home/app/api-tomcat/webapps/api为生产环境项目位置
	/home/app/tomcat.bak/api/webapps-时间戳，为老版本webapps备份位置
	/home/app/newwar/api.war为从测试环境获得的新版本war包临时存放位置
	/home/app/newwar/api为新版本war包解压后临时存放的位置

全部以app用户执行

# 二、编写ansible-playbook用的yml文件 #

## 1、升级 ##

这里所有的#开头的注释文字在使用的时候都要去掉，因为yml是没有注释的

	#生产环境主机的ip，这里也可以是/etc/ansible/hosts定义的组名
	- hosts: 192.168.13.51
	#变量，在yml文件中使用变量可以使整个文件可以用在不同的主机上升级，变量的使用方法是{{ 变量名 }}，如果task中的变量在冒号后则一定要将冒号后整句加上双引号""，因为yml文件自动把冒号后的大括号的内容识别为列表，如shell:"{{ oldhome }}/bin/startup.sh"
	  vars:
	#测试环境IP地址
	    testIP: 192.168.13.49
	#测试环境中项目的位置
	    testhome: /home/app/api-tomcat/webapps
	#测试环境中项目war包的名字
	    warname: api.war
	#生产环境中项目的tomcat所在的位置
	    oldhome: /home/app/api-tomcat
	#生产环境中老版本项目所在webapps备份目录的位置
	    backupwebapps: /home/app/tomcat.bak
	#从测试环境获取的新版本war包所在的位置
	    newwar: /home/app/newwar
	#新版本war包解压后目录的名字
	    zipname: api
	#整个远程自动化操作中所使用的账户，这里整个从生产环境到测试环境的操作都是用app用户执行的
	  remote_user: app
	#具体操作
	  tasks:
	    - name: 生产环境删除/home/app/newwar目录，若目录不存在则忽略错误（删这个目录的原因是因为之后要新建这个目录，确保整个yml文件可以多次执行，ignore_errors为是否忽略错误返回值）
	      file: path={{ newwar }} state=absent
	      ignore_errors: yes
	    - name: 生产环境创建/home/app/newwar目录，改权限，（其中recurse是递归创建目录，state是文件类型为目录）
	      file: path={{ newwar }} recurse=yes mode=775 owner=app group=app state=directory
	    - name: 从测试环境192.168.13.49复制新版本/home/app/api-tomcat/webapps/api.war包到生产环境192.168.13.51的/home/app/newwar目录下，此处之后的操作都是在生产环境下
	      shell: scp app@{{ testIP }}:{{ testhome }}/{{ warname }} {{ newwar }}
	    - name: 给/home/app/newwar递归改权限（因为整改操作都是以app用户身份执行的，所以一定要保证权限为app的权限）
	      file: dest={{ newwar}} recurse=yes mode=775 owner=app group=app
	    - name: 解压/home/app/newwar/api.war包在/home/app/newwar/api目录
	      shell: unzip -oq {{ newwar }}/{{ warname }} -d {{ newwar }}/{{ zipname }}
	    - name: 再次给/home/app/newwar递归改权限（确保新版本为app的权限）
	      file: dest={{ newwar}} recurse=yes mode=775 owner=app group=app
	    - name: 创建用来备份老版本webapps的目录/home/app/tomcat.bak/api并改递归权限
	      file: path={{ backupwebapps }}/{{ zipname }} recurse=yes mode=775 owner=app group=app state=directory
	    - name: 备份/home/app/api-tomcat/webapps到目录/home/app/tomcat.bak/api/webapps-时间戳（这个备份目录是用来回滚的）
	      shell: cp -a {{ oldhome }}/webapps {{ backupwebapps }}/{{ zipname }}/webapps-`date +%Y%m%d%H%M`
	    - name: kill进程方式停止服务.忽略错误返回值（用这种方式才能确保老版本停止运行，否则会出现冲突）
	      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
	      ignore_errors: yes
	    - name: kill进程方式停止服务.忽略错误返回值（再次确保老版本不再运行）
	      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
	      ignore_errors: yes
	    - name: 再次kill进程方式停止服务.忽略错误返回值
	      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
	      ignore_errors: yes
	    - name: 查看停止服务的结果，进程是否还在
	      shell: ps -ef | grep {{ oldhome }}
	    - name: 删除老版本的/home/app/api-tomcat/webapps/api.war包
	      file: path={{ oldhome }}/webapps/{{ warname }} state=absent
	      ignore_errors: yes
	    - name: 删除老版本的/home/app/api-tomcat/webapps/api程序目录
	      file: path={{ oldhome }}/webapps/{{ zipname }} state=absent
	      ignore_errors: yes
	    - name: 复制新版本目录/home/app/newwar/api到/home/app/api-tomcat/webapps目录下
	      shell: cp -a {{ newwar }}/{{ zipname }} {{ oldhome }}/webapps/
	    - name: 复制新版本war包/home/app/newwar/api.war包到/home/app/api-tomcat/webapps目录下
	      shell: cp -a {{ newwar }}/{{ warname }} {{ oldhome }}/webapps/
	    - name: 启动服务/home/app/api-tomcat/bin/startup.sh(source是为了载入jdk的环境变量，nohup是为了保证yml跑完了进程依然不退出)
	      shell: "source /etc/profile;nohup {{ oldhome }}/bin/startup.sh &"
	    - name: 查看进程中是否存在启动的服务
	      shell: ps -ef | grep {{ oldhome }}

## 2、回滚 ##

	#生产环境主机地址
	- hosts: 192.168.13.51
	#变量和升级的相同
	  vars:
	    testIP: 192.168.13.49
	    testhome: /home/app/api-tomcat/webapps
	    warname: api.war
	    oldhome: /home/app/api-tomcat
	    backupwebapps: /home/app/tomcat.bak
	    newwar: /home/app/newwar
	    zipname: api
	#远程操作依然使用app用户
	  remote_user: app
	#以下操作都是在生产环境中进行
	  tasks:
	    - name: kill进程方式停止服务.忽略错误返回值
	      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
	      ignore_errors: yes
	    - name: kill进程方式停止服务.忽略错误返回值
	      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
	      ignore_errors: yes
	    - name: 再次kill进程方式停止服务.忽略错误返回值
	      shell: ps -ef | grep {{ oldhome }} | grep -v grep | xargs kill
	      ignore_errors: yes
	    - name: 查看停止服务的结果.进程是否还在
	      shell: ps -ef | grep {{ oldhome }}
	    - name: 删除/home/app/api-tomcat/webapps目录
	      file: path={{ oldhome }}/webapps state=absent
	    - name: 显示/home/app/tomcat.bak/api/中最新备份的webapps目录，目录名应该是webapps-最近时间戳
	      shell: ls -r {{ backupwebapps }}/{{ zipname }} | head -1
	    - name: 复制备份的/home/app/tomcat.bak/api/webapps-最新时间戳，到项目并改名/home/app/api-tomcat/webapps
	      shell: cp -a {{ backupwebapps }}/{{ zipname }}/$(ls -r {{ backupwebapps }}/{{ zipname }} | head -1) {{ oldhome }}/webapps
	    - name: 启动服务/home/app/api-tomcat/bin/startup.sh
	      shell: "source /etc/profile;nohup {{ oldhome }}/bin/startup.sh &"
	    - name: 删除刚才回滚的备份文件
	      shell: rm -rf {{ backupwebapps }}/{{ zipname }}/$(ls -r {{ backupwebapps }}/{{ zipname }}
	    - name: 查看进程中是否存在启动的服务
	      shell: ps -ef | grep {{ oldhome }}

# 三、升级操作和注意事项 #

## 1、升级前免密钥操作 ##

ansible所在主机192.168.13.45

	#在app用户下生成密钥
	ssh-keygen -t rsa
	#发送公钥到测试环境
	ssh-copy-id -i .ssh/id_rsa.pub app@192.168.13.49
	#发送公钥到生产环境
	ssh-copy-id -i .ssh/id_rsa.pub app@192.168.13.51

生产环境主机192.168.13.51

	#在app用户下生成密钥
	ssh-keygen -t rsa
	#发送公钥到测试环境
	ssh-copy-id -i .ssh/id_rsa.pub app@192.168.13.49

为了业务安全，ansible所在主机和生产环境主机、测试环境主机是互通的。生产环境主机能连上测试环境主机，但测试环境主机不能连上生产环境主机，所以这里测试环境主机不需要将密钥发送给生产环境主机

## 2、升级和回滚 ##

升级

	ansible-playbook /home/app/api.yml -v

回滚

	ansible-playbook /home/app/api-rollback.yml -v

ansible-playbook后面跟上之前写的yml文件路径，-v是为了显示详细执行信息

## 3、注意 ##

如果在jenkins中执行升级和回滚的yml文件，一定要将在jenkins用户的公钥发送给生产环境主机和测试环境主机，否则会报权限错误

两个yml文件已在生产环境中验证
