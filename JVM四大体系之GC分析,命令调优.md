# JVM四大体系之GC分析,命令调优 #
## 参考资料 ##
[http://www.ityouknow.com/jvm/2017/09/18/GC-Analysis.html](http://www.ityouknow.com/jvm/2017/09/18/GC-Analysis.html)  
[http://www.ityouknow.com/java.html](http://www.ityouknow.com/java.html)
## jvm体系 ##
jvm体系总体分四大块: 

1. 类的加载机制
2. jvm内存结构
3. GC算法,垃圾回收
4. GC分析,命令调优
## 如何获得Java GC日志 ##
一般情况可以通过两种方式来获取GC日志,一种是使用命令动态查看,一种是在容器中设置相关参数打印GC日志.
### 命令动态查看 ###
Java自动的工具行命令,jstat可以用来动态监控JVM内存的使用,统计垃圾回收的各项信息.
比如常用命令,**jstat -gc** 统计垃圾回收堆的行为

	$ jstat -gc 1262
	 S0C    S1C     S0U     S1U   EC       EU        OC         OU        PC       PU         YGC    YGCT    FGC    FGCT     GCT   
	26112.0 24064.0 6562.5  0.0   564224.0 76274.5   434176.0   388518.3  524288.0 42724.7    320    6.417   1      0.398    6.815

也可以设置间隔固定时间来打印:

	$ jstat -gc 1262 2000 20
这个命令意思就是每隔2000ms输出1262的gc情况，一共输出20次
### GC参数 ###
JVM的GC日志的主要参数包括如下几个：

- -XX:+PrintGC 输出GC日志
- -XX:+PrintGCDetails 输出GC的详细日志
- -XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
- -XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2017-09-04T21:53:59.234+0800）
- -XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息
- -Xloggc:../logs/gc.log 日志文件的输出路径  

在生产环境中，根据需要配置相应的参数来监控JVM运行情况。
### Tomcat 设置示例 ###
我们经常在tomcat的启动参数中添加JVM相关参数，这里有一个典型的示例：

	JAVA_OPTS="-server -Xms2000m -Xmx2000m -Xmn800m -XX:PermSize=64m -XX:MaxPermSize=256m -XX:SurvivorRatio=4
	-verbose:gc -Xloggc:$CATALINA_HOME/logs/gc.log 
	-Djava.awt.headless=true 
	-XX:+PrintGCTimeStamps -XX:+PrintGCDetails 
	-Dsun.rmi.dgc.server.gcInterval=600000 -Dsun.rmi.dgc.client.gcInterval=600000
	-XX:+UseConcMarkSweepGC -XX:MaxTenuringThreshold=15"

根据上面的参数我们来做一下解析：  

- **-Xms2000m -Xmx2000m -Xmn800m -XX:PermSize=64m -XX:MaxPermSize=256m**  
Xms，即为jvm启动时得JVM初始堆大小,Xmx为jvm的最大堆大小，xmn为新生代的大小，permsize为永久代的初始大小，MaxPermSize为永久代的最大空间。

- **-XX:SurvivorRatio=4**  
SurvivorRatio为新生代空间中的Eden区和救助空间Survivor区的大小比值，默认是8，则两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10。调小这个参数将增大survivor区，让对象尽量在survitor区呆长一点，减少进入年老代的对象。去掉救助空间的想法是让大部分不能马上回收的数据尽快进入年老代，加快年老代的回收频率，减少年老代暴涨的可能性，这个是通过将-XX:SurvivorRatio 设置成比较大的值（比如65536)来做到。

- **-verbose:gc -Xloggc:$CATALINA_HOME/logs/gc.log**  
将虚拟机每次垃圾回收的信息写到日志文件中，文件名由file指定，文件格式是平文件，内容和-verbose:gc输出内容相同。

- **-Djava.awt.headless=true**  
Headless模式是系统的一种配置模式。在该模式下，系统缺少了显示设备、键盘或鼠标。

- **-XX:+PrintGCTimeStamps -XX:+PrintGCDetails**  
设置gc日志的格式

- **-Dsun.rmi.dgc.server.gcInterval=600000 -Dsun.rmi.dgc.client.gcInterval=600000**
指定rmi调用时gc的时间间隔

- **-XX:+UseConcMarkSweepGC -XX:MaxTenuringThreshold=15**  
采用并发gc方式，经过15次minor gc 后进入年老代

## 如何分析GC日志 ##
摘录GC日志一部分

Young GC回收日志:

	2016-07-05T10:43:18.093+0800: 25.395: [GC [PSYoungGen: 274931K->10738K(274944K)] 371093K->147186K(450048K), 0.0668480 secs] [Times: user=0.17 sys=0.08, real=0.07 secs]

Full GC回收日志:

	2016-07-05T10:43:18.160+0800: 25.462: [Full GC [PSYoungGen: 10738K->0K(274944K)] [ParOldGen: 136447K->140379K(302592K)] 147186K->140379K(577536K) [PSPermGen: 85411K->85376K(171008K)], 0.6763541 secs] [Times: user=1.75 sys=0.02, real=0.68 secs]

通过上面日志分析得出，PSYoungGen、ParOldGen、PSPermGen属于Parallel收集器。其中PSYoungGen表示gc回收前后年轻代的内存变化；ParOldGen表示gc回收前后老年代的内存变化；PSPermGen表示gc回收前后永久区的内存变化。young gc 主要是针对年轻代进行内存回收比较频繁，耗时短；full gc 会对整个堆内存进行回城，耗时长，因此一般尽量减少full gc的次数

通过两张图非常明显看出gc日志构成：  
Young GC日志:  
![](https://i.imgur.com/k7jPBpc.png)  
Full GC日志:  
![](https://i.imgur.com/jcFFm43.png)  
## GC分析工具 ##
### GC Easy ###
这是一个web工具,在线使用非常方便.  
地址: [http://gceasy.io](http://gceasy.io)  
进入官网，讲打包好的zip或者gz为后缀的压缩包上传，过一会就会拿到分析结果.  
![](https://i.imgur.com/6bd8lci.png)  
![](https://i.imgur.com/FwhwgJB.png)

## JDK监控和故障处理命令 ##
参考链接: [http://www.ityouknow.com/jvm/2017/09/03/jvm-command.html](http://www.ityouknow.com/jvm/2017/09/03/jvm-command.html)