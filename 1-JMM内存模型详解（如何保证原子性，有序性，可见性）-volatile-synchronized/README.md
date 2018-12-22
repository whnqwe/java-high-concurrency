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

> **Synchronized如何实现锁**
>
> **为什么每个对象都可以称为锁**

> 解决原子性，可见性，有序性
>
> 在多线程并发编程中synchronized一直是元老级角色，很多人都会称呼它为
>
> - **重量级锁**
>
> Java SE 1.6中为了减少获得锁和释放锁带来的性能消耗而引入的
>
> - **偏向锁 **
>
> - **轻量级锁**
>
> 以及锁的**存储结构**和**升级过程**。

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



> 全局锁 （Object.class）

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

> 全局锁 （静态方法）

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

> 全局锁（静态变量）

```java
public class Demo {
    private    static  int count  = 0;
    private Object lock = new Object();
    public  static void incr(){
        synchronized (lock){
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



#### 修饰代码块



#### 修饰实例方法

```java
synchronized (this){
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        count ++;
    }
```

#### synchronized括号后面的对象

> synchronized括号后面的对象是一把锁
>
> - 在java中任意一个对象都可以成为锁
>
> 简单来说，我们把object比喻是一个key，拥有这个key的线程才能执行这个方法，拿到这个key以后在执行方法过程中，这个key是随身携带的，并且只有一把。如果后续的线程想访问当前方法，因为没有key所以不能访问只能在门口等着，等之前的线程把key放回去。所以，synchronized锁定的对象必须是同一个，如果是不同对象，就意味着是不同的房间的钥匙，对于访问者来说是没有任何影响的

#### synchronized在方法体上，在方法体里面javap的不同

##### 方法体上

> ACC_SYNCHRONIZED

![](image/fanfangtishang.png)



##### 方法体里面

> - monitorenter
>
> - monitorexit

![](image/fanfanglimian.png)

#### synchronized的字节码指令(monitorenter/monitorexit)

> 通过javap -v 来查看对应代码的字节码指令，对于同步块的实现使用了
>
> - monitorenter
> - monitorexit
>
> 前面我们在讲JMM的时候，提到过这两个指令，他们隐式的执行了
>
> - Lock
>
> - UnLock
>
> 用于提供原子性保证。
>
> - monitorenter指令插入到同步代码块开始的位置
>
> - monitorexit指令插入到同步代码块结束位置
>
> jvm需要保证每个monitorenter都有一个monitorexit对应。
>
> 这两个指令，本质上都是对一个对象的
>
> - 监视器(monitor)进行获取，这个过程是排他的
>
> 也就是说同一时刻只能有一个线程获取到由synchronized所保护对象的监视器线程执行到monitorenter指令时，会尝试获取对象所对应的monitor所有权，也就是尝试获取对象的锁；而执行monitorexit，就是释放monitor的所有权



#### synchronized的锁的原理

>  jdk1.6以后对synchronized锁进行了优化，包含
>
> - 偏向锁
>
> - 轻量级锁
>
> - 重量级锁 
>
> 在了解synchronized锁之前，我们需要了解两个重要的概念，一个是对象头、另一个是monitor

##### 锁存放在哪个地方(对象头)

> 对象在内存中的布局分为三块区域：
>
> - 对象头
>
> - 实例数据
>
> - 对齐填充
>
> Java对象头是实现synchronized的锁对象的**基础**，一般而言，synchronized使用的**锁对象是存储在Java对象头里**。它是轻量级锁和偏向锁的关键



###### mark work

> Mark Word用于存储对象自身的运行时数据，
>
> - 哈希码（HashCode）
>
> - GC分代年龄
>
> - 锁状态标志
>
> - 线程持有的锁
>
> - 偏向线程 ID
>
> - 偏向时间戳等等。
>
> Java对象头一般占有两个机器码（在32位虚拟机中，1个机器码等于4字节，也就是32bit）



> 32位

![](image/newmarkwork.png)



> 64位

###### 对象头JVM源码中的体现(oop.hpp)

> 如果想更深入了解对象头在JVM源码中的定义，需要关心几个文件，oop.hpp/markOop.hpp
> oop.hpp，每个 Java Object 在 JVM 内部都有一个 native 的 C++ 对象 oop/oopDesc 与之对应。先在oop.hpp中看oopDesc的定义

```c++

class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
}
```



######  对象头JVM源码中的体现(markOop.hpp)

> 从上面的枚举定义中可以看出，对象头中主要包含了GC分代年龄、锁状态标记、哈希码、epoch等信息。

   ```c++

enum { age_bits                 = 4,
      lock_bits                = 2,
      biased_lock_bits         = 1,
      max_hash_bits            = BitsPerWord - age_bits - lock_bits - biased_lock_bits,
      hash_bits                = max_hash_bits > 31 ? 31 : max_hash_bits,
      cms_bits                 = LP64_ONLY(1) NOT_LP64(0),
      epoch_bits               = 2
};
   ```



> 对象的状态一共有五种，分别是无锁态、轻量级锁、重量级锁、GC标记和偏向锁。在32位的虚拟机中有两个Bits是用来存储锁的标记为的，但是我们都知道，两个bits最多只能表示四种状态：00、01、10、11，那么第五种状态如何表示呢 ，就要额外依赖1Bit的空间，使用0和1来区分。
>
> - locked_value(00) = 0
>
> - unlocked_value(01) = 1
>
> - monitor_value(10) = 2
>
> - marked_value(11) = 3
>
> - biased*lock*pattern(101) = 5

markOop.hpp类中有关于对象状态的定义：

```c++
enum { locked_value             = 0,
         unlocked_value           = 1,
         monitor_value            = 2,
         marked_value             = 3,
         biased_lock_pattern      = 5
  };
```

##### 为什么任何一个对象都可以锁

-  oop.hpp下的oopDesc类是JVM对象的顶级基类，所以每个object对象都包含markOop

```c++
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
}
```

- markOop.hpp中markOopDesc继承自oopDesc，并扩展了自己的monitor方法，这个方法返回一个
  ObjectMonitor指针对象

```c++
  ObjectMonitor* monitor() const {
    assert(has_monitor(), "check");
    // Use xor instead of &~ to provide one extra tag-bit check.
    return (ObjectMonitor*) (value() ^ monitor_value);
  }
```

-  objectMonitor.hpp,在hotspot虚拟机中，采用ObjectMonitor类来实现monitor

```c++
  ObjectMonitor() {
    _header       = NULL; //markwork对象头
    _count        = 0;
    _waiters      = 0, //等待线程数
    _recursions   = 0; //重入次数
    _object       = NULL;
    _owner        = NULL; //指向获得ObectWaiter对象的线程
    _WaitSet      = NULL; //处于wait状态的线程，会被加入watieset
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;//JVM为每一个尝试进去synchronized的线程创建一个ObjectWait并加入到_cxq中
    FreeNext      = NULL ;
    _EntryList    = NULL ;
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }
```



##### synchronized如何实现锁(锁升级和获取过程)

> 了解了对象头以及monitor以后，接下来去分析synchronized的锁的实现，就会非常简单了。前面讲过
> synchronized的锁是进行过优化的，引入了偏向锁、轻量级锁；锁的级别从低到高逐步升级
>
> - 无锁
>
> - 偏向锁
>
> - 轻量级锁
>
> - 重量级锁



##### 



