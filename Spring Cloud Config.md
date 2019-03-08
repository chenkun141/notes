## Spring Cloud Config ##

Spring Cloud Config分为Config Server和Config Client两部分,为分布式系统外部化配置提供了支持.Spring Cloud Config非常适合Spring应用程序,也能与其他变成语言编写的应用组合使用.  
  微服务在启动时,通过Config Client请求Config Server以获取配置内容,同时会缓存这些内容.
### 参考链接 ###
官方网站 : [https://cloud.spring.io/spring-cloud-static/spring-cloud-config/2.0.0.RELEASE/single/spring-cloud-config.html](https://cloud.spring.io/spring-cloud-static/spring-cloud-config/2.0.0.RELEASE/single/spring-cloud-config.html)   
源码分析参考网站 : [https://blog.coding.net/blog/spring-cloud-config](https://blog.coding.net/blog/spring-cloud-config)   
测试demo地址：[https://github.com/chenkun141/spring-cloud-config](https://github.com/chenkun141/spring-cloud-config)  
### Config Server ###
  Config Server是一个集中式,可扩展的配置服务器,它可以集中管理应用程序各个环境下的配置,默认使用git存储配置内容.  
  添加pom依赖文件:

	<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
	
	
在主入口函数,加上注解@EnableConfigServer,开启Spring Cloud Config的服务端功能

	@EnableConfigServer
	@SpringBootApplication
	public class Application {
	    public static void main(String[] args) {
	        SpringApplication.run(Application.class,args);
	    }
	}
修改application.yml文件配置  

- 方式一：使用https访问git仓库
	
		spring:
		  application:
		    name: configServer
		  cloud:
		    config:
		      server:
		        git:
		          uri: https://github.com/chenkun141/spring-cloud-config #gitc仓库位置
		          username: chenkun141@163.com  #访问 Git 仓库的用户名
		          password: **********  #访问 Git 仓库的密码
		          search-paths:  /configServer/src/main/resources/configServer,/configServer/src/main/resources/configClient
		server:
		  port: 8081
		#服务注册中心实例的主机名
		eureka:
		  client:
		    service-url:
		      defaultZone: http://127.0.0.1:8090/eureka
- 方式二： 使用ssh方式访问git仓库

1. 生成公钥和私钥  
参考链接: [https://git-scm.com/book/zh/v2](https://git-scm.com/book/zh/v2)  
2. 将.pub文件内容,设置到git仓库的配置下
3. 获得私钥文件内容,设置到application.yml配置文件

			spring:
			  application:
			   name: configServer
			  cloud:
			    config:
			      server:
			        git:
			          uri: git@github.com:chenkun141/spring-cloud-config.git 
			          ignore-local-ssh-settings: true
			          private-key: | 秘钥内容
					  strict-host-key-checking: true
			          search-paths:  /configServer/src/main/resources/configServer, /configServer/src/main/resources/configClient
			server:
			  port: 8081
			#服务注册中心实例的主机名
			eureka:
			  client:
			    service-url:
			      defaultZone: http://127.0.0.1:8090/eureka

其中git的配置分别表示如下内容

	spring.cloud.config.server.git.uri : 配置的git仓库位置
	spring.cloud.config.server.git.search-paths : 配置仓库路径下的相对搜索位置,可以配置多个
	spring.cloud.config.server.git.username : 访问git的用户名
	spring.cloud.config.server.git.password : 访问git仓库的用户密码
http请求url地址:
	
	/{application}-{profile}.yml
	/{label}/{application}-{profile}.yml
	/{application}-{profile}.properties
	/{label}/{application}-{profile}.properties
上面的url会映射

	{application}通常使用微服务名称,对应git仓库中文件名的前缀
	{profile}对应{application}-后面的dev,pro,test等
	{lable}对应git上不同的分支,默认为master

### Config Client ###
添加pom依赖文件

	<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
	<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
	
创建springBoot主入口函数

	@SpringBootApplication
	public class ConfigClientApplication {
	    public static void main(String[] args) {
	        SpringApplication.run(ConfigClientApplication.class,args);
	    }		
	}

修改bootstrap.yml配置文件.设置服务器地址spring.cloud.config.uri

	spring:
	  cloud:
	    config:
	      uri: http://127.0.0.1:8081/  #config server地址
	      profile: dev
	      label: master
	      discovery:
	        enabled: true #如果需要做服务治理需要设置为true，默认为false
	  application:
	    name: configClient  #server里面search-paths 相关
	server:
	  port: 8082
	#服务注册中心实例的主机名
	eureka:
	  client:
	    service-url:
	      defaultZone: http://127.0.0.1:8090/eureka #注册中心地址
配置参数与git中存储的配置文件各个部分的对应关系:
	
	spring.applicatiopn.name:对应配置文件规则中的{application}部分
	spring.cloud.config.profile:对应配置文件规则中{profile}部分
	spring.cloud.config.lable:对应配置文件规则中的{label}部分
	spring.cloud.config,uri:对应配置中心config-server的地址

#### **注意** ####
resources目录下必须配置文件bootstrap.yml,这样config-server中的配置信息才能被正确加载.springboot对配置文件的加载顺序,对于本应用jar包之外的配置未见加载会又去应用jar包内的配置内容,而通过bootstrap.yml对config-server-git的配置,使得该应用会从config-server-git中获取一些外部配置信息,这些信息的优先级比本地的内容要高,从而实现外部化配置.  
springboot应用配置文件加载顺序是 bootstrap.yml > config server 中的配置 > application.yml中的配置.






