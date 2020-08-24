## HashMap/ConcurrentHashMap 





ConcurrentHashMap
1 底层数据结构
1.7：Segment + HashEntry,Segment是可重入锁（ReentrantLock）,HashEntry用于存储键值对
1.8：Node + CAS + Synchronized，数据结构与HashMap1.8类似，都是使用数组+链表（红黑树），Synchronized只是锁定当前链表或红黑树的首节点，只要Hash不冲突就不会产生并发
HashEntry和Node 中value和Entry都是用volatile修饰，保证可见性

2 put()操作
1.7：先定位Segment再定位桶，同一个Segment put全程加ReentryLock,没有获取锁的线程提前找桶的位置，并最多自旋64次获取锁，超过则挂起。
1.8：移除了Segment，类似HashMap直接定位到桶，拿到first节点后进行判断，0、为空则初始化数组；1、为空则CAS插入；2、为-1则说明在扩容，则跟着一起扩容；3、synchronized的方式将节点插入到链表或红黑树中；4、判断是否需要扩容

3 size()
1.7：先计算两次，如果不变就返回；否则锁住所有Segment求和
1.8：

4 resize()
1.7：跟HashMap步骤一样，只不过是搬到单线程中执行，避免了HashMap在1.7中扩容时死循环的问题，保证线程安全。
1.8：支持并发扩容，HashMap扩容在1.8中由头插改为尾插（为了避免死循环问题），ConcurrentHashmap也是，迁移也是从尾部开始，扩容前在桶的头部放置一个hash值为-1的节点，这样别的线程访问时就能判断是否该桶已经被其他线程处理过了。
