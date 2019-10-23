### 线程局部变量ThreadLocal ###
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
