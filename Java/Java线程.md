[TOC]

>进程和线程

 进程：是系统进行资源分配和调度的一个独立单位，也是一个具有独立功能的程序；
 线程：线程是进程的一个实体,是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位.线程自己基本上不拥有系统资源,只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈),但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源。

那么系统在创建进程的时候会分配什么资源呢？可参考[App启动流程](http://www.jianshu.com/p/8b58ef70c3ab)

进程和线程的关系好比国有企业，进程就好比企业的董事长，他可以向国家财务部申请资金，财务部答应给两千万，好了，那么这些资金怎么用呢？是这个国有企业下面的每一个部门，这些个部门是最终消费的个个体，这就好比线程，是最终CPU调度和分配的基本单位。

>创建线程的方式

###### 一、继承Thread类创建线程类
（1）定义Thread类的子类，并重写该类的run方法，该run方法的方法体就代表了线程要完成的任务。因此把run()方法称为执行体。
（2）创建Thread子类的实例，即创建了线程对象。
（3）调用线程对象的start()方法来启动该线程

```java
package com.thread;  
  
public class FirstThreadTest extends Thread{  
    int i = 0;  
    //重写run方法，run方法的方法体就是现场执行体  
    public void run()  
    {  
        for(;i<100;i++){  
        System.out.println(Thread.currentThread().getName()+"  "+i);  
        }
    }  
    public static void main(String[] args)  {  
         new FirstThreadTest().start();  
         new FirstThreadTest().start();  
    }  
}  
```
###### 二、通过Runnable接口创建线程类
（1）定义runnable接口的实现类，并重写该接口的run()方法，该run()方法的方法体同样是该线程的线程执行体。
（2）创建 Runnable实现类的实例，并依此实例作为Thread的target来创建Thread对象，该Thread对象才是真正的线程对象。
（3）调用线程对象的start()方法来启动该线程。

```java
package com.thread;  
  
public class RunnableThreadTest implements Runnable  
{  
  
    private int i;  
    public void run()  
    {  
        for(i = 0;i <100;i++)  
        {  
            System.out.println(Thread.currentThread().getName()+" "+i);  
        }  
    }  
    public static void main(String[] args)  
    {  
                RunnableThreadTest rtt = new RunnableThreadTest();  
                new Thread(rtt,"新线程1").start();  
                new Thread(rtt,"新线程2").start();  
    }  
} 
```
###### 三、通过Callable和Future创建线程
（1）创建Callable接口的实现类，并实现call()方法，该call()方法将作为线程执行体，并且有返回值。
（2）创建Callable实现类的实例，使用FutureTask类来包装Callable对象，该FutureTask对象封装了该Callable对象的call()方法的返回值。
（3）使用FutureTask对象作为Thread对象的target创建并启动新线程。
（4）调用FutureTask对象的get()方法来获得子线程执行结束后的返回值。

```
package com.thread;  
  
import java.util.concurrent.Callable;  
import java.util.concurrent.ExecutionException;  
import java.util.concurrent.FutureTask;  
  
public class CallableThreadTest implements Callable<Integer>  
{  
    public static void main(String[] args)  
    {  
        CallableThreadTest ctt = new CallableThreadTest();  
        FutureTask<Integer> ft = new FutureTask<>(ctt);  
       new Thread(ft,"有返回值的线程").start();  
        try  
        {  
            System.out.println("子线程的返回值："+ft.get());  
        } catch (InterruptedException e)  
        {  
            e.printStackTrace();  
        } catch (ExecutionException e)  
        {  
            e.printStackTrace();  
        }  
    }  
    @Override  
    public Integer call() throws Exception  
    {  
        int i = 0;  
        for(;i<100;i++)  
        {  
            System.out.println(Thread.currentThread().getName()+" "+i);  
        }  
        return i;  
    }  
}
```
###### 创建线程的三种方式的对比
采用实现Runnable、Callable接口的方式创见多线程时
优势是：
​        线程类只是实现了Runnable接口或Callable接口，还可以继承其他类。
​       在这种方式下，多个线程可以共享同一个target对象，所以非常适合多个相同线程来处理同一份资源的情   况，从而可以将CPU、代码和数据分开，形成清晰的模型，较好地体现了面向对象的思想。
劣势是：
​      编程稍微复杂，如果要访问当前线程，则必须使用Thread.currentThread()方法。

使用继承Thread类的方式创建多线程时
优势是：
编写简单，如果需要访问当前线程，则无需使用Thread.currentThread()方法，直接使用this即可获得当前线程。
劣势是：
线程类已经继承了Thread类，所以不能再继承其他父类。

>线程的五种状态

###### 新建状态
当用new操作符创建一个线程时。此时程序还没有开始运行线程中的代码。
###### 就绪状态
一个新创建的线程并不自动开始运行，要执行线程，必须调用线程的start()方法。当线程对象调用start()方法即启动了线程，start()方法创建线程运行的系统资源，并调度线程运行run()方法。当start()方法返回后，线程就处于就绪状态。
处于就绪状态的线程并不一定立即运行run()方法，线程还必须同其他线程竞争CPU时间，只有获得CPU时间才可以运行线程。因为在单CPU的计算机系统中，不可能同时运行多个线程，一个时刻仅有一个线程处于运行状态。因此此时可能有多个线程处于就绪状态。对多个处于就绪状态的线程是由Java运行时系统的线程调度程序来调度的。

###### 运行状态（running）
当线程获得CPU时间后，它才进入运行状态，真正开始执行run()方法。
###### 阻塞状态（blocked）
线程运行过程中，可能由于各种原因进入阻塞状态：
1.线程通过调用sleep方法进入睡眠状态；
2.线程调用一个在I/O上被阻塞的操作，即该操作在输入输出操作完成之前不会返回到它的调用者；
3.线程试图得到一个锁，而该锁正被其他线程持有；
4.线程在等待某个触发条件；
所谓阻塞状态是正在运行的线程没有运行结束，暂时让出CPU，这时其他处于就绪状态的线程就可以获得CPU时间，进入运行状态。

###### 死亡状态（dead）
有两个原因会导致线程死亡：
1.run方法正常退出而自然死亡；
2.一个未捕获的异常终止了run方法而使线程猝死；

为了确定线程在当前是否存活着（就是要么是可运行的，要么是被阻塞了），需要使用isAlive方法，如果是可运行或被阻塞，这个方法返回true；如果线程仍旧是new状态且不是可运行的，或者线程死亡了，则返回false。

![线程的状态](http://upload-images.jianshu.io/upload_images/3747483-af72526dc9a8dd54.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
圈1:线程调用start()方法
圈2:该线程抢占到CPU资源
圈3:被别的线程抢到CPU资源
圈4:线程阻塞了，有可能调用了sleep()方法，或者wait()方法，或者正等待一个输入、输出操作，或者需要满足某种条件下才可以继续执行
圈5:输入、输出操作结束，或者解除了wait()或sleep()操作
圈6:线程运行结束

>线程池

###### 创建线程池的四种方式

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
方式一：newSingleThreadExecutor() 初始化的线程池中只有一个线程，如果该线程异常结束，会重新创建一个新的线程继续执行任务，唯一的线程可以保证所提交任务的顺序执行，内部使用LinkedBlockingQueue作为阻塞队列。
```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
方式二：newCachedThreadPool()初始化一个可以缓存线程的线程池，默认缓存60s，线程池的线程数可达到Integer.MAX_VALUE，即2147483647，内部使用SynchronousQueue作为阻塞队列;和newFixedThreadPool创建的线程池不同，newCachedThreadPool在没有任务执行时，当线程的空闲时间超过keepAliveTime，会自动释放线程资源，当提交新任务时，如果没有空闲线程，则创建新线程执行任务，会导致一定的系统开销；
```java
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```
方式三：newFixedThreadPool()初始化一个指定线程数的线程池，其中corePoolSize == maximumPoolSize，使用LinkedBlockingQuene作为阻塞队列，不过当线程池没有可执行任务时，也不会释放线程;
```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {
        ···
public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }

public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue(), threadFactory);
    }
     ···
}
```
方式四：newScheduledThreadPool()初始化的线程池可以在指定的时间内周期性的执行所提交的任务，在实际的业务场景中可以使用该线程池定期的同步数据。

我们从上面的四种创建线程的方式来看，这四种方式最终都是归根到ThreadPoolExecutor的构造方法上，只是相关的一些参数不同而已；
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
![](https://img-blog.csdn.net/20160724192641221)

**注**:

1. 当任务<=核心线程 使用核心线程执行任务

 	2. 当任务>核心线程,队列未满,将任务存放至队列中
 	3. 当任务>核心线程,队列已满,开启新线程执行任务
 	4. 当线程数量>最大线程数量 根据Handler的不同,处理结果不同,默认情况下是

###### corePoolSize

线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。
###### maximumPoolSize
线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于maximumPoolSize；
###### keepAliveTime
线程空闲时的存活时间，即当线程没有任务执行时，继续存活的时间；默认情况下，该参数只在线程数大于corePoolSize时才有用；
###### unit
keepAliveTime的单位；

###### workQueue
用来保存等待被执行的任务的阻塞队列，且任务必须实现Runable接口，在JDK中提供了如下阻塞队列：
1、ArrayBlockingQueue：基于数组结构的有界阻塞队列，按FIFO排序任务；
2、LinkedBlockingQuene：基于链表结构的阻塞队列，按FIFO排序任务，吞吐量通常要高于ArrayBlockingQuene；
3、SynchronousQuene：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene；
4、priorityBlockingQuene：具有优先级的无界阻塞队列；

###### threadFactory
创建线程的工厂，通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名。
###### handler
线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略：
1、AbortPolicy：直接抛出异常，默认策略；
2、CallerRunsPolicy：用调用者所在的线程来执行任务；
3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
4、DiscardPolicy：直接丢弃任务；当然也可以根据应用场景实现RejectedExecutionHandler接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。

>线程同步和同步锁

 因为线程的运行权是通过一种叫抢占的方式获得的。一个程序运行到一半的时候，突然被另一个线程获得了运行，此时这个线程的数据处理了一半，而另一个线程的也在处理这个数据，就会造成数据的混乱，最终导致整个系统的混乱。那么这个时候就需要将线程同步，在Java语言中，解决同步问题的方法有三种，一种是synchrnized锁，另一种是lock锁，还有一种是关键字volatile[ˈvɑ:lətl]。

##### 1.synchronnized锁
synchronized锁有两种实现方式，一种是同步方法，另外一种是同步代码块
 ###### 同步代码块
​     synchronized(锁对象){
​          需要被锁的代码//线程只有拿到了锁对象，才能执行这里的代码！！！换言之，这里的代码如果执行了，说明该线程拿到了锁对象，其他线程不能拿到该锁对象
​             }
​     注意
​          多个线程必须使用同一个锁对象,要不然锁无效

###### 同步方法
​     public synchronized void show(){} //普通方法的锁是this
​     public static synchronized void show(){} //静态方法的锁是当前类的字节码文件对象 类名.class

##### 2.lock锁
Lock和synchronized有一点非常大的不同，采用synchronized不需要用户去手动释放锁，当synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用；而Lock则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象。

lock主要方法：
```
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```
　 下面来讲述Lock接口中=方法的使用，lock()、tryLock()、tryLock(long time, TimeUnit unit)和lockInterruptibly()是用来获取锁的。unLock()方法是用来释放锁的。newCondition()这个方法暂且不讲述。
在Lock中声明了四个方法来获取锁，那么这四个方法有何区别呢？

　　首先lock()方法是平常使用得最多的一个方法，就是用来获取锁。如果锁已被其他线程获取，则进行等待。
　　由于在前面讲到如果采用Lock，必须主动去释放锁，并且在发生异常时，不会自动释放锁。因此一般来说，使用Lock必须在try{}catch{}块中进行，并且将释放锁的操作放在finally块中进行，以保证锁一定被被释放，防止死锁的发生。通常使用Lock来进行同步的话，是以下面这种形式去使用的
```
Lock lock = ...;
lock.lock();
try{
    //处理任务
}catch(Exception ex){
     
}finally{
    lock.unlock();   //释放锁
}
```
tryLock()方法是有返回值的，它表示用来尝试获取锁，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false，也就说这个方法无论如何都会立即返回。在拿不到锁时不会一直在那等待。

tryLock(long time, TimeUnit unit)方法和tryLock()方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。

所以，一般情况下通过tryLock来获取锁时是这样使用的：
```
Lock lock = ...;
if(lock.tryLock()) {
     try{
         //处理任务
     }catch(Exception ex){
         
     }finally{
         lock.unlock();   //释放锁
     } 
}else {
    //如果不能获取锁，则直接做其他事情
}
```
lockInterruptibly()方法比较特殊，当通过这个方法去获取锁时，如果线程正在等待获取锁，则这个线程能够响应中断，即中断线程的等待状态。也就使说，当两个线程同时通过lock.lockInterruptibly()想获取某个锁时，假若此时线程A获取到了锁，而线程B只有在等待，那么对线程B调用threadB.interrupt()方法能够中断线程B的等待过程。

由于lockInterruptibly()的声明中抛出了异常，所以lock.lockInterruptibly()必须放在try块中或者在调用lockInterruptibly()的方法外声明抛出InterruptedException。
因此lockInterruptibly()一般的使用形式如下：
```
public void method() throws InterruptedException {
    lock.lockInterruptibly();
    try {  
     //.....
    }
    finally {
        lock.unlock();
    }  
}
```
注意，当一个线程获取了锁之后，是不会被interrupt()方法中断的。
因此当通过lockInterruptibly()方法获取某个锁时，如果不能获取到，只有进行等待的情况下，是可以响应中断的。而用synchronized修饰的话，当一个线程处于等待某个锁的状态，是无法被中断的，只有一直等待下去。
##### 3.volatile
volatile作者在开发过程中没有用到过，但源码里好多地方用到，今天查资料，才发现也可以实现同步，就小小的总结下：

我们知道，在创建线程的时候，会给线程分配一些堆栈内存的，那么线程中在操作主存的变量的时候，是怎样操作的呢？是先将主存变量在线程的分配的内存中写一个副本，操作完这个副本之后在，将最新的值传给主存，从而改变变量。

而对volatile修饰的变量，具有一个特性：可以保证任何一个线程在写这个变量副本的时候，是最新的，就算别的线程将这个变量已经修改过，也会通过一些机制，将主存的这个变量保证最新。

###### 根据以上的描述，我们总结一下Lock和synchronized的不同：
1）Lock是一个接口，而synchronized是Java中的关键字，synchronized是内置的语言实现；
2）synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；
3）Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；
4）通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。
5）Lock可以提高多个线程进行读操作的效率。（有具体的实现）
在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时（即有大量线程同时竞争），此时Lock的性能要远远优于synchronized。所以说，在具体使用时要根据适当情况选择

>死锁

##### 死锁产生的条件：
互斥条件：一个资源每次只能被一个线程使用。
请求与保持条件：一个线程因请求资源而阻塞时，对已获得的资源保持不放。
不剥夺条件：线程已获得的资源，在未使用完之前，不能强行剥夺。
循环等待条件：若干线程之间形成一种头尾相接的循环等待资源关系。

总归，死锁就是因为同步引起的，你不同步啥事没有，直接原因就是同步循环嵌套，你等我释放，我等你释放，这就好比两人吃西餐，一个人拿着刀，另一个人拿着叉，都想吃那块鲜牛肉，你等着我给你刀，我等着你给我叉，到最后大家都吃不到，这就变成了死锁。

避免方法：
1.有多个同步嵌套的，按照相同的顺序来获得锁
2.尽量不要在同步方法里，调用外部方法