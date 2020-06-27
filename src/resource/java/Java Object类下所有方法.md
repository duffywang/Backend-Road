## Java Object类下所有方法

一共有11个方法，9大类

### 我是谁
String toString()：返回当前对象信息，默认为对象类名+ @ +hashCode的16进制数字，如Test$A@1b6d3586，一般会重写方便调试
```java
public class Student {

    private int age;

    private String name;

    @Override
    public String toString() {
        return "Student{" +
                "age=" + age +
                ", name='" + name + '\'' +
                '}';
    }
}

```

Class<?> getClass()：返回对象运行时的类，不可重写，要调用的话，一般和getName()联合使用

### 我和他人是否一致
boolean equals()：判断对象指向的内存地址引用是否一致，

int hashcode()：对象的哈希码值，如-1951809095
常问：equals和hashCode之间关系
1. 两个对象equals相等则hashCode也一致，重写equals则需要重写hashCode？HashMap/HashSet的“键”部分存放自定义的对象，对象实际是相同的但指向地址不同，所以需要重写。 
2. 两个对象的hashCode相等，equals不一定相等
3. 一般hashCode用作在HashMap 、 HashSet等中比较两个对象是否，如果相同再通过equals对比
4. 一个相同的类在不同的JVM下调用hashCode可能返回不同的值

### 垃圾回收
void finalize()：垃圾回收器释放对象占用的存储空间前 —— 执行finalize() —— 下一次垃圾回收时，真正的回收对象占用的内存。如果实现了这个方法，对象可能在这个方法中再次复活，从而避免被 GC 回收

void clone()：实现对象的浅拷贝，只有实现对象的Cloneable才可以调用该方法。浅拷贝是指对象内属性引用的对象只会拷贝引用地址，而不会将引用的对象重新分配内存，深拷贝是指将引用对象也重新创建

### 我和其他线程之间
void notify()：唤醒该对象上的**某个**等待线程

void notifyAll()：唤醒该对象上的**所有**等待线程

void wait()、void wait(long timeout)、void wait(long timeout, int nanos)
当前线程进入等待状态，释放持有的锁，直到以下4种情况出现，该线程才可以被调度

1. 其他线程调用了该对象的notify方法
2. 其他线程调用了该对象的notifyAll方法
3. 其他线程调用了interrupt中断该线程,如果是被中断的话就抛出一个InterruptedException异常
4. 时间间隔timeout到了
**注：notify 和wait方法必须在synchronized 修饰的同步方法或同步代码中使用，Java提供的是对象级的锁而不是线程级的，wait释放锁前提是获取锁，notify将锁还给执行wait的线程**


可重写的方法是：equals() hashCode() finalize() clone() toString()
不能被重写是为了保证一个子类有多重继承关系时，子类和父类表现一致