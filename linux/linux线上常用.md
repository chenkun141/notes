## Linux 上查看java最耗时的线程信息 ##
1. 找到JAVA进程pid  

		ps -ef|grep java或则jps -mlv

在这里插入图片描述  
2. 找进程下耗时的线程TID  
使用top -Hp pid可以查看某个进程的线程信息 -H 显示线程信息，-p指定pid
top -Hp 10906 查看最耗时的 TID即线程id

在这里插入图片描述  
3. printf "%x\n" [tid] 转成16进制

在这里插入图片描述

4. java中的线程类相关信息
jstack 线程ID 可以查看某个线程的堆栈情况，特别对于hung挂死的线程
jstack [pid] | grep [tid] 特定线程信息
jstack -l [pid] 列出所有线程信息。
	
在这里插入图片描述  

5. jstack
jstack是java虚拟机自带的一种堆栈跟踪工具。用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。

PS :
在实际运行中，往往一次 dump的信息，还不足以确认问题。建议产生三次 dump信息，如果每次 dump都指向同一个问题，我们才确定问题的典型性。也就是多进行几次线程快照，观察变化，查看问题所在。


## Linux 线上日志查看 ##
1. 查看关键字左右的日志信息
	- 看上下10行：grep -C 10 'NullPointerException' logback.log
	- 看上面10行：grep -B 10 'NullPointerException' logback.log
	- 看下面10行：grep -A 10 'NullPointerException' logback.log


## Linux 开放端口 ##
1. 查看防火墙开放端口  
	more /etc/sysconfig/iptables
2. 添加端口  
	-A INPUT -p tcp -m state --state NEW -m tcp --dport 9301 -j ACCEPT
3. 保存  
	service iptables save
4. 重启防火墙  
	service iptables restart
5. 查看防火墙状态  
	systemctl status iptables.service 
## linux设置运行脚本 ##
1. 创建脚本启动文件  
	touch test.sh 

2. 编辑脚本信息  
	vi test.sh
 
3. 授权脚本  
	chmod +x test.sh

4. 启动脚本  
	./test.sh