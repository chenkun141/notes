## 参考资料 ##
项目嵌入jetty服务器[https://www.jianshu.com/p/b88de39d00e9](https://www.jianshu.com/p/b88de39d00e9)  
jetty介绍[https://www.jianshu.com/p/401e9e0ef8fa](https://www.jianshu.com/p/401e9e0ef8fa)  
官网地址[http://www.eclipse.org/jetty/index.html](http://www.eclipse.org/jetty/index.html)
## 简介 ##
Tomcat和Jetty都是一种Servlet引擎，他们都支持标准的servlet规范和JavaEE的规范。由于同样遵循servlet2.5和jdk1.5的标准，jetty和tomcat的使用针对项目是透明（没有差异）的。
### 目录结构 ###

	bin：可执行脚本文件
	demo- base：
	etc：Jetty模块定义的XML配置文件的目录
	lib：Jetty依赖的库文件
	logs：Jetty的日志目录
	modules：Jetty的模块
	resources：外部资源配置文件的目录
	webapps：项目WAR文件的目录还需要关心根目录下的一个文件：start.d（Wondows系统是start.ini文件），它定义了Jetty的活动模块。
### 基本配置 ###
- 修改jetty端口  
	编辑start.d（Windows系统是start.ini文件），找到jetty.http.port行，修改为：jetty.http.port=7070.
	保存并退出，再重启Jetty.
- 修改webapps目录  
	修改start.d配置文件（Windows系统是start.ini文件），找到jetty.deploy.monitoredDir=webapps,并把内容修改到你指定的目录。保存并退出，再重启Jetty。

### Jetty和Tomcat简单对比 ###

1. 架构比较  
Jetty的架构比Tomcat的更为简单  
Jetty的架构是基于Handler来实现的，主要的扩展功能都可以用Handler来实现，扩展简单。  
Tomcat的架构是基于容器设计的，进行扩展是需要了解Tomcat的整体设计结构，不易扩展。
2. 性能比较  
Jetty和Tomcat性能方面差异不大  
Jetty可以同时处理大量连接而且可以长时间保持连接，适合于web聊天应用等等。  
Jetty的架构简单，因此作为服务器，Jetty可以按需加载组件，减少不需要的组件，减少了服务器内存开销，从而提高服务器性能。  
Jetty默认采用NIO结束在处理I/O请求上更占优势，在处理静态资源时，性能较高  
Tomcat适合处理少数非常繁忙的链接，也就是说链接生命周期短的话，Tomcat的总体性能更高。  
Tomcat默认采用BIO处理I/O请求，在处理静态资源时，性能较差。  
3. 其它比较  
Jetty的应用更加快速，修改简单，对新的Servlet规范的支持较好。  
Tomcat目前应用比较广泛，对JavaEE和Servlet的支持更加全面，很多特性会直接集成进来。  
jetty结合maven使用  


## Jetty嵌入式启动 ##
### API方式 ###
添加maven依赖
	
	<dependency>
	  <groupId>org.eclipse.jetty</groupId>
	  <artifactId>jetty-webapp</artifactId>
	  <version>9.3.2.v20150730</version>
	  <scope>test</scope>
	</dependency>
	<dependency>
	  <groupId>org.eclipse.jetty</groupId>
	  <artifactId>jetty-annotations</artifactId>
	  <version>9.3.2.v20150730</version>
	  <scope>test</scope>
	</dependency>
	<dependency>
	  <groupId>org.eclipse.jetty</groupId>
	  <artifactId>apache-jsp</artifactId>
	  <version>9.3.2.v20150730</version>
	  <scope>test</scope>
	</dependency>
	<dependency>
	  <groupId>org.eclipse.jetty</groupId>
	  <artifactId>apache-jstl</artifactId>
	  <version>9.3.2.v20150730</version>
	  <scope>test</scope>
	</dependency>

官方启动代码
	
	public class SplitFileServer
		{
		    public static void main( String[] args ) throws Exception
		    {
		        // 创建Server对象，并绑定端口
		        Server server = new Server();
		        ServerConnector connector = new ServerConnector(server);
		        connector.setPort(8090);
		        server.setConnectors(new Connector[] { connector });
		
		        // 创建上下文句柄，绑定上下文路径。这样启动后的url就会是:http://host:port/context
		        ResourceHandler rh0 = new ResourceHandler();
		        ContextHandler context0 = new ContextHandler();
		        context0.setContextPath("/");
		      
		        // 绑定测试资源目录（在本例的配置目录dir0的路径是src/test/resources/dir0）
		        File dir0 = MavenTestingUtils.getTestResourceDir("dir0");
		        context0.setBaseResource(Resource.newResource(dir0));
		        context0.setHandler(rh0);
		
		        // 和上面的例子一样
		        ResourceHandler rh1 = new ResourceHandler();
		        ContextHandler context1 = new ContextHandler();
		        context1.setContextPath("/");
		        File dir1 = MavenTestingUtils.getTestResourceDir("dir1");
		        context1.setBaseResource(Resource.newResource(dir1));
		        context1.setHandler(rh1);
		
		        // 绑定两个资源句柄
		        ContextHandlerCollection contexts = new ContextHandlerCollection();
		        contexts.setHandlers(new Handler[] { context0, context1 });
		        server.setHandler(contexts);
		
		        // 启动
		        server.start();
		
		        // 打印dump时的信息
		        System.out.println(server.dump());
		
		        // join当前线程
		        server.join();
		    }
		}

直接运行Main方法，就可以启动web服务。  
