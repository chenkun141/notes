## Nginx负载均衡 ##
### 参考资料 ###
[https://juejin.im/post/5adc425f518825670f7b6fc8](https://juejin.im/post/5adc425f518825670f7b6fc8)
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

