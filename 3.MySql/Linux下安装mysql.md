
## 参考网址 ##
[https://www.jianshu.com/p/276d59cbc529](https://www.jianshu.com/p/276d59cbc529) 

Mysql数据库的安装对于开发者来说，是我们必然会面对的问题，它的安装过程其实并不复杂，并且网络上的安装教程也非常多，但是对于新手来说，各种不同形式的安装教程，又给新手们带来了要选择哪种方式进行安装的难题，而且很多时候按照教程也没有能够安装成功，安装过程出现各种各样的错误。  
下面记录了我在Linux环境下安装Mysql的完整过程，如有错误或遗漏，欢迎指正。

## 一 安装前准备 ##
1、检查是否已经安装过mysql，执行命令

	[root@localhost /]# rpm -qa | grep mysql

从执行结果，可以看出我们已经安装了mysql-libs-5.1.73-5.el6_6.x86_64，执行删除命令

	[root@localhost /]# rpm -e --nodeps mysql-libs-5.1.73-5.el6_6.x86_64
再次执行查询命令，查看是否删除

	[root@localhost /]# rpm -qa | grep mysql

2、查询所有Mysql对应的文件夹

	[root@localhost /]# whereis mysql
	mysql: /usr/bin/mysql /usr/include/mysql
	[root@localhost lib]# find / -name mysql
	/data/mysql
	/data/mysql/mysql
删除相关目录或文件

	[root@localhost /]#  rm -rf /usr/bin/mysql /usr/include/mysql /data/mysql /data/mysql/mysql 
验证是否删除完毕

	[root@localhost /]# whereis mysql
	mysql:
	[root@localhost /]# find / -name mysql
	[root@localhost /]# 
3、检查mysql用户组和用户是否存在，如果没有，则创建

	[root@localhost /]# cat /etc/group | grep mysql
	[root@localhost /]# cat /etc/passwd |grep mysql
	[root@localhost /]# groupadd mysql
	[root@localhost /]# useradd -r -g mysql mysql
	[root@localhost /]# 
4、从官网下载是用于Linux的Mysql安装包

下载命令：

	[root@localhost /]#  wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz

二 安装Mysql
1、在执行wget命令的目录下或你的上传目录下找到Mysql安装包：mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz
执行解压命令：

	[root@localhost /]#  tar xzvf mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz
	[root@localhost /]# ls
	mysql-5.7.24-linux-glibc2.12-x86_64
	mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz
解压完成后，可以看到当前目录下多了一个解压文件，移动该文件到/usr/local/mysql
执行移动命令：

	[root@localhost /]# mv mysql-5.7.24-linux-glibc2.12-x86_64 /usr/local/mysql
2、在/usr/local/mysql目录下创建data目录

	[root@localhost /]# mkdir /usr/local/mysql/data
3、更改mysql目录下所有的目录及文件夹所属的用户组和用户，以及权限

	[root@localhost /]# chown -R mysql:mysql /usr/local/mysql
	[root@localhost /]# chmod -R 755 /usr/local/mysql
4、编译安装并初始化mysql,务必记住初始化输出日志末尾的密码（数据库管理员临时密码）

	[root@localhost /]# cd /usr/local/mysql/bin
	[root@localhost bin]# ./mysqld --initialize --user=mysql --datadir=/usr/local/mysql/data --basedir=/usr/local/mysql
补充说明：
此时可能会出现错误：  
![](https://i.imgur.com/qnR5xZb.png)

出现该问题首先检查该链接库文件有没有安装使用 命令进行核查

	[root@localhost bin]# rpm -qa|grep libaio   
	[root@localhost bin]# 
运行命令后发现系统中无该链接库文件

	[root@localhost bin]#  yum install  libaio-devel.x86_64
安装成功后，继续运行数据库的初始化命令，此时可能会出现如下错误：


执行如下命令后，再次运行数据库的初始化命令：

	[root@localhost bin]#  yum -y install numactl
5、运行初始化命令成功后，输出日志如下：  
![](https://i.imgur.com/lslorN7.png)

记录日志最末尾位置root@localhost:后的字符串，此字符串为mysql管理员临时登录密码。

6、编辑配置文件my.cnf，添加配置如下

	[root@localhost bin]#  vi /etc/my.cnf
	
	[mysqld]
	datadir=/usr/local/mysql/data
	port = 3306
	sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
	symbolic-links=0
	max_connections=400
	innodb_file_per_table=1
	#表名大小写不明感，敏感为
	lower_case_table_names=1
7、启动mysql服务器

	[root@localhost /]# /usr/local/mysql/support-files/mysql.server start
显示如下结果，说明数据库安装成功


如果出现如下提示信息

	Starting MySQL... ERROR! The server quit without updating PID file
查看是否存在mysql和mysqld的服务，如果存在，则结束进程，再重新执行启动命令

	#查询服务
	ps -ef|grep mysql
	ps -ef|grep mysqld
	
	#结束进程
	kill -9 PID
	
	#启动服务
	 /usr/local/mysql/support-files/mysql.server start

8、添加软连接，并重启mysql服务

	[root@localhost /]#  ln -s /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql 
	[root@localhost /]#  ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql
	[root@localhost /]#  service mysql restart
9、登录mysql，修改密码(密码为步骤5生成的临时密码)

	[root@localhost /]#  mysql -u root -p
	Enter password:
	mysql>set password for root@localhost = password('yourpass');

10、开放远程连接

	mysql>use mysql;
	msyql>update user set user.Host='%' where user.User='root';
	mysql>flush privileges;
![](https://i.imgur.com/odV6CRe.png)  

11、设置开机自动启动

	1、将服务文件拷贝到init.d下，并重命名为mysql
	[root@localhost /]# cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
	2、赋予可执行权限
	[root@localhost /]# chmod +x /etc/init.d/mysqld
	3、添加服务
	[root@localhost /]# chkconfig --add mysqld
	4、显示服务列表
	[root@localhost /]# chkconfig --list
