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
HashMap的数组长度是16,随着HashMap里的键值对越来越多,在数组数量不变的情况下,查找效率会越来越低,解决效率低的问题就是需要响应的把数组的数量增加(HashMap内部是原来数组长度乘以2),这就是网上所谓的扩容.负载因子就是规定什么时候需要扩容.HashMap存在一个阈值(threshold),只要HashMap里的键值对大于等于这个阈值,那么就需要扩容.阈值的计算公式:阈值=当前数组长度*负载因子.HashMap中默认的负载因子是0.75,默认情况下第一次扩容判断阈值是16*0.75=12.所以第一次存键值对的时候,在存到第13个键值对时就需要扩容了.

### hash和Rehash ###
每次扩容后,转移旧表键值对到新表之前都要重新rehash,计算键值对在新表的索引.

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
