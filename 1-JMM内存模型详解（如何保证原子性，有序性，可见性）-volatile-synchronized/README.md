# JMM



## 多线程之间的通信

#### 共享内存

![](image/gongxiangneicun.png)

#### 消息传递

> wait()/notify()



## JMM产生的问题

#### 问题1-可见性

> 工作内存的值，什么时候同步到主内存

![](image/wenti1.png)



> 主内存什么时候同步到工作内存中

![](image/wenti2.png)

#### 问题2-原子性

> 主内存什么时候同步到工作内存中

![](image/yuanzixing.png)



#### 问题2-有序性

> - 编译器重排序
> - 处理器重排序
> - 内存系统重排序

## JMM与物理硬件之间的映射

![](image/wuliyinshe.png)



## 解决原子性、可见性、有序性

> 在Java中提供了一系列和并发处理相关的关键字，比如volatile、Synchronized、final、juc等，这些就是Java内存模型封装了底层的实现后提供给开发人员使用的关键字

> 原子性：Synchronized ，CAS, AQS
>
> > monitorenter
> >
> > monitorexit 
>
> 可见性：
>
> > volatile : 被其修饰的变量在被修改后可以立即同步到主内存，被其修饰的变量在每次是用之前都从主内存刷新。因此，可以使用volatile来保证多线程操作时变量的可见性。
> >
> > Synchronized 
> >
> >  final
>
> 有序性：使用synchronized和volatile来保证多线程之间操作的有序性
>
> > volatile:禁止指令重排
> >
> > Synchronized:保证同一时刻只允许一条线程操作



# volatile 

## 保证可见性

#### 查看lock汇编指令

1. 下载hsdis工具 ，https://sourceforge.net/projects/fcml/files/fcml-1.1.1/hsdis-1.1.1-win32-amd64.zip/download

2. 解压后存放到jre目录的server路径下

3. 然后跑main函数，跑main函数之前，加入如下虚拟机参数：

   ```java
   -server -Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*App.getInstance（替换成实际运行的代码）
   ```

#### lock的作用

> volatile变量修饰的共享变量，在进行写操作的时候会多出一个lock前缀的汇编指令，这个指令在前面我们讲解CPU高速缓存的时候提到过，会触发总线锁或者缓存锁，通过缓存一致性协议来解决可见性问题对于声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，把这个变量所在的缓存行的数据写回到系统内存，再根据我们前面提到过的MESI的缓存一致性协议，来保证多CPU下的各个高速缓存中的数据的一致性。

## 防止指令重排序

#### 指令重排序代码演示

> 1. x=1  y=1
> 2. x=0  y=1
> 3. x=1 y=0
> 4. x=0  y=0

```java
public class VolatileDemo {

    private static int x=0;
    private static int y=0;
    private static int a=0;
    private static int b=0;

    public static void main(String[] args) throws InterruptedException {
      Thread t1 = new Thread(()->{
           a=1;
           x=b;
        });
        Thread t2 =  new Thread(()->{
            b=1;
            y=a;
        });
        t1.start();
        t2.start();
        t1.join(); //等待线程执行结果
        t2.join();//等待线程执行结果
        System.out.println("x="+y+"  "+"y="+y);
    }

}

```

#### 存在的问题

- 编译器的指令重排序   

  > 解决办法：优化屏障		 

- 处理器的指令重排序

  > 解决办法：内存屏障

#### 防止CPU的指令重排序--内存屏障

>  现在的CPU架构都提供了内存屏障功能，在x86的cpu中，实现了相应的内存屏障
>
> - 写屏障(store barrier)
>
> - 读屏障(load barrier)
>
> - 全屏障(Full Barrier)
>
>   主要的作用是
>
>   > - 防止指令之间的重排序
>
>   > - 保证数据的可见性

##### store barrier

> store barrier称为写屏障，相当于storestore barrier,
>
> > 强制所有在storestore内存屏障之前的所有执行，都要在该内存屏障之前执行，并发送缓存失效的信号。
> >
> > 所有在storestore barrier指令之后的store指令，都必须在storestore barrier屏障之前的指令执行完后再被执行。也就是禁止了写屏障前后的指令进行重排序。
> >
> > store barrier之前发生的内存更新都是可见的（这里的可见指的是修改值可见以及操作结果可见）。



![](image/storestore.png)

##### load barrier

> load barrier称为读屏障，相当于loadload barrier。
>
> > 强制所有在load barrier读屏障之后的load指令，都在loadbarrier屏障之后执行。
> >
> > 也就是禁止对load barrier读屏障前后的load指令进行重排序。
> >
> > 配合store barrier，使得所有store barrier之前发生的内存更新，对load barrier之后的load操作是可见的。



![](image/loadload.png)

##### Full Barrier

> full barrier成为全屏障，相当于storeload
>
> > 是一个全能型的屏障，因为它同时具备前面两种屏障的效果。强制了所有在storeload barrier之前的store/load指令，都在该屏障之前被执行，
> >
> > 所有在该屏障之后的的store/load指令，都在该屏障之后被执行。禁止对storeload屏障前后的指令进行重排序。

![](image/storeload.png)



##### 总结

> 内存屏障只是解决顺序一致性问题，不解决缓存一致性问题，缓存一致性是由cpu的缓存锁以及MESI协议来完成的。而缓存一致性协议只关心缓存一致性，不关心顺序一致性。



#### 编译器层面防止指令重排序---(优化屏障)

>在编译器层面，通过volatile关键字，取消编译器层面的缓存和重排序。保证编译程序时在优化屏障之前的指令不会在优化屏障之后执行。这就保证了编译时期的优化不会影响到实际代码逻辑顺序。

> 如果硬件架构本身已经保证了内存可见性，那么volatile就是一个空标记，不会插入相关语义的内存屏障。如果硬件架构本身不进行处理器重排序，有更强的重排序语义，那么volatile就是一个空标记，不会插入相关语义的内存屏障。

> 在JMM中把内存屏障指令分为4类，通过在不同的语义下使用不同的内存屏障来禁止特定类型的处理器重排序，从而来保证内存的可见性



> 对每个volatile 写操作的前面插入  storestorebarriers
>
> 对每个volatile 写操作的后面插入 storeloadbarriers
>
> 对每个volatile 读操作的前面插入  laodlaodbarriers
>
> 对每个volatile 读操作的后面插入  loadstorebarriers

##### LoadLoad Barriers

> load1 ;     LoadLoad;      load2 
>
> > 确保load1数据的装载优先于load2及所有后续装载指令的装载



##### StoreStore Barriers

> store1; storestore;store2 
>
> > 确保store1数据对其他处理器可见优先于store2及所有后续存储指令的存储



##### LoadStore Barries 

> load1;loadstore;store2
>
> > 确保load1数据装载优先于store2以及后续的存储指令刷新到内存



##### StoreLoad Barries

> store1; storeload;load2
>
> > 确保store1数据对其他处理器变得可见， 优先于load2及所有后续装载指令的装载；这条内存屏障指令是一个全能型的屏障，在前面讲cpu层面的内存屏障的时候有提到。它同时具有其他3条屏障的效果



## volatile为什么不能保证原子性

> 无法保证复合操作的原子性

```java
public class Demo {
  volatile int i;
  public void incr(){
    i++;
 }
  public static void main(String[] args) {
    new Demo().incr();
 }
}
```

> 查看字节码,对一个原子递增的操作，会分为三个步骤：
>
> 1. getfield 
>
> >  读取volatile变量的值；
>
> 2. iadd
>
> > 增加变量的值；
>
> 3. putfield 
>
>    > 把值写回让其他线程可见



  通过内存屏障可以保证store与load的顺序

> loadA  loadB  storeA  storeB  可以保证  store能都在load指令前面或者后面执行。但是并不能保证loadA  ，storeA的连续执行
>
> 已i++ 为例，只能在一个线程store完毕之后才能对其他线程可见，否则其他线程得到的可能是旧的值。
>
>

# Synchronized 

> 解决原子性，可见性，有序性
>
> 在多线程并发编程中synchronized一直是元老级角色，很多人都会称呼它为重量级锁。Java SE 1.6中为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁，以及锁的存储结构和升级过程。

## synchronized代码演示

```java
public class Demo {
    private    static  int count  = 0;

    public  static void incr(){
        synchronized (Demo.class){
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            count ++;
        }

    }
    public static void main(String[] args) throws InterruptedException {
        for(int i=0;i<1000;i++){
            new Thread(Demo::incr).start();
        }
        Thread.sleep(4000);
        System.out.println(count);
    }
}
```

## synchronized的三种应用方式

####  静态方法

> 作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁

> 全局锁

```java
public class Demo {
    private    static  int count  = 0;

    public  static void incr(){
        synchronized (Demo.class){
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            count ++;
        }
    }
}
```

```java
public class Demo {
    private    static  int count  = 0;

    public  synchronized static void incr(){
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            count ++;
    }
}
```



#### 修饰代码块





#### 修饰实例方法

public class Demo {
​    private    static  int count  = 0;

```java
public  static void incr(){
    synchronized (this){
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        count ++;
    }
  }
}
```


