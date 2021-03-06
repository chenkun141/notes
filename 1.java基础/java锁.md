## 乐观锁 ##
乐观锁是一种乐观思想,认为**读多写少**,遇到并发写的可能性低,每次去拿数据的时候都认为别人不会修改,所以不会上锁,但是**在更新的时候会判断一下在此期间别人有没有去更新这个数据,采取在写时先独处当前版本号,然后加锁操作**(比较跟上一次的版本号,如果一样则更新),如果失败则要重复读-比较-写的操作.  
java中的乐观锁基本都是通过cas操作实现的,cas是一种更新的原子操作,比较当前值跟传入值是否一样,一样则更新,否则失败.
## 悲观锁 ##
悲观锁是就是悲观思想，即认为写多，遇到并发写的可能性高，每次去拿数据的时候都认为别人会修改，所以每次在读写数据的时候都会上锁，这样别人想读写这个数据就会block直到拿到锁。java中的悲观锁就是Synchronized,AQS框架下的锁则是先尝试cas乐观锁去获取锁，获取不到，才会转换为悲观锁，如ReentrantLock。

## 非公平锁 ##
jvm按随机,就近原则分配锁的机制则成为不公平锁,ReentrantLock在构造函数中提供了是否公平锁的初始化方式,默认为非公平锁.非公平锁事假执行的效率要远远超出公平锁,除非程序有特殊需要,否则最常用非公平锁的分配机制.

## 公平锁 ##
公平锁指的是锁的分配机制是公平的,通常先对锁提出获取请求的线程会仙贝分配到锁,ReentrantLock在构造函数中提供了是否公平锁的初始化方式来定义公平锁.

## ReentrantLock 与synchronized ##
1. ReentrantLock通过方法lock() 与unlock()来进行加锁与解锁操作,与**synchronized会被jvm自动解锁机制不同,ReentrantLock加锁后需要手动进行解锁**.为了避免程序出现异常而无法正常解锁的情况,使用ReentrantLock必须在finally控制块中进行解锁操作.
2. ReentrantLock相比synchronized的优势是可中断,公平锁,多个锁.这种情况下需要使用ReentrantLock.

## synchronized和ReentrantLock的区别 ##
### Synchronized同步锁 ###
synchronized它可以把任意一个非NULL的对象当作锁。他属于独占式的悲观锁，同时属于可重入锁。
### ReentrantLock ###
ReentrantLock继承接口Lock并实现了接口中定义的方法,他是一种可重入锁,除了能完成synchronized所能完成的所有工作外,还提供了诸如可响应中断锁,可轮询锁请求,定时锁等避免多线程死锁的方法.  
lock接口的主要方法:
1. void lock() : 执行此方法时,如果锁处于空闲状态,当前线程将获得到锁.相反,如果锁已近被其他线程持有,将禁用当前线程,直到当前线程获取到锁.
2. boolean tryLock（）：如果锁可用,则获取锁,并立即返回true,否则返回false.该方法和lock()的区别在于,tryLock()只是"试图"获取锁,不会导致当前线程被禁用,当前线程仍然继续往下执行代码.而lock()方法则是一定要获取到锁,如果锁不可用,就一直等待,在未获得锁之前,当前线程并不继续向下执行.
3. void unlock():执行此方法时,当前线程将释放持有的锁.锁只能由持有者释放,如果线程并不持有锁,却执行该方法,可能导致异常的发生.
4. Condition newCondition():条件对象,获取等待通知组件.该组件和当前的锁绑定,当前线程只有获取了锁,才能调用该组件的await()方法,而调用后,当前线程将缩放锁.
### 共同点 ###
1. 都是用来协调多线程对共享对象,变量的访问
2. 都是可重入锁,同一线程可以多次获得同一个锁
3. 都保证了可见性和互斥性


### 不同点 ###
1. ReentrantLock显示的获得,释放锁,synchronized隐式获得释放锁
2. ReentrantLock可响应中断,可轮回,synchronized是不可以响应中断的,为处理所得不可用性提供了更高的灵活性
3. ReentrantLock是API级别的，synchronized是JVM级别的
4. ReentrantLock可以实现公平锁
5. ReentrantLock通过Condition可以绑定多个条件 
6. 底层实现不一样， synchronized是同步阻塞，使用的是悲观并发策略，lock是同步非阻塞，采用的是乐观并发策略
7. Lock是一个接口，而synchronized是Java中的关键字，synchronized是内置的语言实现。 
8. synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁。 
9. Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断。 
10. 通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。 
11. Lock可以提高多个线程进行读操作的效率，既就是实现读写锁等。
