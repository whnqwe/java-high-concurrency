# java并发笔记

## 硬件架构模型图

![](image/mode.png)



## 什么时候使用多线程

1. 通过并行提高程序的性能
2. 网络等待，IO响应导致的延迟问题



## 应用多线程

### 继承Thread

> Thread 类本质上是实现了 Runnable 接口的一个实例，代表一个线程的实例。启动线程的唯一方法就是通过 Thread 类的 start()实例方法。start()方法是一个native 方法，它会启动一个新线程，并执行 run()方法。这种方式实现多线程很简单，通过自己的类直接 extend Thread，并复写 run()方法，就可以启动新线
> 程并执行自己定义的 run()方法



```java
public class MyThread extends Thread {

    public void run() {
        System.out.println("MyThread.run()");
    }

    public static void main(String[] args) {
        MyThread myThread1 = new MyThread();
        MyThread myThread2 = new MyThread();
        myThread1.start();
        myThread2.start();
    }
}
```



### 实现Runnable

```java
public class MyThread implements Runnable{

    public void run() {
        System.out.println("MyThread.run()");
    }

    public static void main(String[] args) {
        Thread myThread1 = new Thread(new MyThread());
        Thread myThread2 = new Thread(new MyThread());
        myThread1.start();
        myThread2.start();
    }
}
```



###  带返回值的ExecutorService、Callable、Future

>有的时候，我们可能需要让一步执行的线程在执行完成以后，提供一个返回值给到当前的主线程，主线程需要依赖这个值进行后续的逻辑处理，那么这个时候，就需要用到带返回值的线程了



```java
public class CallableDemo implements Callable<Boolean> {


    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executorService =  Executors.newFixedThreadPool(6);
        Future<Boolean> future = executorService.submit(new CallableDemo());
        if(future.get()){
            System.out.println("线程执行成功");
        }
    }


    @Override
    public Boolean call() throws Exception {
        System.out.println("线程开始执行");
        System.out.println("线程执行结束");
        return true;
    }
}
```



## 并发编程基础

### 线程



#### 线程的状态

> BLOCKED:阻塞状态
>
> - 等待阻塞：运行的线程执行 wait 方法，jvm 会把当前线程放入到等待队列
>
> - 同步阻塞: 运行的线程在获取对象的同步锁时，若该同步锁被其他线程锁占
>   用了，那么 jvm 会把当前的线程放入到锁池中
>
> - 其他阻塞:运行的线程执行 Thread.sleep 或者 t.join 方法，或者发出了 I/O
>   请求时，JVM 会把当前线程设置为阻塞状态，当 sleep 结束、join 线程终止、
>   io 处理完毕则线程恢复

```java
public enum State {

        NEW,  //初始状态，线程被构建，但是还没有调用 start 方法

        RUNNABLE, //运行状态，JAVA 线程把操作系统中的就绪和运行两种状态统一称为“运行中”

        BLOCKED, //阻塞状态，表示线程进入等待状态,也就是线程因为某种原因放弃了 CPU 使用权，阻塞分为：等待阻塞，同步阻塞，其他阻塞

        WAITING, //等待状态

        TIMED_WAITING, //超时等待状态，超时以后自动返回

        TERMINATED; //终止状态，表示当前线程执行完毕
    }
```

 ![](image/threadstate.png)





#### 线程状态的演示

```java
public class ThreadStatus {

    public static void main(String[] args) {
        //TIME_WAITING
        new Thread(()->{
           while (true){
               try {
                   TimeUnit.SECONDS.sleep(1);
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
        },"timewaiting").start();

        //WAITING，线程在 ThreadStatus 类锁上通过 wait 进行等待
        new Thread(()->{
            while(true){
                synchronized (ThreadStatus.class){
                    try {
                        ThreadStatus.class.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        },"Waiting").start();

        //线程在 ThreadStatus 加锁后，不会释放锁
        new Thread(new BlockedDemo(),"BlockDemo-01").start();
        new Thread(new BlockedDemo(),"BlockDemo-02").start();

    }



    static class BlockedDemo extends Thread{
        public void run(){
            synchronized (BlockedDemo.class){
                while(true){
                    try {
                        TimeUnit.SECONDS.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}

```



