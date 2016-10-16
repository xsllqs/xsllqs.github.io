---
date: 2016-10-16 10:35:24+08:00
layout: post
title: linux安全设置检测脚本
categories: linux
tags: Linux系统安全 信息安全 密码安全 远程登录控制 shell脚本
---

之前有发过一片关于Linux安全设置的文章，后来发现根据文章中的每项依次人工检测太慢太耗时，决定写一个shell脚本自动检测。

前一篇文章位置：[https://xsllqs.github.io/2016/10/03/Linux-safe.html](https://xsllqs.github.io/2016/10/03/Linux-safe.html "https://xsllqs.github.io/2016/10/03/Linux-safe.html")

本脚本只用于centos系统，debian系统的一些文件位置和centos不同，有需要的可以自己修改。

检测是否存在非root用户可执行文件处可以自己根据需要添加目录位置。

检测未授权的SUID/SGID文件处也可以根据自己需要添加目录位置。

检测无法识别的隐藏文件处也可以根据自己需要添加目录位置。

检测当前init级别开启且无法识别的服务处也可根据自己需要添加已知服务名称。

以下为脚本内容：

	#!/bin/bash
	#email:xsllqs@qq.com
	#by:信风
	#time:2016-10-16
	#本脚本只用于centos6系列
	
	psmxday=$(cat /etc/login.defs | egrep -v "^#|^$" | grep "PASS_MAX_DAYS" | awk '{print $2}')
	psminday=$(cat /etc/login.defs | egrep -v "^#|^$" | grep "PASS_MIN_DAYS" | awk '{print $2}')
	pswarn=$(cat /etc/login.defs | egrep -v "^#|^$" | grep "PASS_WARN_AGE" | awk '{print $2}')
	pskong=$(awk -F: '($2 == "") { print $1 }' /etc/shadow)
	
	if [ $psmxday -ne 90 ];then
		echo "/etc/login.defs的PASS_MAX_DAYS不为90 为$psmxday"
	fi
	
	if [ $psminday -ne 0 ];then
		echo "/etc/login.defs的PASS_MIN_DAYS不为0 为$psminday"
	fi
	
	if [ $pswarn -ne 0 ];then
		echo "/etc/login.defs的PASS_WARN_AGE不为14 为$pswarn"
	fi
	
	if [ -z "$pskong" ];then
		echo "OK,不存在空口令帐号"
	else 
		echo "存在空口令帐号 $pskong"
	fi
	
	pswd=$(cat /etc/pam.d/system-auth | grep "^password[[:space:]]\+requisite.*$")
	if [[ $pswd =~ "difok=3" ]] && [[ $pswd =~ "minlen=8" ]] && [[ $pswd =~ "ucredit=-1" ]] && [[ $pswd =~ "lcredit=-1" ]] && [[ $pswd =~ "dcredit=1" ]];then
		echo "OK,密码复杂度OK"
	else
		echo "/etc/pam.d/system-auth密码复杂度password requisite不足"
	fi
	
	tally=$(cat /etc/pam.d/system-auth | grep "pam_tally.so")
	if [ -z "$tally" ];then
		echo "无用户锁定策略，需要添加/etc/pam.d/system-auth中pam_tally.so项"
	elif [ -n "$tally" ] && [[ $tally =~ "onerr=fail" ]] && [[ $tally =~ "deny=10" ]] && [[ $tally =~ "unlock_time=300" ]];then
		echo "OK,用户锁定策略正确"
	else
		echo "用户锁定策略存在但不正确，修改/etc/pam.d/system-auth中pam_tally.so"
	fi
	
	console=$(cat /etc/securetty | grep -i "Console")
	if [ -n "$console" ] && [[ $console =~ "console=/dev/tty01" ]] || [[ $console =~ "Console=/dev/tty01" ]] || [[ $console =~ "console = /dev/tty01" ]] || [[ $console =~ "Console = /dev/tty01" ]] || [[ $console =~ "console=/dev/tty1" ]] || [[ $console =~ "Console=/dev/tty1" ]] || [[ $console =~ "console = /dev/tty1" ]] || [[ $console =~ "Console = /dev/tty1" ]];then
		echo "OK,root用户远程登录限制正确"
	else
		echo "root用户远程登录限制不正确，请修改/etc/securetty中CONSOLE = /dev/tty01"
	fi
	
	pswduid=$(awk -F: '($3 == 0) { print $1 }' /etc/passwd)
	if [ "$pswduid" = "root" ];then
		echo "OK,不存在root以外uid为0用户"
	else
		echo "存在root以外uid为0用户 $pswduid"
	fi
	
	fupath=$(echo $PATH | egrep '(^|:)(\.|:|$)')
	chpath=$(find `echo $PATH | tr ':' ' '` -type d \( -perm -002 -o -perm -020 \) -ls 2>/dev/null)
	if [ -n "$fupath" ];then
		echo "root用户环境变量包含父目录 $fupath"
	else
		echo "OK,root用户环境变量无父目录"
	fi
	if [ -n "$chpath" ];then
		echo "root用户环境变量包含组目录权限为777的目录 $chpath"
	else
		echo "OK,root用户环境变量无777目录"
	fi
	
	ycnetrc=$(find  / -name  .netrc)
	ycrhosts=$(find  / -name  .rhosts)
	if [ -n "$ycnetrc" ];then
		echo "含有.netrc文件 请删除$ycnetrc"
	else
		echo "OK,无.netrc文件"
	fi
	if [ -n "$ycrhosts" ];then
		echo "含有.rhosts文件 请删除$ycrhosts"
	else
		echo "OK,无.rhosts文件"
	fi
	
	pro1="/etc/profile"
	csh2="/etc/csh.login"
	cshrc3="/etc/csh.cshrc"
	bash4="/etc/bashrc"
	for contx in {$pro1,$csh2,$cshrc3,$bash4}
	do
	        cat $contx | egrep -v "^#|^[[:space:]]*#" | grep umask | while read huanj
	        do
	                if [[ $huanj =~ "027" ]];then
	                echo "OK,$contx的umask值为027"
	                else
	                echo "$contx的其中一个umask值不为027 为$huanj"
	                fi
	        done
	done
	
	mulu1="/etc/"
	mulu2="/etc/rc.d/init.d/"
	mulu3="/etc/inetd.conf"
	mulu4="/etc/passwd"
	mulu5="/etc/shadow"
	mulu6="/etc/group"
	mulu7="/etc/security"
	mulu8="/etc/services"
	mulu9="/etc/rc*.d"
	for suoyoumulu in {$mulu1,$mulu2,$mulu3,$mulu4,$mulu5,$mulu6,$mulu7,$mulu8,$mulu9}
	do
		mulujieguo=$(find $suoyoumulu -perm /u+x 2>/dev/null)
		if [[ -n $mulujieguo ]];then
			echo "$suoyoumulu存在非root用户可执行文件,建议 chmod -R 750 $suoyoumulu"
		else
			echo "OK,$suoyoumulu不存在非root用户可执行文件"
		fi
	done
	
	xitongzidai="
	/sbin/netreport
	/sbin/pam_timestamp_check
	/sbin/unix_chkpwd
	/bin/ping6
	/bin/umount
	/bin/ping
	/bin/su
	/bin/mount
	/usr/sbin/postdrop
	/usr/sbin/usernetctl
	/usr/sbin/postqueue
	/usr/bin/wall
	/usr/bin/newgrp
	/usr/bin/sudo
	/usr/bin/chsh
	/usr/bin/write
	/usr/bin/chfn
	/usr/bin/passwd
	/usr/bin/crontab
	/usr/bin/chage
	/usr/bin/gpasswd
	/usr/libexec/openssh/ssh-keysign
	/usr/libexec/pt_chown
	/usr/libexec/utempter/utempter"
	for PARTa in `grep -v ^# /etc/fstab | awk '($6 != "0") {print $2 }'`; do find $PARTa \( -perm -04000 -o -perm -02000 \) -type f -xdev -print; done 2>/dev/null | while read chengxusg
	do
		if [[ $xitongzidai =~ $chengxusg ]];then
			continue
		else
			echo "无法识别且未授权的SUID/SGID文件 $chengxusg"
		fi
	done
	
	wrdocument=$(for PARTb in `awk '($3 == "ext2" || $3 == "ext3") { print $2 }' /etc/fstab`; do find $PARTb -xdev -type d \( -perm -0002 -a ! -perm -1000 \) -print; done)
	if [ -n "$wrdocument" ];then
		echo "存在任何人都有写权限的目录$wrdocument"
	else
		echo "OK,不存在任何人都有写权限的目录"
	fi
	
	wrfile=$(for PARTc in `grep -v ^# /etc/fstab | awk '($6 != "0") {print $2 }'`; do find $PARTc -xdev -type f \( -perm -0002 -a ! -perm -1000 \) -print; done)
	if [ -n "$wrfile" ];then
		echo "存在任何人都有写权限的文件$wrfile"
	else
		echo "OK,不存在任何人都有写权限的文件"
	fi
	
	wushuz=$( for PART in `grep -v ^# /etc/fstab | awk '($6 != "0") {print $2 }'`; do find $PART -nouser -o -nogroup -print; done 2>/dev/null)
	if [ -n "$wushuz" ];then
		echo "存在没有属主的文件$wushuz"
	else
		echo "OK,不存在没有属主的文件"
	fi
	
	defyincang="
	/tmp/.ICE-unix
	/root/.viminfo
	/root/.cshrc
	/root/.tcshrc
	/root/.bash_history
	/root/.bash_profile
	/root/.bash_logout
	/root/.bashrc
	/etc/skel/.bash_profile
	/etc/skel/.bash_logout
	/etc/skel/.bashrc
	/etc/.pwd.lock
	/lib64/.libgcrypt.so.11.hmac
	/lib64/.libfipscheck.so.1.1.0.hmac
	/lib64/.libfipscheck.so.1.hmac
	/usr/lib64/.libcrypto.so.10.hmac
	/usr/lib64/.libcrypto.so.1.0.1e.hmac
	/usr/lib64/.libssl.so.1.0.1e.hmac
	/usr/lib64/.libssl.so.10.hmac
	/usr/sbin/.sshd.hmac
	/usr/bin/.fipscheck.hmac
	/usr/share/man/man1/..1.gz
	/usr/share/man/man5/.k5identity.5.gz
	/usr/share/man/man5/.k5login.5.gz
	/var/lib/rpm/.rpm.lock
	/.autofsck"
	find / -name ".*" -print -xdev 2>/dev/null | while read yingcangd
	do
		if [[ $defyincang =~ $yingcangd ]];then
			continue
		else
			echo "无法识别的隐藏文件 $yingcangd"
		fi
	done
	
	dengluc=$(cat /etc/profile |grep TMOUT)
	if [[ $dengluc =~ "TMOUT=180" ]];then
		echo "OK,登录超时设置正确"
	else
		echo "登录超时设置不正确，目前为$dengluc，建议/etc/profile开头添加TMOUT=180"
	fi
	
	yuancheng=$(service telnet status 2>/dev/null)
	if [[ "unrecognized" =~ $yuancheng ]] && [[ "not running" =~ $yuancheng ]];then
	        echo "OK,telnet未开启或运行"
	else
	        echo "telnet开启，请关闭"
	fi
	
	perolo=$(cat /etc/ssh/sshd_config | egrep -v "^#|^$" | grep PermitRootLogin)
	banbenpro=$(cat /etc/ssh/sshd_config | egrep -v "^#|^$" | grep Protocol)
	if [ -z "$perolo" ] || [[ $perolo =~ "yes" ]];then
		echo "PermitRootLogin不为no，请在/etc/ssh/sshd_config中设置为no"
	else
		echo "OK，已禁止root远程登录"
	fi
	if [ "$banbenpro" = "Protocol 2" ];then
		echo "OK,ssh协议版本正确"
	else
		echo "Protocol不正确，请在/etc/ssh/sshd_config中设置为2"
	fi
	
	dedangqianon="
	auditd
	blk-availability
	crond
	ip6tables
	iptables
	lvm2-monitor
	netfs
	network
	postfix
	rsyslog
	sshd
	udev-post"
	dangqianjb=$(who -r | awk '{print $2}')
	echo -e "当前init级别开启且无法识别的服务请自行识别\n"
	chkconfig --list | grep "$dangqianjb:on" | awk '{print $1}' | while read dangqianon
	do
		if [[ $defdangqianon =~  $dangqianon ]];then
			continue
		else
			echo "$dangqianon"
		fi
	done
	
	syslogzhi=$(cat /etc/syslog.conf 2>/dev/null | egrep -v "^#|^$" | grep authpriv)
	rsyslogzhi=$(cat /etc/rsyslog.conf 2>/dev/null | egrep -v "^#|^$" | grep authpriv)
	if [ -n "$syslogzhi" ] && [[ $syslogzhi =~ "authpriv.*" ]] || [ -n "$rsyslogzhi" ] && [[ $rsyslogzhi =~ "authpriv.*" ]];then
		echo "OK,authpriv值设置正确"
	else
		echo -e "authpriv值不正确，请设置authpriv.*，目前的设置为\n$syslogzhi$rsyslogzhi"
	fi
	
	sysshe=$(cat /etc/syslog.conf 2>/dev/null | egrep -iv "^#|modload" | egrep -i "kern|info|emerg|local")
	rsysshe=$(cat /etc/rsyslog.conf 2>/dev/null | egrep -iv "^#|modload" | egrep -i "kern|info|emerg|local")
	syszhi=$(cat /etc/syslog.conf 2>/dev/null | egrep -iv "^#|modload" | egrep -i "kern|info|emerg|local" | wc -l)
	rsyszhi=$(cat /etc/rsyslog.conf 2>/dev/null | egrep -iv "^#|modload" | egrep -i "kern|info|emerg|local" | wc -l)
	if [ "$syszhi" = 4 ] || [ "$rsyszhi" = 4 ];then
		echo "OK,syslog设置正确"
	else
		echo -e "/etc/syslog.conf设置不正确，请参考安全基线3.2.1节，目前的设置为\n$sysshe$rsysshe"
	fi
	
	socore=$(cat /etc/security/limits.conf | egrep -v "^#" | grep "soft[[:space:]]\+core")
	hacore=$(cat /etc/security/limits.conf | egrep -v "^#" | grep "hard[[:space:]]\+core")
	if [ -n "$socore" ] && [[ $socore =~ "0" ]];then
		echo "OK，soft core设置正确"
	else
		echo "soft core设置不正确，请在/etc/security/limits.conf中设置* soft core 0"
	fi
	if [ -n "$hacore" ] && [[ $hacore =~ "0" ]];then
		echo "OK，hard core设置正确"
	else
		echo "hard core设置不正确，请在/etc/security/limits.conf中设置* hard core 0"
	fi
	



