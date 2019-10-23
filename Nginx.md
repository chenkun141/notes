# Nginx #
## 参考资料 ##
[https://juejin.im/post/5adc425f518825670f7b6fc8](https://juejin.im/post/5adc425f518825670f7b6fc8)
[https://mp.weixin.qq.com/s/W4pjMCZ2bNRAw2EOMiDJqQ](https://mp.weixin.qq.com/s/W4pjMCZ2bNRAw2EOMiDJqQ)
Nginx(发音同engine x)是一步框架的网页服务器,也可以用作反向代理,负载平衡和HTTP缓存
## Nginx安装 ##
### 下载Nginx安装包 ###
直接访问Nginx官网(https://nginx.org),下载对应的安装包,本次案例选择的是nginx-1.6.3.tar.gz版本，安装环境是centos7。
上传到对应服务器的文件夹或者直接在服务器使用wget命令

	#下载nginx-1.6.3.tar.gz
	wget -c https://nginx.org/download/nginx-1.6.3.tar.gz
如果出现如下信息：

	-bash: wget: command not found
	
提示wget命令找不到，使用如下命令，进行安装，之后再次执行上述下载命令	

	yum install wget

### 安装Nginx ###
在安装Nginx之前,需要安装相应运行库环境,操作如下 
 
1. 安装 gcc 环境

		yum install gcc-c++
2. 安装 PCRE 依赖库

		yum install -y pcre pcre-devel
3. 安装 zlib 依赖库

		yum install -y zlib zlib-devel
4. 安装 OpenSSL  
		
		安全套接字层密码库
		yum install -y openssl openssl-devel
5. 解压 Nginx


		安装完以上环境库之后，接着进行解压操作
		#解压文件夹
		tar -zxvf nginx-1.6.3.tar.gz

6. 执行配置命令  
	
		cd进入文件夹
		cd nginx-1.6.3
		
		执行配置命令
		./configure
	如下图，表示执行配置成功！

	![](https://i.imgur.com/f0yX9RX.jpg)

	当然,也可以执行自定义配置文件,例如:
	
		./configure \
		--prefix=/usr/local/nginx \
		--conf-path=/usr/local/nginx/conf/nginx.conf \
		--pid-path=/usr/local/nginx/conf/nginx.pid \
		--lock-path=/var/lock/nginx.lock \
		--error-log-path=/var/log/nginx/error.log \
		--http-log-path=/var/log/nginx/access.log \
		--with-http_gzip_static_module \
		--http-client-body-temp-path=/var/temp/nginx/client \
		--http-proxy-temp-path=/var/temp/nginx/proxy \
		--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
		--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
		--http-scgi-temp-path=/var/temp/nginx/scgi

	> 注意：临时文件目录指定为/var/temp/nginx，需要在/var下创建temp及nginx目录

7. 执行编译安装命令  

		make install
8. 查找安装路径

		whereis nginx结果如下：

	![](https://i.imgur.com/fqnFyJC.jpg)  
9. 启动服务 

	进入 nginx 的目录

		cd /usr/local/nginx/sbin/ 
	![](https://i.imgur.com/dvnJUPr.jpg)  

	执行如下命令
	
		#启动
		./nginx
		
		#停止，此方式相当于先查出nginx进程id再使用kill命令强制杀掉进程
		./nginx -s stop
		
		#停止，此方式停止步骤是待nginx进程处理任务完毕进行停止
		./nginx -s quit
		
		#重新加载配置文件，Nginx服务不会中断
		./nginx -s reload

10. 修改配置文件  

	比如，修改端口号，默认端口号为80，咱们这里改成81；
	
	进入配置文件夹
	
		cd /usr/local/nginx/conf
	备份原始配置文件
	
		cp nginx.conf nginx.conf.back
	编辑nginx.conf配置文件
	
		vim nginx.conf

	![](https://i.imgur.com/X9X6MF0.jpg)  
	找到server中的listen，修改端口号为81  
	![](https://i.imgur.com/WKmL9ZY.jpg)  

	启动服务
	
		./nginx
	查看 nginx 进程

		ps -ef|grep nginx
	![](https://i.imgur.com/AIVHDH1.jpg)  
	到此，nginx 安装基本完成，直接在浏览器上访问服务器地址ip:81，就可以进入页面  
	![](https://i.imgur.com/h4dCxc1.jpg)


## Nginx负载均衡 ##
### 反向代理 ###
假设我现在需要本地访问www.baidu.com;配置如下：

	server {
    #监听80端口
    listen 80;
    server_name localhost;
     # individual nginx logs for this web vhost
    access_log /tmp/access.log;
    error_log  /tmp/error.log ;

    location / {
        proxy_pass http://www.baidu.com;
    }

### 负载均衡 ###
下面主要验证最常用的三种负载策略。虚拟主机配置：

	server {
	    #监听80端口
	    listen 80;
	    server_name localhost;
	    
	    # individual nginx logs for this web vhost
	    access_log /tmp/access.log;
	    error_log  /tmp/error.log ;
	
	    location / {
	        #负载均衡
	        #轮询 
	        #proxy_pass http://polling_strategy;
	        #weight权重
	        #proxy_pass http://weight_strategy;
	        #ip_hash
	        # proxy_pass http://ip_hash_strategy;
	        #fair
	        # proxy_pass http://fair_strategy;
	        #url_hash
	        # proxy_pass http://url_hash_strategy;
	        #重定向
	        #rewrite ^ http://localhost:8080;
	    }

1. 轮询策略

		# 1、轮询（默认）
		# 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。 
		upstream polling_strategy { 
		    server glmapper.net:8080; # 应用服务器1
		    server glmapper.net:8081; # 应用服务器2
		} 
测试结果（通过端口号来区分当前访问）：

		8081：hello
		8080：hello
		8081：hello
		8080：hello

2. 权重策略

		#2、指定权重
		#指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。 
		upstream  weight_strategy { 
		    server glmapper.net:8080 weight=1; # 应用服务器1
		    server glmapper.net:8081 weight=9; # 应用服务器2
		}

	测试结果：总访问次数15次，根据上面的权重配置，两台机器的访问比重：2：13；满足预期  
3. ip hash策略

		#3、IP绑定 ip_hash
		#每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，
		#可以解决session的问题;在不考虑引入分布式session的情况下，
		#原生HttpSession只对当前servlet容器的上下文环境有效
		upstream ip_hash_strategy { 
		    ip_hash; 
		    server glmapper.net:8080; # 应用服务器1
		    server glmapper.net:8081; # 应用服务器2
		} 

iphash 算法:ip是基本的点分十进制，将ip的前三个端作为参数加入hash函数。这样做的目的是保证ip地址前三位相同的用户经过hash计算将分配到相同的后端server。作者的这个考虑是极为可取的，因此ip地址前三位相同通常意味着来着同一个局域网或者相邻区域，使用相同的后端服务让nginx在一定程度上更具有一致性。

为什么说要解释下iphash,因为采坑了；和猪弟在进行这个策略测试时使用了5台机器来测试的，5台机器均在同一个局域网内【192.168.3.X】;测试时发现5台机器每次都路由到了同一个服务器上，一开始以为是配置问题，但是排查之后也排除了这个可能性。最后考虑到可能是对于同网段的ip做了特殊处理，验证之后确认了猜测  
4. 其他负载均衡策略    
这里因为需要安装三方插件，时间有限就不验证了，知悉即可！  

		#4、fair（第三方）
		#按后端服务器的响应时间来分配请求，响应时间短的优先分配。 
		upstream fair_strategy { 
		    server glmapper.net:8080; # 应用服务器1
		    server glmapper.net:8081; # 应用服务器2
		    fair; 
		} 
		
		#5、url_hash（第三方）
		#按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，
		#后端服务器为缓存时比较有效。 
		upstream url_hash_strategy { 
		    server glmapper.net:8080; # 应用服务器1
		    server glmapper.net:8081; # 应用服务器2 
		    hash $request_uri; 
		    hash_method crc32; 
		} 


### Nginx 开启gzip来提高页面加载速度 ###
#### 参考资料 ####
[https://blog.csdn.net/bigtree_3721/article/details/79849503](https://blog.csdn.net/bigtree_3721/article/details/79849503)

#### 默认配置 ####

	gzip on;	#开启Gzip
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_comp_level 5;	
    gzip_http_version 1.1;
    gzip_types text/plain application/x-javascript text/css text/htm application/xml;
    include defaults/*.conf;
    include extra/*.conf;
