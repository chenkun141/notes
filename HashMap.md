## HashMap ##
### 参考网站 ###
[https://juejin.im/post/5a215783f265da431d3c7bba](https://juejin.im/post/5a215783f265da431d3c7bba)  
[https://juejin.im/post/5a224e1551882535c56cb940](https://juejin.im/post/5a224e1551882535c56cb940)
[https://juejin.im/post/5a23f82ff265da432003109b](https://juejin.im/post/5a23f82ff265da432003109b)
### HashMap是什么 ###
HashMap是以数组和链表的组合构成的数据结构,Java8中将链表长度超过8的链表转化成红黑树;存取是都会根据键值计算出HashCode,在根据HashCode定位到数组中的位置并执行操作.

### HashCode是什么 ###
HashCode是一个对象的标识,Java中对象的HashCode是一个int类型值,通过HashCode来指定数组的索引可以快速定位到对象在数组中的位置,之后在遍历链表找到对应值,理想情况下时间复杂度为O(1),并且不同对象可以拥有相同的HashCode.

### 负载因子是什么 ###
HashMap的数组长度是16,随着HashMap里的键值对越来越多,在数组数量不变的情况下,查找效率会越来越低,解决效率低的问题就是需要响应的把数组的数量增加(HashMap内部是原来数组长度乘以2),这就是网上所谓的扩容.负载因子就是规定什么时候需要扩容.HashMap存在一个阈值(threshold),只要HashMap里的键值对大于等于这个阈值,那么就需要扩容.阈值的计算公式:阈值 = 当前数组长度 x 负载因子.HashMap中默认的负载因子是0.75,默认情况下第一次扩容判断阈值是 16 x 0.75 = 12.所以第一次存键值对的时候,在存到第13个键值对时就需要扩容了.

### hash和Rehash ###
每次扩容后,转移旧表键值对到新表之前都要重新rehash,计算键值对在新表的索引.

### Java7--put ###

	public V put(K key, V value) {
	        //数组为空时创建数组
	        if (table == EMPTY_TABLE) {
	            inflateTable(threshold);
	        }
	        //key为空单独对待
	        if (key == null)
	            return putForNullKey(value);
	        //①根据key计算hash值
	        int hash = hash(key);
	        //②根据hash值和当前数组的长度计算在数组中的索引
	        int i = indexFor(hash, table.length);
	        //遍历整条链表
	        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
	            Object k;
	            //③情况1.hash值和key值都相同的情况，替换之前的值
	            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
	                V oldValue = e.value;
	                e.value = value;
	                e.recordAccess(this);
	                //返回被替换的值
	                return oldValue;
	            }
	        }
	
	        modCount++;
	        //③情况2.坑位没人,直接存值或发生hash碰撞都走这
	        addEntry(hash, key, value, i);
	        return null;
	    }

### 问题 ###
- 我们总是习惯用一个String作为HashMap的key，这是为什么呢？其它的类可以做为HashMap的key吗？   
	这里因为String是不可以变的，并且java为它实现了hashcode的缓存技术。我们在put和get中都需要获取key的hashcode，这些方法的效率很大程度上取决于获取hashcode的，所以用String的原因：1、它是不可变的。2、它实现了hashcode的缓存，效率更高。

- 在多线程情况下使用HashMap时,可能会出现环形链表,判断环形链表的方法是(嵌套for循环,首先创建两个指针1和2（在java里就是两个对象引用），同时指向这个链表的头节点。然后开始一个大循环，在循环体中，让指针1每次向下移动一个节点，让指针2每次向下移动两个节点，然后比较两个指针指向的节点是否相同。如果相同，则判断出链表有环，如果不同，则继续下一次循环。),在高并发情况下推荐使用**ConcurrentHashMap**,即兼顾线程安全和性能,

- HashMap和HashTable的区别  
1. hashMap是线程不安全的,hashMap是一个接口,是Map的一个子接口,是将键映射到值的对象,不允许键值重复,允许空键和空值,由于非线程安全,所以hashMap的效率要比hashTable的效率高一些.  
2. hashTable是线程安全的一个集合,不允许null值作为一个key值或者value值.  
3. hashTable是sychronize,多个线程访问是不需要自己为它实现同步,在被多个线程访问时候需要自己为它的方法实现同步.    

- WeakHashMap 是怎么工作的？  
WeakHashMap 的工作与正常的 HashMap 类似，但是使用弱引用作为 key，意思就是当 key 对象没有任何引用时，key/value 将会被回收。

## ConcurrentHashMap ##
ConcurrentHashMap,它内部细分了若干个晓得HashMap,称之为段(Segment).默认情况下一个ConcurrentHashMap被进一步细分为16个段,就是锁的并发度.  
如果需要在ConcurrentHashMap中添加一个新的表项,并不是将整个HashMap加锁,而是首先根据hashcode得到该表项应该存放在哪个段中,然后对该段加锁,并完成put操作,则线程间可以做到真正的并行.  
### ConcurrentHashMap是由segment数组结构和HashEntry数组结构组成 ###
ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成.Segment是一种可重入锁ReentrantLock,在ConcurrentHashMap里扮演锁的角色,HashEntry则用于存储键值对数据.一个ConcurrentHashMap里包含一个Segment数组,Segment的结构和HashMap类似,是一种数组和链表结构,一个Segment里包含一个HashEntry数组,每个HashEntry是一个链表结构的元素,每个Segment守护一个HashEntry数组里的元素,当对HashEntry数组的数据进行修改时,必须首先获得它对应的Segment锁.  
![](https://i.imgur.com/Z2IzNOn.png)
### Segment段 ###
ConcurrentHashMap 和 HashMap 思路是差不多的，但是因为它支持并发操作，所以要复杂一些。整个 ConcurrentHashMap 由一个个 Segment 组成，Segment 代表”部分“或”一段“的意思，所以很多地方都会将其描述为分段锁。注意，行文中，我很多地方用了“槽”来代表一个 segment。
### 线程安全（Segment 继承 ReentrantLock 加锁） ###
简单理解就是，ConcurrentHashMap 是一个 Segment 数组，Segment 通过继承 ReentrantLock 来进行加锁，所以每次需要加锁的操作锁住的是一个 segment，这样只要保证每个 Segment 是线程安全的，也就实现了全局的线程安全。
