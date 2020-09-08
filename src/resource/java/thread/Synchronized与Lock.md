## Synchronized与Lock

### 区别
- 存在层面：Synchronized 是Java中的一个关键字，存在于JVM层面；Lock是并发包中一个接口
- 操作层面：Synchronized可修饰方法，方法块；Lock() unlock() tryLock() 等待超时tryLock(time,unit) 等待可中断lockInterruptibly()
- 锁的类型：都是可重入的，Synchronized是不可中断的，是非公平锁；Lock是可中断的，可选公平锁或非公平锁

### AQS
考察点 ：公平/非公平 加锁 可重入 
- 用于构建锁和同步器的框架，支持ReentrantLock、Semaphore，ReentrantReadWriteLock、CountDownLatch
- 内部使用一个volatile的变量state作为资源，获取或修改均为CAS操作
- 实现排队和阻塞机制，内部使用先进先出双端队列（同步队列），使用head 和tail标示队列的头部和尾部；以及等待队列（单向队列）
- 独占模式：一次只能一个线程获取，acquire(arg)arg始终是1，如ReentrantLock 共享模式：可以被多个线程获取，具体资源个数可以通过参数指定，如CountDownLatch
- 对于非公平锁只要CAS设置同步状态成功则表示当前线程获取了锁，而公平锁则不同。公平锁使用tryAcquire方法，该方法与nonfairTryAcquire的唯一区别就是判断条件中多了对同步队列中当前节点是否有前驱节点的判断，如果该方法返回true表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。
- 重进入指的是任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实现需要解决两个问题：①线程再次获取锁，锁需要去识别获取锁的线程是否为当前占有锁的线程，如果是则再次获取成功。②锁的最终释放，线程重复n次获取了锁，随后在第n次释放该锁后，其他现场能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而被锁释放时，计数自减，当计数为0时表示锁已经成功释放。
