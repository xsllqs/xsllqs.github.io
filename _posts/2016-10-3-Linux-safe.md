---
date: 2016-10-3-Linux-safe 09:32:31+08:00
layout: post
title: Linux系统安全设置
categories: linux
tags: Linux系统安全 信息安全 密码安全 远程登录控制
---
一、账号管理

1、用户密码

检测方法：

	（1）是否存在如下类似的简单用户密码配置，比如：root/root, test/test, root/root1234
	（2）执行：more /etc/login.defs，检查PASS_MAX_DAYS/ PASS_MIN_DAYS/PASS_WARN_AGE参数
	（3）执行：awk -F: '($2 == "") { print $1 }' /etc/shadow, 检查是否存在空口令帐号

建议安全标准：

	（1）在/etc/login.defs文件中配置：
	PASS_MAX_DAYS   90        #新建用户的密码最长使用天数
	PASS_MIN_DAYS   0          #新建用户的密码最短使用天数
	PASS_WARN_AGE   14        #新建用户的密码到期提前提醒天数
	（2）不允许存在空口令帐号

2、密码强度

检测方法：

	/etc/pam.d/system-auth文件中是否对pam_cracklib.so的参数进行了正确设置。

建议安全标准：

	密码长度至少8位，并包括数字、小写字母、大写字母和特殊符号4种中至少3种
	建议在/etc/pam.d/system-auth 文件中配置：
	password  requisite pam_cracklib.so difok=3 minlen=8 ucredit=-1 lcredit=-1 dcredit=1 
	

3、用户锁定策略

检测方法：

	/etc/pam.d/system-auth文件中是否对pam_tally.so的参数进行了正确设置。

建议安全标准：

	设置连续输错10次密码，帐号锁定5分钟，
	使用命令“vim /etc/pam.d/system-auth”修改配置文件，添加
	auth required pam_tally.so onerr=fail deny=10 unlock_time=300
	注：解锁用户 faillog  -u  <用户名>  -r

4、禁止root用户远程登录

检测方法：

	执行：more /etc/securetty，检查Console参数

建议安全标准：

	建议在/etc/securetty文件中配置：CONSOLE = /dev/tty01

5、检查是否存在root外的UID为0用户

检测方法：

	执行：awk -F: '($3 == 0) { print $1 }' /etc/passwd

建议安全标准：

	UID为0的任何用户都拥有系统的最高特权，保证只有root用户的UID为0，返回值包括“root”以外的条目，则低于安全要求；

6、检查root环境变量中是否包含777的目录

检测方法：

	执行：echo $PATH | egrep '(^|:)(\.|:|$)'，检查是否包含父目录，
	执行：find `echo $PATH | tr ':' ' '` -type d \( -perm -002 -o -perm -020 \) -ls，检查是否包含组目录权限为777的目录

建议安全标准：

	确保root用户的系统路径中不包含父目录，在非必要的情况下，不应包含组权限为777的目录

7、远程连接的安全性配置

检测方法：

	执行：find  / -name  .netrc，检查系统中是否有.netrc文件，
	执行：find  / -name  .rhosts ，检查系统中是否有.rhosts文件

建议安全标准：

	如无必要，删除这两个文件

8、用户的umask安全配置

检测方法：

	执行：more /etc/profile  more /etc/csh.login  more /etc/csh.cshrc  more /etc/bashrc检查是否包含umask值且umask=027

建议安全标准：

	建议设置用户的默认umask=027

9、重要目录和文件的权限

检测方法：

	执行以下命令检查目录和文件的权限设置情况：
	ls -l /etc/
	ls -l /etc/rc.d/init.d/
	ls -l /etc/inetd.conf	
	ls -l /etc/passwd
	ls -l /etc/shadow
	ls -l /etc/group
	ls -l /etc/security
	ls -l /etc/services
	ls -l /etc/rc*.d

建议安全标准：

	对于重要目录，建议执行如下类似操作：
	chmod -R 750 /etc/rc.d/init.d/*
	这样只有root可以读、写和执行这个目录下的脚本。

10、查找未授权的SUID/SGID文件

检测方法：

	for PART in `grep -v ^# /etc/fstab | awk '($6 != "0") {print $2 }'`; do
	find $PART \( -perm -04000 -o -perm -02000 \) -type f -xdev -print
	done

建议安全标准：

	若存在未授权的文件，则低于安全要求
	建议经常性的对比suid/sgid文件列表，以便能够及时发现可疑的后门程序

11、检查任何人都有写权限的目录

检测方法：

	for PART in `awk '($3 == "ext2" || $3 == "ext3") \
	{ print $2 }' /etc/fstab`; do
	find $PART -xdev -type d \( -perm -0002 -a ! -perm -1000 \) -print
	done

建议安全标准：

	若返回值非空，则低于安全要求

12、查找任何人都有写权限的文件

检测方法：

	for PART in `grep -v ^# /etc/fstab | awk '($6 != "0") {print $2 }'`; do
	find $PART -xdev -type f \( -perm -0002 -a ! -perm -1000 \) -print
	done

建议安全标准：

	若返回值非空，则低于安全要求

13、检查没有属主的文件

检测方法：

	for PART in `grep -v ^# /etc/fstab | awk '($6 != "0") {print $2 }'`; do
	find $PART -nouser -o -nogroup -print
	done
	注意：不用管“/dev”目录下的那些文件。

建议安全标准：

	若返回值非空，则低于安全要求
	发现没有属主的文件往往就意味着有黑客入侵你的系统了。不能允许没有主人的文件存在。如果在系统中发现了没有主人的文件或目录，先查看它的完整性，如果一切正常，给它一个主人。有时候卸载程序可能会出现一些没有主人的文件或目录，在这种情况下可以把这些文件和目录删除掉

14、检查异常隐含文件

检测方法：

	用“find”程序可以查找到这些隐含文件。例如：
	find  / -name ".. *" -print -xdev
	find  / -name "…*" -print -xdev | cat -v
	同时也要注意象“.xx”和“.mail”这样的文件名的。（这些文件名看起来都很象正常的文件名）

建议安全标准：

	若返回值非空，则低于安全要求
	在系统的每个地方都要查看一下有没有异常隐含文件（点号是起始字符的，用“ls”命令看不到的文件），因为这些文件可能是隐藏的黑客工具或者其它一些信息（口令破解程序、其它系统的口令文件，等等）。在UNIX下，一个常用的技术就是用一些特殊的名，如：“…”、“..    ”（点点空格）或“..^G”（点点control-G），来隐含文件或目录。

15、登录超时设置

检测方法：

	使用命令“cat /etc/profile |grep TMOUT”查看TMOUT是否设置

建议安全标准：

	使用命令“vi /etc/profile”修改配置文件，添加“TMOUT=”行开头的注释，建议设置为“TMOUT=180”，即超时时间为3分钟

16、远程登录设置

检测方法：

	查看SSH服务状态：
	service ssh status
	查看telnet服务状态：
	service telnet status

建议安全标准：

	SSH服务状态查看结果为：running 
	telnet服务状态查看结果为：not running/unrecognized

17、Root远程登录限制

检测方法：

	使用命令“cat /etc/ssh/sshd_config”查看配置文件
	（1）检查是否允许root直接登录
	检查“PermitRootLogin ”的值是否为no
	（2）检查SSH使用的协议版本
	检查“Protocol”的值

建议安全标准：

	（1）不允许root直接登录
	设置“PermitRootLogin ”的值为no
	设置后root用户需要使用普通用户远程登录后su进行系统管理
	（2）修改SSH使用的协议版本
	设置“Protocol”的版本为2

18、关闭不必要的服务

检测方法：

	使用命令“who -r”查看当前init级别
	使用命令“chkconfig --list <服务名>”查看所有服务的状态

建议安全标准：

	若有不必要的系统在当前级别下为on，则低于安全要求
	使用命令“chkconfig --level <init级别> <服务名> on|off|reset”设置服务在个init级别下开机是否启动

二、日志审计

1、syslog登录事件记录

检测方法：

	执行命令：more /etc/syslog.conf
	查看参数authpriv值

建议安全标准：

	若未对所有登录事件都记录，则低于安全要求

2、Syslog.conf的配置审核

检测方法：

	执行：more /etc/syslog.conf，查看是否设置了下列项：
	kern.warning;*.err;authpriv.none\t@loghost
	*.info;mail.none;authpriv.none;cron.none\t@loghost
	*.emerg\t@loghost
	local7.*\t@loghost

建议安全标准：

	若未设置，则低于安全要求
	建议配置专门的日志服务器，加强日志信息的异地同步备份

三、系统文件

1、系统core dump状态

检测方法：

    执行：more /etc/security/limits.conf 检查是否包含下列项：
    * soft core 0
    * hard core 0

建议安全标准：

	若不存在，则低于安全要求
	core dump中可能包括系统信息，易被入侵者利用，建议关闭
