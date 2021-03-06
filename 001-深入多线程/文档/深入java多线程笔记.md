# jdk源码&多线程&高并发-【阶段1、深入多线程】

[toc]

**码炫课堂讲师简介**

smart哥，互联网悍将，历经从传统软件公司到大型互联网公司的洗礼，入行即在中兴通讯等大型通信公司担任项目leader，后随着互联网的崛起，先后在美团支付等大型互联网公司担任架构师，公派旅美期间曾与并发包大神Doug Lea探讨java多线程等最底层的核心技术。对互联网架构底层技术有相当的研究和独特的见解，在多个领域有着丰富的实战经验。



## 一、java多线程实现及API详解

### 1、进程 线程 协程的概念和区别



#### **1）什么是进程和线程？**

进程是应用程序的启动实例，进程拥有代码和打开的文件资源、数据资源、独立的内存空间。

线程从属于进程，是程序的实际执行者，一个进程至少包含一个主线程，也可以有更多的子线程，线程拥有自己的栈空间。

![img](https:////upload-images.jianshu.io/upload_images/4933701-4dfd867ca99f40d7.png?imageMogr2/auto-orient/strip|imageView2/2/w/646/format/webp)

**操作系统中的进程和线程**

对操作系统而言，**线程是最小的执行单元，进程是最小的资源管理单元**。无论是进程还是线程，都是由操作系统所管理的。

**线程的状态**

线程具有五种状态：初始化、可运行、运行中、阻塞、销毁

![img](https:////upload-images.jianshu.io/upload_images/4933701-167191f7722edd23.png?imageMogr2/auto-orient/strip|imageView2/2/w/653/format/webp)

线程状态的转化关系

new Thread（）.start（）                                                             run(){ thread.sleep(3000);} 



**进程与线程的区别**

1. 进程是CPU资源分配的基本单位，线程是独立运行和独立调度的基本单位（CPU上真正运行的是线程）。
2. 进程拥有自己的资源空间，一个进程包含若干个线程，线程与CPU资源分配无关，多个线程共享同一进程内的资源。
3. 线程的调度与切换比进程快很多。



经验：

**CPU密集型代码(各种循环处理、计算等等)：使用多进程。**很少跟硬盘打交道，io少，阻塞少了

**IO密集型代码(文件处理、网络爬虫等)：使用多线程**。跟硬盘，网络打交道比较多，硬盘io，网络io比较多



#### **2）线程之间是如何进行协作的呢？**

最经典的例子是生产者/消费者模式，即若干个生产者线程向队列中生产数据，若干个消费者线程从队列中消费数据。

![img](https:////upload-images.jianshu.io/upload_images/4933701-5ecd0e299591dcfe.png?imageMogr2/auto-orient/strip|imageView2/2/w/429/format/webp)

生产者/消费者模式

生产者/消费者模式的性能问题是什么？

- 涉及到同步锁
- 涉及到线程阻塞状态和可运行状态之间的切换
- 设置到线程上下文的切换

#### **3）什么是协程？**

协程（Coroutines）是一种比线程更加轻量级的存在，正如一个进程可以拥有多个线程一样，一个线程可以拥有多个协程。

![img](https:////upload-images.jianshu.io/upload_images/4933701-4a7846c5d7c1290c.png?imageMogr2/auto-orient/strip|imageView2/2/w/646/format/webp)

**操作系统中的协程**

协程不是被操作系统内核所管理的，而是完全由程序所控制，也就是在用户态执行。这样带来的好处是性能大幅度的提升，因为不会像线程切换那样消耗资源。

协程不是进程也不是线程，而是一个特殊的函数，这个函数可以在某个地方挂起，并且可以重新在挂起处外继续运行。所以说，协程与进程、线程相比并不是一个维度的概念。

一个进程可以包含多个线程，一个线程也可以包含多个协程。简单来说，一个线程内可以由多个这样的特殊函数在运行，但是有一点必须明确的是，一个线程的多个协程的运行是串行的。如果是多核CPU，多个进程或一个进程内的多个线程是可以并行运行的，但是一个线程内协程却绝对是串行的，无论CPU有多少个核。毕竟协程虽然是一个特殊的函数，但仍然是一个函数。一个线程内可以运行多个函数，但这些函数都是串行运行的。当一个协程运行时，其它协程必须挂起。

#### **4）进程、线程、协程的对比**

- 协程既不是进程也不是线程，协程仅仅是一个特殊的函数，协程与进程和线程不是一个维度的。
- 一个进程可以包含多个线程，一个线程可以包含多个协程。
- 一个线程内的多个协程虽然可以切换，但是多个协程是串行执行的，只能在一个线程内运行，没法利用CPU多核能力。
- 协程与进程一样，切换是存在上下文切换问题的。

**上下文切换**（画图）

- 进程的切换者是操作系统，切换时机是根据操作系统自己的切换策略，用户是无感知的。进程的切换内容包括页全局目录、内核栈、硬件上下文，切换内容保存在内存中。进程切换过程是由“用户态到内核态到用户态”的方式，切换效率低。
- 线程的切换者是操作系统，切换时机是根据操作系统自己的切换策略，用户无感知。线程的切换内容包括内核栈和硬件上下文。线程切换内容保存在内核栈中。线程切换过程是由“用户态到内核态到用户态”， 切换效率中等。
- 协程的切换者是用户（编程者或应用程序），切换时机是用户自己的程序所决定的。协程的切换内容是硬件上下文，切换内存保存在用户自己的变量（用户栈或堆）中。协程的切换过程只有用户态，即没有陷入内核态，因此切换效率高。



需求：100w的线程和100w的协程，同时做运算，处理200w次运算处理

java-agent

#### 5）线程vs协程性能对比

java线程案例

~~~java
package com.mx.lang.Object;

public class JavaThread {


    /**
     * 10w个线程，每个线程处理2百万次运算
     * @param argus
     * @throws InterruptedException
     */
    public static void main(String[] argus) throws InterruptedException {
        long begin = System.currentTimeMillis();
        int threadLength = 100000;//10w
        Thread[] threads = new Thread[threadLength];
        for (int i = 0; i < threadLength; i++) {
            threads[i] = new Thread(() -> {
                calc();
            });
        }


        for (int i = 0; i < threadLength; i++) {
            threads[i].start();
        }
        for (int i = 0; i < threadLength; i++) {
            threads[i].join();
        }
        System.out.println(System.currentTimeMillis() - begin);
    }


    //200w次计算
    static void calc() {
        int result = 0;
        for (int i = 0; i < 10000; i++) {
            for (int j = 0; j < 200; j++) {
                result += i;
            }
        }
    }
}
~~~

java协程案例

~~~java
package com.mx.lang.Object;

import co.paralleluniverse.fibers.Fiber;

import java.util.concurrent.ExecutionException;

public class JavaFiber {
    /**
     * 10w个协程，每个协程处理2百万次运算
     * @param argus
     * @throws InterruptedException
     */
    public static void main(String[] argus) throws ExecutionException, InterruptedException {
        long begin = System.currentTimeMillis();
        int fiberLength = 100000;//10w
        Fiber<Void>[] fibers = new Fiber[fiberLength];
        for (int i = 0; i < fiberLength; i++) {
            fibers[i] = new Fiber(() -> {
                calc();
            });
        }


        for (int i = 0; i < fiberLength; i++) {
            fibers[i].start();
        }
        for (int i = 0; i < fiberLength; i++) {
            fibers[i].join();
        }
        System.out.println(System.currentTimeMillis() - begin);
    }

    //200w次计算
    static void calc() {
        int result = 0;
        for (int i = 0; i < 10000; i++) {
            for (int j = 0; j < 200; j++) {
                result += i;
            }
        }
    }
}
~~~

查看jvm此时有多少线程？

10w协程分配给3个线程

***结论：25w个协程共用一个线程（一个线程中跑多个协程，协程不需要调度，对内核透明），4个线程一个100w个协程。***



***上下文切换是有成本的。因此，思考一个问题：多线程一定快吗？***

代码演示：

~~~java
package com.mx.lang.Object;

import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

/**
 * 验证多线程是否就一定快
 */
public class ConcurrencyTest {
    private static final long count = 1000000000l;

    public static void main(String[] args) throws InterruptedException {
        concurrency();
        serial();
    }

    private static void concurrency() throws InterruptedException {
        long start = System.currentTimeMillis();
//        Thread thread = new Thread(new Runnable() {
//            @Override
//            public void run() {
//                int a = 0;
//                for (long i = 0; i < count; i++) {
//                    a += 5;
//                }
//            }
//        });

        FutureTask task=new FutureTask<Integer>(new Callable<Integer>(){
            @Override
            public Integer call() throws Exception {
                int a = 0;
                for (long i = 0; i < count; i++) {
                    a += 5;
                }
                return a;
            }
        });

        Thread thread2 = new Thread(task);
        thread2.start();
        int b = 0;
        for (long i = 0; i < count; i++) {
            b--;
        }
        //两种方式计算才准：1、用join 2、task.get() 区别是计算总时间放的位置
//        thread2.join();
//        long time = System.currentTimeMillis() - start;
        try {
            System.out.println("b=" + b + ",a=" + task.get() );
            long time = System.currentTimeMillis() - start;
            System.out.println("concurrency :" + time + "ms");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void serial() {
        long start = System.currentTimeMillis();
        int a = 0;
        for (long i = 0; i < count; i++) {
            a += 5;
        }
        int b = 0;
        for (long i = 0; i < count; i++) {
            b--;
        }
        long time = System.currentTimeMillis() - start;
        System.out.println("serial:" + time + "ms,b=" + b + ",a=" + a);
    }
}
~~~



### 2、案例：数据迁移

需求：多线程实现表数据迁移（画图说明）

图：![数据迁移案例](E:\mypro\imgbed\myimgs\jthread\数据迁移案例.jpg)



#### 1）jvm进程被拉起后里面有哪些线程？分别是有什么作用？

~~~java
 /**
  * 获取所有线程
  */
Map<Thread, StackTraceElement[]> allStackTraces = Thread.getAllStackTraces();

Set<Thread> threads = allStackTraces.keySet();
for (Thread thread : threads) {
    System.out.println("线程名：" + thread.getName());
}
~~~

结果如下：

~~~shell
线程名：Attach Listener
线程名：Finalizer
线程名：Reference Handler
线程名：Signal Dispatcher
线程名：Monitor Ctrl-Break
线程名：main

~~~



说明：Monitor Ctrl-Break，官方解释：

> ```
> Monitoring Thread Activity With Thread Dumps Thread dumps,
> or "thread stack traces," reveal information about
> an application's activity that can help you diagnose problems
> and better optimize application and JVM performance;
> for example, thread dumps can show the occurrence of "deadlock" conditions,
> which can seriously impact application performance.
> You can create a thread dump by invoking a control break
> (usually by pressing Ctrl-Break or Ctrl-\ or SIGQUIT on linux).
> This section provides information on working with thread dumps.
> It includes information on these subjects:
> 1.Lock Information in Thread Dumps 2.Detecting Deadlocks
> ```



#### 2）JVM源码跟踪debug分析main方法一般流程

**面试题：main方法只能是main线程调用吗？main方法只能由JVM调用吗？**

代码演示：

~~~java
public class TestMain {
    public static void main(String[] args) {
        DataTransport transport = new DataTransport();
        transport.main(new String[]{"maxuan"});
    }
}
~~~



结论：应该是考线程的知识，main线程其实是jvm给你启动的，因为main方法是入口，所以得有个原始线程去调他，此时其实是jvm给你创建的main线程。其实main方法没那么神秘，任何线程都可以去调他，只不过其他线程的入口是哪里呢？其实他也有自己的main线程入口。



思考问题：

**JVM是如何调用main方法，是如何调用main方法的呢？**

java当中的线程和操作系统的线程是什么关系？
猜想：（c++） java thread —-对应-—> **OS thread** ---原生线程（**hotspot**）ibm j9，ali 华为

GDB



JavaCalls::call_special—》java侧的main线程



演示代码：

~~~java
package com.mx.lang.Thread;

/***************************************
 *
 * jdk源码&多线程&高并发-【阶段1、深入多线程】
 * 主讲：smart哥
 *
 ***************************************/
public class TestMain {
    public static void main(String[] args) {
//        DataTransport transport = new DataTransport();
//        transport.main(new String[]{"maxuan"});
//        Thread
        System.out.println(">>>>>>in main:"+Thread.currentThread().getName());
        TestMain testMain=new TestMain();
        Thread.currentThread().setName("main-name");
        System.out.println(">>>>>>>after set name in main:"+Thread.currentThread().getName());
        testMain.startThread();
    }

    private void startThread() {
        Thread t= new Thread(new Runnable() {
            @Override
            public void run() {
                int i=0;
                while(i<3){
                    System.out.println(">>>>>>in thread:"+Thread.currentThread().getName());
                    i++;
                }
                i++;
                Thread.currentThread().setName("aaaa");
                System.out.println(">>>>>>in thread,name:"+Thread.currentThread().getName());
            }});
        t.setName("bbbb");
        t.start();
    }
}
~~~



具体演示见视频讲解



**JVM源码的角度分析main方法一般流程：**

1、加载虚拟机（检查库文件libjvm.so中是否方法都已经准备好）：LoadJavaVM(jvmpath, &ifn)。

2、ok，准备好了，创建新线程：**pthread_create**，以下都是在这个线程中做的，本质是调的JavaMain方法

   **1） --》创建虚拟机：InitializeJVM(&vm, &env, &ifn)**

​           ---》 ifn->CreateJavaVM(pvm, (void **)penv, &args);（jni.cpp的JNI_CreateJavaVM方法）

​             ----》创建虚拟机的过程中创建main线程：JavaThread* main_thread = new JavaThread();(注意和普通线                      						程的区别)。

​                     main_thread->set_as_starting_thread()（真正和内核线程关联的是这句代码）

​                     JavaCalls::call_special—》映射java侧的main线程

   **2）--》mainClass = LoadMainClass(env, mode, what);**

   **3）--》调用main方法： (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);**



流程图：

![java中main方法执行流程](E:\mypro\imgbed\myimgs\jthread\java中main方法执行流程.png)



原图：http://huatu.xuanheng05.cn/lct/#R4a768a9e0da6b8dafa58ad76b8680bd2



 结论：java Thread：os thread=1：1

io多路复用：epoll ，poll，nio native



#### 3）线程改造数据迁移案例

面试题：java中实现实现线程有几种方式？

1、继承Thread类

2、实现Runnable接口

3、实现Callable接口（拿回返回值，FutuTask）

4、线程池



**方式一：继承Thread类的方式**

1. 创建一个继承于Thread类的子类
2. 重写Thread类中的run()：将此线程要执行的操作声明在run()
3. 创建Thread的子类的对象
4. 调用此对象的start():①启动线程 ②调用当前线程的run()方法

**方式二：实现Runnable接口的方式**

1. 创建一个实现Runnable接口的类
2. 实现Runnable接口中的抽象方法：run():将创建的线程要执行的操作声明在此方法中
3. 创建Runnable接口实现类的对象
4. 将此对象作为参数传递到Thread类的构造器中，创建Thread类的对象
5. 调用Thread类中的start():① 启动线程 ② 调用线程的run() --->调用Runnable接口实现类的run()

以下两种方式是jdk1.5新增的！

**方式三：实现Callable接口**

说明：

1. 与使用Runnable相比， Callable功能更强大些
2. 实现的call()方法相比run()方法，可以返回值
3. 方法可以抛出异常
4. 支持泛型的返回值
5. 需要借助FutureTask类，比如获取返回结果

- Future接口可以对具体Runnable、Callable任务的执行结果进行取消、查询是否完成、获取结果等。
- FutureTask是Futrue接口的唯一的实现类
- FutureTask 同时实现了Runnable, Future接口。它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值



方式3和方式2的区别就是有些场景需要有返回值，此时方式3派上用场



**方式四：使用线程池**

说明：

- 提前创建好多个线程，放入线程池中，使用时直接获取，使用完放回池中。可以避免频繁创建销毁、实现重复利用。类似生活中的公共交通工具。

好处：

1.  提高响应速度（减少了创建新线程的时间）
2.  降低资源消耗（重复利用线程池中线程，不需要每次都创建）
3.  便于线程管理



~~~java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;
import java.util.concurrent.ThreadPoolExecutor;

//方式一
class ThreadTest extends Thread {
	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			System.out.println(Thread.currentThread().getName() + ":" + i);
		}
	}
}

// 方式二
class RunnableTest implements Runnable {
	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			System.out.println(Thread.currentThread().getName() + ":" + i);
		}
	}
}

// 方式三
class CallableTest implements Callable<Integer> {

	@Override
	public Integer call() throws Exception {
		int sum = 0;
		for (int i = 0; i < 10; i++) {
			System.out.println(Thread.currentThread().getName() + ":" + i);
			sum += i;
		}
		return sum;
	}

}

// 方式四
class ThreadPool implements Runnable {

	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			System.out.println(Thread.currentThread().getName() + ":" + i);
		}
	}

}

public class Test {
	public static void main(String[] args) {
		// 继承Thread
		ThreadTest thread = new ThreadTest();
		thread.setName("方式一");
		thread.start();

		// 实现Runnable
		RunnableTest runnableTest = new RunnableTest();
		Thread thread2 = new Thread(runnableTest, "方式二");
		thread2.start();

		// 实现Callable<> 有返回值
		CallableTest callableTest = new CallableTest();
		FutureTask<Integer> futureTask = new FutureTask<>(callableTest);
		new Thread(futureTask, "方式三").start();
		// 返回值
		try {
			Integer integer = futureTask.get();
			System.out.println("返回值（sum）：" + integer);
		} catch (Exception e) {
			e.printStackTrace();
		}

		// 线程池
		ExecutorService pool = Executors.newFixedThreadPool(10);

		ThreadPoolExecutor executor = (ThreadPoolExecutor) pool;
		/*
		 * 可以做一些操作:
		 * corePoolSize：核心池的大小 
		 * maximumPoolSize：最大线程数
		 * keepAliveTime：线程没任务时最多保持多长时间后会终止
		 */
		executor.setCorePoolSize(5);

		// 开启线程
		executor.execute(new ThreadPool());
		executor.execute(new ThreadPool());
		executor.execute(new ThreadPool());
		executor.execute(new ThreadPool());

	}

}
~~~

#### 4）消费者线程安全改造分析

1）用synchronized和原子类的区别

2）synchronized放置位置的不同会引起错误的执行结果

详见视频

3）线程之间的状态及转换

![image-20201203132842569](E:\mypro\imgbed\myimgs\jthread\image-20201203132842569.png)



#### 5）线程状态转换验证

~~~java
package com.mx.lang.Thread.threadstate;

/***************************************
 *
 * jdk源码&多线程&高并发-【阶段1、深入多线程】
 * 主讲：smart哥
 *
 ***************************************/
public class MyLock {
   static Object obj= new Object();
}
~~~



~~~java
package com.mx.lang.Thread.threadstate;

/***************************************
 *
 * jdk源码&多线程&高并发-【阶段1、深入多线程】
 * 主讲：smart哥
 *
 ***************************************/
public class MyThread extends Thread{

    /**
     * NEW  创建线程，未启动
     * RUNNABLE
     * BLOCKED---------------------
     * WAITING---------------------object.wait
     * TIMED_WAITING
     * TERMINATED
     */
    public MyThread() {
        System.out.println("this thread state:"+this.getState());//NEW
    }

    @Override
    public void run() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("this thread in run:"+this.getState());//RUNNABLE
    }
}
~~~

~~~java
package com.mx.lang.Thread.threadstate;

/***************************************
 *
 * jdk源码&多线程&高并发-【阶段1、深入多线程】
 * 主讲：smart哥
 *
 ***************************************/
public class MyThreadForBlock1 extends Thread {

    /**
     * NEW  创建线程，未启动
     * RUNNABLE
     * BLOCKED---------------------
     * WAITING---------------------object.wait
     * TIMED_WAITING
     * TERMINATED
     */
//    Object obj = new Object();

    @Override
    public void run() {
        //进入run和抢到锁直接有时间差
        synchronized (MyLock.obj) {
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

~~~



~~~java
package com.mx.lang.Thread.threadstate;

/***************************************
 *
 * jdk源码&多线程&高并发-【阶段1、深入多线程】
 * 主讲：smart哥
 *
 ***************************************/
public class MyThreadForBlock2 extends Thread {

    /**
     * NEW  创建线程，未启动
     * RUNNABLE
     * BLOCKED---------------------
     * WAITING---------------------object.wait
     * TIMED_WAITING
     * TERMINATED
     */
//    Object obj = new Object();

    @Override
    public void run() {
        synchronized (MyLock.obj) {
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

~~~



~~~java
package com.mx.lang.Thread.threadstate;

/***************************************
 *
 * jdk源码&多线程&高并发-【阶段1、深入多线程】
 * 主讲：smart哥
 *
 ***************************************/
public class MyThreadForWait extends Thread {

    /**
     * NEW  创建线程，未启动
     * RUNNABLE
     * BLOCKED---------------------
     * WAITING---------------------object.wait
     * TIMED_WAITING
     * TERMINATED
     */
    Object obj = new Object();

    @Override
    public void run() {
        synchronized (obj){
            try {
                obj.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

~~~



~~~java
package com.mx.lang.Thread.threadstate;

/***************************************
 *
 * jdk源码&多线程&高并发-【阶段1、深入多线程】
 * 主讲：smart哥
 *
 ***************************************/
public class StateTest {
    public static void main(String[] args) {
//        MyThread myThread = new MyThread();
//        System.out.println("in main the thread state1 is:"+myThread.getState());
//        try {
//            Thread.sleep(1000);
//        } catch (InterruptedException e) {
//            e.printStackTrace();
//        }
//        myThread.start();
//        try {
//            TimeUnit.SECONDS.sleep(1);
//            System.out.println("in main the thread state2 is:"+myThread.getState());
//        } catch (InterruptedException e) {
//            e.printStackTrace();
//        }
//        System.out.println("in main the thread state3 is:"+myThread.getState());

        //测试waitting状态
//        MyThreadForWait thread = new MyThreadForWait();
//        thread.start();
//        System.out.println("in main before sleep the thread state is:"+thread.getState());
//
//        try {
//            Thread.sleep(1000);
//        } catch (InterruptedException e) {
//            e.printStackTrace();
//        }
//        System.out.println("in main after sleep the thread state is:"+thread.getState());

        //测试block状态
        MyThreadForBlock1 myThreadForBlock1 = new MyThreadForBlock1();
        MyThreadForBlock2 myThreadForBlock2 = new MyThreadForBlock2();
        myThreadForBlock1.start();
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        myThreadForBlock2.start();
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("myThreadForBlock1 state is:"+myThreadForBlock1.getState());
        System.out.println("myThreadForBlock2 state is:"+myThreadForBlock2.getState());
    }
}

~~~



#### 6）JVM源码debug分析线程start0方法

面试题：Thread类的start0方法究竟发生了什么？为什么执行start方法之后，系统会自动执行run方法？

猜想：在jvm层面启动一个原生线程，然后原生线程置为RUNNABLE状态，然后回调java中的run方法

结论：

1、在jvm层面new thread之后（此时状态是initialized状态），然后会一直while死循环判断，判断如果是非initialized状态，才会继续往下执行run方法（等待调用start方法之后，讲状态改为runnable）；

2、执行start方法，修改线程状态为runnable，然后第1步，开始放行，执行run方法（回调java层面的run）

详细分析见视频



![image-20201203132842569](E:\mypro\imgbed\myimgs\jthread\java中Thread的start0方法源码流程.jpg)



流程图详见：http://huatu.xuanheng05.cn/lct/#R451fb285bcd8ce066b090823fec6df85



#### 7）jvm中手写自定义java 线程对应的原生线程，并实现java侧run方法回调

![image-20201210125441510](E:\mypro\imgbed\myimgs\jthread\image-20201210125441510.png)



详细讲解见视频

#### 8）native方法类比总结

![native方法类比总结](E:\mypro\imgbed\myimgs\jthread\native方法类比总结.png)

具体讲解见视频

### 3、线程API详解



#### 1）构造方法解析演示



#### 2）线程组



#### 3）如何防止第三方系统在使用线程中搞破坏？

```java
Thread(Runnable target, AccessControlContext acc)
```



#### 4）线程和stacksize的关系

栈容量：默认值是多少？？？

栈深度（增大单个栈帧的大小，栈深度变小）

1）试验：默认栈容量下的栈深度和调整局部变量后的栈深度（*JTI*编译器优化了，局部变量不使用的话，不会分配内存）

1）-Xss的影响（分别在main和thread上的区别）

2）-Xss的最大最小值（win上和linux上），为什么是236k，有的博客是228k，源码解析

3）linux上默认栈容量为1M？源码解析

4）比较win和linux上的栈深度

#### 5）线程优先级

1）本质上就是调的操作系统函数：setpriority

setpriority, pthread_create, pthread_join

优先级只是让操作系统和cpu尽量优先分配资源，但并不能用来控制执行顺序。



#### 6）守护线程

1）守护线程（能不能把main线程设置成守护线程？）

2）jvm源码解析守护线程为什么在jvm退出时退出----

wait,notify

3）守护线程场景：Java获取系统内存、CPU、磁盘等信息

4）补充：问题：DestroyJavaVM是守护线程吗？

5）能不能救守护线程一命？



#### 7）hook（钩子）线程

1）为什么要有钩子线程？

2）钩子线程与destroyJavaVM线程的关系？

3）钩子线程在哪个阶段被执行？

4）可以有多个hook线程吗？

5）源码解析

6）使用场景：一个完整的钩子线程的运用案例（演示kill -9）



比如有一个任务，一直在运行，如果说发生异常（main线程，main结束），

1、此时需要上报异常，邮件通知相关人员

2、此时需要释放资源，网络，数据库连接资源等等



方案：加两个钩子



#### 8、线程出现异常的处理

1）try。。catch。。在run方法外能否获取线程异常？

2）如何在外面拿到异常？

3）setDefaultUncaughtExceptionHandler 设置的一定会被调用到么？

4）改变异常处理行为（查看源码）

5）异常处理嵌入点及异常处理的优先级（查看源码分析）



## 二、线程间通信及并发

 方法间通信

int method1(){

int i;

........

return  i;

}

String method2(int a){

}

void method3(){

​       int x= method1();

​       method2(x);

}

问题：线程间通信？？

共享变量 int i;

new Thread(()->{

​    //业务逻辑

   i;

});

new Thread(()->{

​    //业务逻辑

​    i;

});

前提：线程之间如何通信及线程间如何同步？

方法之间可以通过传递参数的形式进行通信，那么线程之间怎么传递参数呢？

1、共享内存和消息传递

2、演示各种诡异现象

思考：什么时候线程会重新读取主存共享变量？



### 1、线程并发安全机制

#### 1）volatile机制

马姓

正确理解volatile的姿势：单cpu架构->cpu多级cache结构 -> **缓存一致性协议（MESI）-> store buffer和invalidate queue引入 ->造成mesi协议不一致了**-> 内存屏障-->volatile-->mesi协议再次一致

cpu芯片，非开源



x86没有invalid queue

只有storebuffer



- **cpu多级cache结构** vs **JMM模型**

  - 图解 cpu多级缓存架构

    高级语言（java）--》字节码（jvm识别）-->汇编指令（硬件cpu）

    mov

    jap

    add 0x12333 i

    RAM (主内存)------》cpu（寄存器）0-1ns

    100-120个cpu周期

    -----------------

    cpu 24亿/s（寄存器，storebuffer）

    cache（l1，l2，l3）

    RAM 7%,内存访问时间 80-100ns，1s/100ns=1000w

    硬盘 10ms，100/s

    -----------------------------

    应用，软件领域，来自于硬件

    application

    redis

    db

    

    - 为什么需要cpu高速缓存？

      ![img](E:\mypro\imgbed\myimgs\jthread\1322310.png)

    - 为什么需要cpu多级缓存？

      随着发展，cpu速度和内存速度差距越来越大，所以引入多级缓存

      

  - 图解 jmm模型架构

    

  ![image-20210219172503134](E:\mypro\imgbed\myimgs\jthread\image-20210219172503134.png)

  

  

  - Java内存模型与硬件内存架构的关系

  ![image-20210219164910017](E:\mypro\imgbed\myimgs\jthread\image-20210219164910017.png)

  

  - JMM存在的必要性(JMM8种同步操作和cpu的关系)-------重点

  ![7](E:\mypro\imgbed\myimgs\jthread\7.png)

  

  关于主内存与工作内存之间的交互协议，即一个变量如何从主内存拷贝到工作内存。如何从工作内存同步到主内存中的实现细节。java内存模型定义了8种操作来完成。这8种操作每一种都是原子操作。8种操作如下：

  - lock(锁定)：作用于主内存，它把一个变量标记为一条线程独占状态；
  - read(读取)：作用于主内存，它把变量值从主内存传送到线程的工作内存中，以便随后的load动作使用；
  - load(载入)：作用于工作内存，它把read操作的值放入工作内存中的变量副本中；
  - **use(使用)：作用于工作内存，它把工作内存中的值传递给执行引擎，每当虚拟机遇到一个需要使用这个变量的指令时候，将会执行这个动作；**
  - **assign(赋值)：作用于工作内存，它把从执行引擎获取的值赋值给工作内存中的变量，每当虚拟机遇到一个给变量赋值的指令时候，执行该操作；**
  - store(存储)：作用于工作内存，它把工作内存中的一个变量传送给主内存中，以备随后的write操作使用；
  - write(写入)：作用于主内存，它把store传送值放到主内存中的变量中。
  - unlock(解锁)：作用于主内存，它将一个处于锁定状态的变量释放出来，释放后的变量才能够被其他线程锁定；

  Java内存模型还规定了执行上述8种基本操作时必须满足如下规则:

  （1）不允许read和load、store和write操作之一单独出现（即不允许一个变量从主存读取了但是工作内存不接受，或者从工作内存发起会写了但是主存不接受的情况），以上两个操作必须按顺序执行，但没有保证必须连续执行，也就是说，read与load之间、store与write之间是可插入其他指令的。

  （2）不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。

  （3）不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中。

  （4）一个新的变量只能从主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。

  （5）一个变量在同一个时刻只允许一条线程对其执行lock操作，但lock操作可以被同一个条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。

  （6）如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。

  （7）如果一个变量实现没有被lock操作锁定，则不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量。

  （8）对一个变量执行unlock操作之前，必须先把此变量同步回主内存（执行store和write操作）。

  

- **cpu缓存一致性协议（mesi协议详解）**

  - cacheline(cache block)结构与原理

    - cacheline基础

      1）什么是cacheline？

      ​      cache block

      

      2）图解cacheline结构

      cache line与内存之间的对应关系

      <1>、Direct Mapped Cache

      

      ![img](E:\mypro\imgbed\myimgs\jthread\v2-7682455a52c19fc023b3190bdff67e61_720w.jpg)

      

      <2>、Two-way Set Associative Cache

      ![image-20210225104121219](E:\mypro\imgbed\myimgs\jthread\image-20210225104121219-1620560564727.png)

      

      

      ![img](E:\mypro\imgbed\myimgs\jthread\2019062318525615.png)

      

      

      3）cpu与cache， 内存交互的过程

      

      ​     Cache写机制：

      ​           缓存命中

      ​				write through------------直写，写通

      ​                修改cache之后，立马写回主存

      

      ​				**write back-----------------回写**

      ​                 **修改完之后，不立即写入主存，而是等到一定的时候，触发一次回写**

      ​        缓存未命中

      ​             write allocate

      ​              

      ​     cache替换策略：

      ​          1、LFU(least frequence used 最不经常使用)

      ​          **2、LRU（least recently used 近期最少使用） 算法复杂，但是命中率高，90%**

      ​         3、随机替换

        

      4）cacheline的两个局部性约束：时间局部性和空间局部性

         

    - cacheline的伪共享问题及解决方案

    ​        如何保证？

    ​       1、通过class的层级padding（jdk8之前推荐使用）

    ​       2、自动padding

    ​       

    ​         伪共享很容易出现吗？什么情况下出现几率大？

    ​         TLAB=thread local allocate buffer

    ​           

    - 实践案例分析

      

      1、ConcurrentHashmap

      2、thread

      [netty中的伪共享]: https://github.com/netty/netty/blob/netty-4.1.48.Final/common/src/main/java/io/netty/util/internal/InternalThreadLocalMap.java#L125

      

      

    - Java中对Cache line经典设计（Disruptor框架）(放到第4或者5阶段)

    ​       更juc包放一起讲

  

  - MESI状态

  ​       M:modified

  ​            cpu拥有cacheline，并且做了修改，但是修改值还没有刷新到主内存

  ​      E: exclusive

  ​            cpu拥有cacheline，但是还没有做修改。

  ​         M，E 重要特性：在所有cpu的cache中，只有唯一的一个cacheline是M或者E状态

  ​    

  ​     S：shared

  ​         所有cpu的cache都对某一个cacheline拥有read，但是不能写

  

  ​     I：无效，失效

  ​          当前某个cacheline数据无效，也就相当于里面没有数据

  

  - MESI协议消息

  ​        read：就是请求数据，读取一个物理内存地址上的数据，把消息通过总线广播，然后等待回应

  ​        read response：包括read请求的数据，这个数据可能来源于其他cpu，也可能来源于主内存

  ​       **invalidate**：包含一个物理地址，告知其他cpu，把cache中的这个地址所对应的cacheline置为无效

  ​       invalidate acknowledge：置无效之后的反馈信息

  ​      

  ​      **read invalidate**：read+invalidate，这种对应cacheline里没有响应的变量的时候

  ​      writeback：回写数据，回写到主内存，m

  

  ![image-20210302160204183](E:\mypro\imgbed\myimgs\jthread\image-20210302160204183-1620560564727.png)

  

  - MESI状态切换

    ![image-20210304135923940](E:\mypro\imgbed\myimgs\jthread\image-20210304135923940-1620560564728.png)

  

  - MESI协议案例分析

    mesi协议动态演示

    https://www.scss.tcd.ie/Jeremy.Jones/VivioJS/caches/MESIHelp.htm

    用的MESI的变种

  ​     x86-MESIF协议

  ​    arm-MEOSI协议

  

  - 体验一个美团面试题

  ​       可见性怎么发生？

  

  

- **早期mesi协议性能低下，于是引入store buffer**

  - store buffer的引入

    消息队列是一样的

  ​       目的：提高效率

  

    **可见性的本质：就是队列的异步化**

  

  - store forwarding机制

  ​       如果指令之间具有依赖性，所以不会重排

  

  - **Store Buffers和内存屏障机制**

  ​      cpu工程师给了应用工程师解决方案：加内存屏障

  ​      本质：串行化操作

  

  - **手动加入写屏障案例演示**

  ​    见视频讲解

  

- **store buffer的引入导致不必要的延迟，于是引入invalidate queue**

  - invalidate queue的引入

  ​        为了ack消息立即回复

  

  - Invalidate Queues带来的问题

  ​       读到脏数据

  

  - **invalidate queues和内存屏障机制**

  ​       加读屏障

  

  - **手动加入读屏障案例演示**

  ​        没法演示，为什么？



附：课程讲解画图地址 ：http://huatu.xuanheng05.cn/lct/#R88b3e579acd7885cf5bb040e149112eb



- 内存一致性模型

​         因为处理器要遵守as-if-serial语义，处理器不会对存在数据依赖性的两个内存操作做重排序

  ![img](E:\mypro\imgbed\myimgs\jthread\1200440-20170811172858257-1790410735.png)

- 剖析unsafe类中的读，写屏障在hotspot中的源码

​    storeFence====storestore  在x86上是空操作。  第一是写操作，中间是storestore  ，第二个也是写操作

​    loadFence=====loadload    在x86上是空操作。第一是读操作，中间是loadload ，第二个也是读操作

​    loadstore=====？？？？numa架构下才会出现，smp架构下一般不会出现这个重排序

​    fullFence======storeload  在x86下唯一的一个屏障 ，第一个是写，中间是storeload  ，第二个是读屏障

​     lock addl

- 解析lock指令前缀

​     https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html

​     intel架构文档第3卷，8.1.4章节

![image-20210315172728935](E:\mypro\imgbed\myimgs\jthread\image-20210315172728935-1620560564728.png)



- volatile和lock指令前缀的关系

  - 一个案例：volatile底层究竟发生了什么？

    代码演示

    需要一个工具来查看底层汇编码：HSDIS 

    下载地址： http://lafo.ssw.uni-linz.ac.at/hsdis/att/

    放置路径：E:\dev_env\java-1.8.0-openjdk\jre\bin\server

    hotspot是用jit编译，jit分c1编译器和c2编译器

    c1编译器------client模式

    c2编译器------server模式

    加上虚拟机参数：

    -server
    -XX:+UnlockDiagnosticVMOptions
    -XX:+PrintAssembly
    -XX:-Inline

    **结论：字段上面如果加上volatile修饰，那么底层汇编码会加上lock前缀指令，从而起到全屏障的作用**

    

  - 再次解释storeload屏障为什么最重？？

    volatile----》底层全屏障（storeload）-----》lock指令-----》

    写读屏障为什么最重？？

    写指令    storeload屏障  读指令

    

    ![image-20210316102934150](E:\mypro\imgbed\myimgs\jthread\image-20210316102934150-1620560564729.png)

    

- 深度解析volatile的两大重要特性

  1、可见性

    每次读volatile变量的时候，读到的总是最新的值（即所有cpu，最后一次写入的操作），就是任何一个线程的最新写入。

  

  2、禁止指令重排

  维护happens-before：

  -对volatile变量的写入不能重排到写入之前的操作之前，从而保证别的线程能够看到最新的写入的值

  -对volatile变量的读操作，不能排到后续的操作之后。

  

  注意：禁止重排并不是禁止所有的重排，只有volatile写入不能向前，读取不能向后。除了这两种以外，其他重排可以可以允许的。

  volatile int b；

  volatile int a；

  a=1；

  b=1；

  

  列举：**normal store，normal load，volatile store，volatile load**

  重点关注后面这个指令，volatile store  

  normal store，**storestore** ，*volatile store*   

  volatile store ，**storestore**，*volatile store* 

  normal load，**loadstore**，*volatile store* ------------numa架构

  volatile load，**loadstore**，*volatile store*------------numa架构

  

  -----------------------------------------

  第一个指令是volatile写指令， 都可以重排

  volatile int a；

  int b；

  a=1；

  x=b；

  

  volatile store，normal load--------可以重排

  volatile store，normal store--------可以重排

  -------------------------------------

  volatile int a；

  volatile int b；

  int c；

  

  x=a；

  y=b；

  z=c；

  

  *volatile load*，**loadload**，volatile load

  *volatile load*，**loadload**，normal load

  *volatile load*，**loadstore**，normal store

  -----------------------------------------

  normal store，volatile load-------------可以重排

  normal load，volatile load-------------可以重排

  ------------

  最特别的一个，x86唯一的一个

  volatile store，**storeload** ，volatile load

  ----------------

  还有4个，都可以重排序

  nomal store，normal load

  nomal load，normal load

  nomal store，normal store

  nomal load，normal store

  

  一共16个组合，记忆规则：

  针对面试

  1）只要看第二个指令，是不是volatile写，如果是volatile写，那么一定需要加xxstore屏障，xx根据第一个指令来，如果是读（不管是normal读还是volatile读）那么xx=load，如果是写（不管是normal写还是volatile写），那么xx=store

  

  2）只要看第一个指令，是不是volatile读，如果是volatile读，那么一定需要加loadxx屏障，xx根据第二个指令来，如果是读（不管是normal读还是volatile读）那么xx=load，如果是写（不管是normal写还是volatile写），那么xx=store

  

  3）还有一个特殊的，volatile写后面接一个volatile读。这就是全屏障，一定会加storeload屏障

  

  4）其余都不需要屏障，可以重排序

  

- volatile运行时指令重排序

  美团案例

  

- volatile编译时重排序（java和c++中编译期指令重排序）

  - java中用jitwatch查看JIT编译期指令重排

  ​       需要工具：HSDIS ，JITWatch

  ​       jitwatch：可以同时对比java源码，java字节码，汇编码

  

  ​      1）jitwatch下载地址 https://github.com/AdoptOpenJDK/jitwatch

  

  ​      2）下载完解压之后，然后进入core文件夹，执行 mvn clean compile exec:java

  编译完之后 打开 launchUI.bat 文件

  

跑程序的时候需要加上虚拟机参数如下：

~~~shell
 -server
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintAssembly
-XX:-Inline
-XX:+LogCompilation
-XX:LogFile=jit.log
~~~



- c++中的编译期指令重排序演示

  g++ 编译的时候选择o2编译器之后，指令发生重排序

  ~~~c++
  int x,y,r;
  
  void method(){
     x=r;
    __asm__ volatile("" ::: "memory");//加入了内存屏障
     y=1;
  }
  ~~~

  

- 总结

  1）具体到x86里面，storeload屏障具体是怎么一回事？

  ​     x86里面没有invalidqueue

  2）解释之前的案例

  ​    

  3）volatile在中间件源码中的运用

  ​    tomcat

  4）早期的mesi协议锁总线，效率低下（强一致性）----》cpu工程师不能忍受，于是加入了storebuffer（好处：提高了访问效率，带来的缺陷是：1、可见性，2、容量问题）相对弱一点----》引入invalid queue(效率再次提升，带来的影响：一致性更加弱了)-----》请软件工程师自己解决（volatile）

  马姓：mesi（storebuffer，invalidqueue）

  


#### 2）synchronized机制

- 变量的安全性问题探讨

  - 多线程高并发，有共享变量

    1、有**共享变量**，有线程安全性问题（实例变量）

    2、**局部变量**没有线程安全性问题

  - synchronized的 可见性，有序性，原子性

​     

- synchronized字节码解析

  synchronized的底层究竟是什么？

  为什么会有多个monitorexit？



- 对象与锁

​       锁的本质就是一个对象，所有线程要争抢这个对象

​       问题：锁代码块和锁方法所对应的那把锁是同一个锁吗？？

​       证明了this 锁

​      证明class锁



- 锁重入机制

​     synchronized是可以重入，重入锁

   重入锁一定要注意，在锁嵌套的时候，所有嵌套的方法签名上一定要加上synchronized关键字

   在继承关系下锁重入机制是否可行？？

  答：可行



- synchronized修饰的方法如果发生异常锁怎么办？会释放吗？

  2个方法，都是有synchronized修饰的，其中一个发生异常了，jvm会在发生异常之后自动释放锁



- 方法同步 vs 代码块同步

  同步代码的优化方案



- 小总结

  是异步还是同步主要就是看synchronized是不是同一把锁。

  如果是同一把锁，那么就是同步执行

  如果不是同一把锁，那么就是异步执行

  

- this锁和class锁究竟在什么情况之下使用？

​      this锁实际上是当前方法所在的实例----》单例比较多

​      class锁实际上是当前方法所在的类----》new N个实例

  	总结：

​      	  1）如果当前class的实例是单例，那么就用this锁

​			2）如果当前class的实例不确定是否是单例，那么如果方法间需要同步，则只用class锁

​			前提：多线程执行多个方法，多个方法间需要同步

- 线程的死锁问题

  1）见视频讲解

  2）如何避免死锁？

  - 锁的获取保持有序
  - 设置超时（synchronized没有超时机制，我们自定义lock来设置超时）
  - 做死锁检测（数据结构）

  

- 锁对象的改变导致异步执行

  见视频讲解

  

- 锁优化-降低锁粒度

  concurrenthashmap，里面有一个叫做锁分段机制

  

- 逃逸分析及锁消除

  -server                             //jit的c2编译器

  -XX:+DoEscapeAnalysis  //逃逸分析

  -XX:+EliminateLocks       //锁消除

  -XX:+EliminateAllocations  //标量替换

  面试题：new创建的对象是否一定分配在堆上面？

  不一定，比如说栈上分配，tlab

  

  new point（int x，int y）；

  1）逃逸分析是什么？

  堆上面是有gc的，full gc

  定义在方法内部的局部变量，

  逃逸分析：实际上就是看局部变量有么有逃出方法之外，

  int method1（）{}

  int x；

  。。。。

  return x；

  }

  

  Point method2（）{}

   

    Point point=new Point（x，y）；

   system.out.print(point.x,point.y)

  ​    return point；

  }

  

  非栈上分配和栈上分配

  非栈上分配

  ```
  // 栈上分配---》锁会自动消除
  ```



- synchronized源码分析

  

  **附：课程讲解画图地址** ：http://huatu.xuanheng05.cn/lct/#R486b3cb25e10d85e15b67295b5a549f4

  

  jdk1.6之前就是用的monitor（操作系统底层的互斥锁）

  

  串行来访问共享资源

  本质：实际上同步互斥访问（多个线程来争取一个**对象**）

  new Object()---->对象最终是丢给jvm来管理----》jvm会在对象上加一些管理信息

  

  包装完之后：

  1）**对象头（重点）**

  2）实例数据

  3）填充数据

  

  32位

  ![image-20210416112302935](E:\mypro\imgbed\myimgs\jthread\image-20210416112302935.png)

  

  64位 Markword

  ![img](E:\mypro\imgbed\myimgs\jthread\640)

  

  

  - 步骤：先分析jvm源码，把流程整理出来，然后java层面写出来

  

  - 系统上下文切换：用户态和内核态直接的切换，是比较消耗资源的  

  

  - 先来实现重量级锁

  ​     

  ​       总结：一旦进入重量级锁，那么线程就会进入队列，一旦进入队列，说明线程已经挂起（进入内核态），        后续被唤醒，又需要从内核态进入用户态

  

  对象头（锁所在的对象头）---->MarkWord[ptrObjectMonitor]------>ObjectMonitor

  

  - 优化方案??

  ​       轻量级锁状态：如果竞争没这么激烈（cas能够拿到锁），是轻微竞争，那么此时就不需要将线程挂起。

  

  ​       LockRecord------》当前获取到锁的线程栈上面的锁记录

  ​      轻量级锁加锁过程：

  ​     1）首先，在线程栈中创建一个锁记录（LockRecord）

  ​     2）拷贝对象头的markword到当前栈帧中的锁记录中

  ​     3）cas尝试将markword中的指针指向当前线程栈中的锁记录，所标记也要改成00，表示此时已经是轻量级锁状态

  ​    4）如果更新失败，表示此时竞争激烈，1、需要进行锁膨胀操作 2、重入锁

  

  - 膨胀过程（轻量级锁膨胀为重量级锁）：

  ​      见图示讲解

  

  - 能不能再优化了？？

  ​       synchronized，都是只有1个线程在执行（在一定的时间之内没有线程竞争）

  

  - 为什么要有锁的撤销？？

  ​       偏向锁撤销

  ​       轻量级锁撤销

  

  ​      疑问：为什么撤销轻量级锁需要cas来撤销，而且还会撤销失败？

  ​     可能存在一种情况如下：

  ​       线程t1获取了轻量级锁，markword指向t1所在在栈帧，此时t2也来请求锁，此时拿不到锁，那么就会升  级膨胀为重量级锁，就把markword更新为ObjectMonitor指针。此时t1线程在退出的时候准备将markword还原，那么此时就会失败。t1只能膨胀为重量级锁退出

  

  - 讲了半天您还是没有实现锁啊？那就自定义实现一个真正能跑的锁。

     juc里面有一个aqs的东西（一个队列+state）

    countdownlatch，barrier

  

  - synchronized 偏向锁 轻量级锁 重量级锁证明

    

  - 案例入手开始分析偏向锁（批量重偏向、批量撤销）、轻量级锁、重量级锁及膨胀过程

    偏向锁：

  

  

  - 锁是否可以降级？

  

  

  - synchronized的非公平锁验证

  

