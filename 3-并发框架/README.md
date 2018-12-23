# 同步锁

> 锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源，
> 在Lock接口出现之前，Java应用程序只能依靠synchronized关键字来实现同步锁的功能，在java5以后，增加了JUC的并发包且提供了Lock接口用来实现锁的功能，它提供了与synchroinzed关键字类似的同步功能，只是它比synchronized更灵活，能够显示的获取和释放锁。



# Lock和synchronized的简单对比

通过我们对Lock的使用以及对synchronized的了解，基本上可以对比出这两种锁的区别了。因为这个也是在面试过程中比较常见的问题

- 从层次上，一个是关键字、一个是类， 这是最直观的差异
- 从使用上，lock具备更大的灵活性，可以控制锁的释放和获取； 而synchronized的锁的释放是被动的，当出现异常或者同步代码块执行完以后，才会释放锁
- lock可以判断锁的状态、而synchronized无法做到
- lock可以实现公平锁、非公平锁； 而synchronized只有非公平锁

# Lock的初步使用

>  Lock是一个接口，核心的两个方法
>
> - **lock**
>
> - **unlock**
>
> 它有很多的实现
>
> - **ReentrantLock**
> - **ReentrantReadWriteLock**

 

  ```java
public interface Lock {
    void lock();
    void unlock();
    Condition newCondition();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
}

public class ReentrantLock 
    	implements Lock, java.io.Serializable {}

public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {｝


  ```

## ReentrantLock

> 重入锁，表示支持重新进入的锁，也就是说，如果当前线程t1通过调用lock方法获取了锁之后，再次调用lock，是不会再阻塞去获取锁的，直接增加重试次数就行了。

### ReentrantLock结构图

![](image/lock.png)

![](image/lock1.png)

### ReentrantLock代码演示

```java

public class lockDemo {
    private static int count = 0;
    private static Lock lock = new ReentrantLock();

    public static void incr() {
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        lock.lock();
        count++;
        lock.unlock();

    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10000; i++) {
            new Thread(lockDemo::incr).start();
        }
        Thread.sleep(4000);
        System.out.println(count);
    }
}

```



## ReentrantReadWriteLock

>  我们以前理解的锁，基本都是排他锁，也就是这些锁在同一时刻只允许一个线程进行访问，而读写所在同一时刻可以允许多个线程访问，但是在写线程访问时，所有的读线程和其他写线程都会被阻塞。读写锁维护了一对锁，一个读锁、一个写锁; 一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量.



###ReentrantReadWriteLock代码演示

 ```java
public class RWLockDemo {
    static Map<String, Object> cacheMap = new HashMap<>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock read = rwl.readLock();
    static Lock write = rwl.writeLock();

    public static final Object get(String key) {
        System.out.println("开始读取数据");
        read.lock(); //读锁
        try {
            return cacheMap.get(key);
        } finally {
            read.unlock();
        }
    }

    public static final Object put(String key, Object value) {
        write.lock();
        System.out.println("开始写数据");
        try {
            return cacheMap.put(key, value);
        } finally {
            write.unlock();
        }
    }
}

 ```

> 在这个案例中，通过hashmap来模拟了一个内存缓存，然后使用读写所来保证这个内存缓存的线程安全性。
>
> 当执行读操作的时候，需要获取读锁，在并发访问的时候，读锁不会被阻塞，因为读操作不会影响执行结果。
>
> 在执行写操作是，线程必须要获取写锁，当已经有线程持有写锁的情况下，当前线程会被阻塞，只有当写锁释放以后，其他读写操作才能继续执行。
>
> 使用读写锁提升读操作的并发性，也保证每次写操作对所有的读写操作的可见性
>
> - 读锁与读锁可以共享
> - 读锁与写锁不可以共享（排他）
> -  写锁与写锁不可以共享（排他）



 # Lock的分析

## AQS

> Lock之所以能实现线程安全的锁
>
> 主要的核心是**AQS**(AbstractQueuedSynchronizer),AbstractQueuedSynchronizer
>
> 提供了一个**FIFO**队列，可以看做是一个用来实现锁以及其他需要同步功能的框架。这里简称该类为				AQS。
>
> AQS的使用依**靠继承**来完成，子类通过继承自AQS并实现所需的方法来管理同步状态。例如常见的ReentrantLock，CountDownLatch等AQS的两种功能
>
> 从使用上来说，AQS的功能可以分为两种：**独占和共享**。
>
> 独占锁模式下，每次只能有一个线程持有锁，比如前面给大家演示的ReentrantLock就是以独占方式实现的互斥锁
>
> 共享锁模式下，允许多个线程同时获取锁，并发访问共享资源，比如ReentrantReadWriteLock。
>
>
>
> 很显然，独占锁是一种悲观保守的加锁策略，它限制了读/读冲突，如果某个只读线程获取锁，则其他读线程都只能等待，这种情况下就限制了不必要的并发性，因为读操作并不会影响数据的一致性。
>
> 共享锁则是一种乐观锁，它放宽了加锁策略，允许多个执行读操作的线程同时访问共享资源



> 总结：AQS ---->FIFO ---->靠继承---->独占(ReentrantLock)和共享(ReentrantReadWriteLock)
>
> 总结：线程包装NODE ----> 从head添加 ---->从tail添加



### AQS的内部实现

> 同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。



```java
static final class Node {
  int waitStatus; //表示节点的状态，包含cancelled（取消）；condition 表示节点在等待condition也就是在condition队列中
  Node prev; //前继节点
  Node next; //后继节点
  Node nextWaiter; //存储在condition队列中的后继节点
  Thread thread; //当前线程
}
```



> AQS类底层的数据结构是使用双向链表，是队列的一种实现。包括一个head节点和一个tail节点，分别表示头结点和尾节点，其中头结点不存储Thread，仅保存next结点的引用。

> compareAndSetTail()  添加节点到尾部
>
> compareAndSetHead() 添加节点到头部
>
>



![](image/AQS-1.png)

> 当一个线程成功地获取了同步状态（或者锁），其他线程将无法获取到同步状态，转而被构造成为节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：compareAndSetTail(Node expect,Nodeupdate)，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。





