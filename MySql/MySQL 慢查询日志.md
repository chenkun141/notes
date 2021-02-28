## Mysql 日志 ##
1. Mysql日志种类
	- 重做日志（redo log）
	- 回滚日志（undo log）
	- 二进制日志（binlog）
	- 中继日志（relay log）
	- 错误日志（errorlog）
	- 慢查询日志（slow query log）
	- 一般查询日志（general log）

## Mysql 日志作用 ##
### 重做日志（redo log） ###
1. 作用:  
　　保持了事务的持久性， 采用循环写的方式将写数据，写入方式请参考 [https://www.cnblogs.com/zhixinSHOU/p/13214933.html](https://www.cnblogs.com/zhixinSHOU/p/13214933.html)
2. 内容:  
　　物理格式的日志，记录的是物理数据页面的修改的信息，其redo log是顺序写入redo log file的物理文件中去的。

3. 开始节点(什么时候产生):  
　　事务开始之后就产生redo log，redo log的落盘并不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入redo log文件中。

4. 结束节点(什么时候释放):  
　　当对应事务的脏页写入到磁盘之后，redo log的使命也就完成了，重做日志占用的空间就可以重用（被覆盖）。

### 回滚日志（undo log） ###
1. 作用:  
　　保证了事务的原子性，当事务开始回滚的时候会用到

2. 内容：  
　　逻辑格式的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于redo log的。

3. 开始节点(什么时候产生):  
　　事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性

4. 结束节点(什么时候释放):  
　　当事务提交之后，undo log并不能立马被删除，而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间。

### 二进制日志（binlog) ###
1. 作用:  
　　用于复制，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步。  
　　用于数据库的基于时间点的还原。

2. 内容：  
　　逻辑格式的日志，可以简单认为就是执行过的事务中的sql语句。  
　　但又不完全是sql语句这么简单，而是包括了执行的sql语句（增删改）反向的信息，也就意味着delete对应着delete本身和其反向的insert；update对应着update执行前后的版本的信息；insert对应着delete和insert本身的信息。  
　　在使用mysqlbinlog解析binlog之后一些都会真相大白。  
　　因此可以基于binlog做到类似于oracle的闪回功能，其实都是依赖于binlog中的日志记录。

3. 开始节点(什么时候产生):  
　　事务提交的时候，一次性将事务中的sql语句（一个事物可能对应多个sql语句）按照一定的格式记录到binlog中。  
　　这里与redo log很明显的差异就是redo log并不一定是在事务提交的时候刷新到磁盘，redo log是在事务开始之后就开始逐步写入磁盘。  
　　因此对于事务的提交，即便是较大的事务，提交（commit）都是很快的，但是在开启了bin_log的情况下，对于较大事务的提交，可能会变得比较慢一些。  
　　这是因为binlog是在事务提交的时候一次性写入的造成的，这些可以通过测试验证。  

4. 结束节点(什么时候释放):   
　　binlog的默认是保持时间由参数expire_logs_days配置，也就是说对于非活动的日志文件，在生成时间超过expire_logs_days配置的天数之后，会被自动删除。

## MySQL 慢查询日志 ##

1. 查看是否开启慢查询功能：

		mysql> show variables like 'slow_query%';
		+---------------------+------------------------------------+
		| Variable_name       | Value                              |
		+---------------------+------------------------------------+
		| slow_query_log      | OFF                                |
		| slow_query_log_file | /var/lib/mysql/instance-1-slow.log |
		+---------------------+------------------------------------+
		2 rows in set (0.01 sec)
		

		mysql> show variables like 'long_query_time';
		+-----------------+-----------+
		| Variable_name   | Value     |
		+-----------------+-----------+
		| long_query_time | 10.000000 |
		+-----------------+-----------+
		1 row in set (0.00 sec)
说明：
	- slow_query_log 慢查询开启状态
	- slow_query_log_file 慢查询日志存放的位置（这个目录需要MySQL的运行帐号的可写权限，一般设置为MySQL的数据存放目录）
	- long_query_time 查询超过多少秒才记录

2. 配置
	- 临时配置  
	 默认没有开启慢查询日志记录，通过命令临时开启：
	
			mysql> set global slow_query_log='ON';
			Query OK, 0 rows affected (0.00 sec)
			 
			mysql> set global slow_query_log_file='/var/lib/mysql/instance-1-slow.log';
			Query OK, 0 rows affected (0.00 sec)
			 
			mysql> set global long_query_time=2;
			Query OK, 0 rows affected (0.00 sec)
	
	- 永久配置
	修改配置文件达到永久配置状态：
	
			/etc/mysql/conf.d/mysql.cnf
			[mysqld]
			slow_query_log = ON
			slow_query_log_file = /var/lib/mysql/instance-1-slow.log
			long_query_time = 2
	配置好后，重新启动 MySQL 即可。

3. 测试
通过运行下面的命令，达到问题 SQL 语句的执行：

		mysql> select sleep(2);
		+----------+
		| sleep(2) |
		+----------+
		|        0 |
		+----------+
		1 row in set (2.00 sec)
然后查看慢查询日志内容：

		$ cat /var/lib/mysql/instance-1-slow.log
		/usr/sbin/mysqld, Version: 8.0.13 (MySQL Community Server - GPL). started with:
		Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
		Time                 Id Command    Argument
		/usr/sbin/mysqld, Version: 8.0.13 (MySQL Community Server - GPL). started with:
		Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
		Time                 Id Command    Argument
		# Time: 2018-12-18T05:55:15.941477Z
		# User@Host: root[root] @ localhost []  Id:    53
		# Query_time: 2.000479  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
		SET timestamp=1545112515;
		select sleep(2);