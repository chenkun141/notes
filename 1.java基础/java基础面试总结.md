## CyclicBarrier、CountDownLatch、Semaphore的用法 ##
### CountDownLatch（线程计数器） ###
CountDownLatch类位于java.util.concurrent包下，利用它可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了。
  
		final CountDownLatch latch = new CountDownLatch(2);
        new Thread(){public void run() {
            System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
            Thread.sleep(3000);
            System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
            latch.countDown();
        };}.start();
        new Thread(){ public void run() {
            System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
            Thread.sleep(3000);
            System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
            latch.countDown();
        };}.start();
        System.out.println("等待2个子线程执行完毕...");
        latch.await();
        System.out.println("2个子线程已经执行完毕");
        System.out.println("继续执行主线程");
### CyclicBarrier(回环栅栏-等待至barrier状态在全部同时执行) ###
通过它可以实现让一组线程等待至某个状态之后在全部同时执行,叫做回环是因为当所有等待线程都被释放以后,CyclicBarrier可以被重用,我们暂且把这个状态叫做barrier,当调用await()方法之后,线程就处于barrier了.
1. 概述  
	CyclicBarrier是一个同步工具类，它允许一组线程互相等待，直到到达某个公共屏障点。与CountDownLatch不同的是该barrier在释放等待线程后可以重用，所以称它为循环（Cyclic）的屏障(Barrier).  
	CyclicBarrier支持一个可选的Runnable命令，在一组线程中的最后一个线程到达之后（但在释放所有线程之前），该命令只在每个屏障点运行一次。若在继续所有参与线程之前更新共享状态，此屏障操作很有用。
2. 实现原理  
	基于ReentrantLock和Condition机制实现。除了getParties()方法，CyclicBarrier的其他方法都需要获取锁。
### Semaphore（信号量-控制同时访问的线程个数） ###
Semaphore翻译成字面意思为信号量，Semaphore可以控制同时访问的线程个数，通过acquire()获取一个许可，如果没有就等待，而release()释放一个许可。
### 差异 ###
- CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同；CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后，它才执行；而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；另外，CountDownLatch是不能够重用的，而CyclicBarrier是可以重用的。
- Semaphore其实和锁有点类似，它一般用于控制对某组资源的访问权限。

### Runable 和 Callable ###
	
	package java.util.concurrent;	

	public interface Callable<V> {
		   
		    V call() throws Exception;
		}


	package java.lang;

	public interface Runnable {
	   
	    public abstract void run();
	}
	
1. Runnable是自从java1.1就有了，而Callable是1.5之后才加上去的
2. Callable规定的方法是call(),Runnable规定的方法是run()
3. Callable的任务执行后可返回值，而Runnable的任务是不能返回值(是void)
4. call方法可以抛出异常，run方法不可以
5. 运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。
6. 加入线程池运行，Runnable使用ExecutorService的execute方法，Callable使用submit方法。

## 线程局部变量ThreadLocal ##
- ThreadLocal的作用和目的: 用于实现线程内数据共享,即对于相同的程序代码,多个模块在同一个线程中运行时要共享一份数据,而在另外线程中运行时又共享另外一份你数据.
- 每个线程调用全局ThreadLocal对象的set方法,在set方法中,首先根据当前线程获取当前线程的ThreadLocalMap对象,然后往这个Map中插入一条记录,key其实是ThreadLocal对象,value是各自的set方法传进去的值.也就是每个线程其实都有一份自己独享的ThreadLocalMap对象,这样会更快释放内存,不调用也可以,因为线程结束后也可以自动释放相关的ThreadLocal变量.
- ThreadLocal的应用场景:
	1. 订单处理: 减库存,增加流水台账,这几个操作要在同一个事务中完成,通常也即同一个线程中进行处理.
	2. 银行转账: 转出啊账户的余额减少,转入账户的余额增加,这个操作在同一事务中完成.
- ThreadLocal的使用方式
	1. 在关联数据类中创建 private static ThreadLocal
		在下面类中,私有静态ThreadLocal实例(serialNum)为调用该类的静态SerialNum.get()方法的每个线程维护了一个"序列号",该方法将返回当前线程的序列号.(线程序列号是在第一次调用SerialNum.get()时分配的,并在后续调用中不会更改.)

			public class SerialNum { 
				// The next serial number to be assigned 
				private static int nextSerialNum = 0; 
				private static ThreadLocal serialNum = new ThreadLocal() { 
					protected synchronized Object initialValue() { 
						return new Integer(nextSerialNum++); 
					} 
				}; 
				
				public static int get() { 
					return ((Integer) (ser ialNum.get())).intValue(); 
				} 
			}
		另一个例子,也是私有静态ThreadLocal实例:
		
			public class ThreadContext { 
				private String userId; 
				private Long transactionId; 
				private static ThreadLocal threadLocal = new ThreadLocal(){ 
					@Override 
					protected ThreadContext initialValue() { 
						return new ThreadContext(); 
					} 
				}; 
				public static ThreadContext get() { 
					return threadLocal.get(); 
				} 
				public String getUserId() { 
					return userId; 
				} 
				public void setUserId(String userId) { 
					this.userId = userId;
				} 
				public Long getTransactionId() { 
					return transactionId;
				} 
				public void setTransactionId(Long transactionId) { 
					this.transactionId = transactionId; 
				} 
			}


## 强引用，软引用，弱引用和虚引用的区别与用法 ##
1. 强引用  
当内存空间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题。
	- 场景 Object o=new Object();
2. 软引用  
如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。
3. 弱引用  
弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。
4. 虚引用  
和没有任何引用一样，在任何时候都可能被垃圾回收。虚引用主要用来跟踪对象被垃圾回收的活动

## 双亲委派机制 ##
1. 原理
	1. 一个类加载器收到类加载请求,它不会自己先加载,而是把这个请求委托给父类的加载器去执行
	2. 如果父类加载器还存在其父类加载器,则进一步向上委托,依次递归,请求最终将到达顶层的引导类加载器;
	3. 如果父类加载器可以完成类加载任务,就成功返回,倘若父类加载器无法完成加载任务,子加载器才会尝试自己去加载,这就是双亲委派机制
	4. 父类加载器一层一层往下分配任务,如果自雷加载器能加载,则加载此类,如果将加载任务分配至系统类加载器也无法加载此类,则抛出异常
2. 优势
	1. 避免类的重复加载
	2. 保护程序安全,防止核心api被随意篡改
		1. 自定义类: java.lang.String(没用)
		2. 自定义类: java.lang.ShkStart(报错,阻止创建java.lang开头的类)


## 内存溢出(OOM) 和 内存泄漏(Memory Leak) ##
1. 内存溢出
	- 解释: 系统已经不能再分配出你所需要的空间，比如你需要100M的空间，系统只剩90M了，这就叫内存溢出
2. 内存泄漏
	- 常发性内存泄漏:发生内存泄漏的代码会被多次执行到,每次被执行的时候都会导致一块内存泄漏
	- 偶发性内存泄漏:发生内存泄漏的代码只有在某些特定环境或操作过程下才会发生.常发性和偶发性是相对的.对于特定的环境,偶发性的也许就变成了常发性的.所以测试环境和测试方法对检测内存泄漏至关重要
	- 一次性内存泄漏:发生内存泄漏的代码只会被执行一次，或者由于算法上的缺陷，导致总会有一块仅且一块内存发生泄漏。比如，在类的构造函数中分配内存，在构造函数中却没有释放该内存，所以内存泄漏只会发生一次
	- 隐式内存泄漏:程序在运行过程中不停的分配内存，但是直到结束的时候才释放内存。严格的说这里并没有发生内存泄漏，因为最终程序释放了所有申请的内存。但是对于一个服务器程序，需要运行几天，几周甚至几个月，不及时释放内存也可能导致最终耗尽系统的所有内存。所以，我们称这类内存泄漏为隐式内存泄漏
3. 可能出现内存泄漏的代码
	- 检查对数据库查询中，是否有一次获得全部数据的查询。一般来说，如果一次取十万条记录到内存，就可能引起内存溢出。这个问题比较隐蔽，在上线前，数据库中数据较少，不容易出问题，上线后，数据库中数据多了，一次查询就有可能引起内存溢出。因此对于数据库查询尽量采用分页的方式查询。
	- 检查代码中是否有死循环或递归调用
	- 检查是否有大循环重复产生新对象实体
	- 检查对数据库查询中，是否有一次获得全部数据的查询。一般来说，如果一次取十万条记录到内存，就可能引起内存溢出。这个问题比较隐蔽，在上线前，数据库中数据较少，不容易出问题，上线后，数据库中数据多了，一次查询就有可能引起内存溢出。因此对于数据库查询尽量采用分页的方式查询
	- 检查List、MAP等集合对象是否有使用完后，未清除的问题。List、MAP等集合对象会始终存有对对象的引用，使得这些对象不能被GC回收。




## 集合 ##
### Java集合框架的基本接口有哪些? ###
![](https://i.imgur.com/g3kRVUi.png) 
Collection : 代表一组对象,每一个对象都是它的子元素.  
Set : 不包含重复元素的Collection.  
List : 有顺序的collection,并且可以包含重复元素.  
Map : 可以把键(key)映射到值(value)的对象,键不能重复.

### Arraylist 与 LinkedList 区别 ###
- Arraylist：
 - 优点：ArrayList是实现了基于动态数组的数据结构,因为地址连续，一旦数据存储好了，查询操作效率会比较高（在内存里是连着放的）。
 - 缺点：因为地址连续， ArrayList要移动数据,所以插入和删除操作效率比较低。  
- LinkedList：
 - 优点：LinkedList基于链表的数据结构,地址是任意的，所以在开辟内存空间的时候不需要等一个连续的地址，对于新增和删除操作add和remove，LinedList比较占优势。LinkedList 适用于要头尾操作或插入指定位置的场景
 - 缺点：因为LinkedList要移动指针,所以查询操作性能比较低。
- 适用场景分析：
 - 当需要对数据进行对此访问的情况下选用ArrayList，当需要对数据进行多次增加删除修改时采用LinkedList。