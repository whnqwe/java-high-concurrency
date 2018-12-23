# 同步锁

> 锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源，
> 在Lock接口出现之前，Java应用程序只能依靠synchronized关键字来实现同步锁的功能，在java5以后，增加了JUC的并发包且提供了Lock接口用来实现锁的功能，它提供了与synchroinzed关键字类似的同步功能，只是它比synchronized更灵活，能够显示的获取和释放锁。

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



# Lock和synchronized的简单对比

通过我们对Lock的使用以及对synchronized的了解，基本上可以对比出这两种锁的区别了。因为这个也是在面试过程中比较常见的问题

- 从层次上，一个是关键字、一个是类， 这是最直观的差异
- 从使用上，lock具备更大的灵活性，可以控制锁的释放和获取； 而synchronized的锁的释放是被动的，当出现异常或者同步代码块执行完以后，才会释放锁
- lock可以判断锁的状态、而synchronized无法做到
- lock可以实现公平锁、非公平锁； 而synchronized只有非公平锁