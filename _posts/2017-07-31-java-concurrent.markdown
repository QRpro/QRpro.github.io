---
layout:     post
title:      " Java 并发编程"
subtitle:   "关于并发编程你需要了解的知识..."
date:       2017-07-31 
header-img: "img/concurrent-programming-bg.jpg"
catalog:    true
author:     "Neil"
tags:
    - 并发编程
    - Java
---  


## 写在前面的话  
　　本文主要讲述了java 并发编程中的一些知识需要读者对java基础线程有一定的了解，其中本文主要包括Volatile关键字、Queue、JUC包、线程池等技术，算是对之前学习的巩固也是对新知识的总结。  

## 概念性的东西
　　**线程安全**：当多个线程访问某一个类（对象或者方法）时，这个类始终都能表现出正确的行为，那么这个类就是线程安全的。  
　　**Synchronized：**一个线程访问一个对象中的synchronized(Object)同步代码块时，其他试图访问该对象的线程将被阻塞，关键字synchronized取得的锁都是对象锁，而不是把一段代码（方法）当做锁，代码中哪个线程先执行synchronized关键字的方法，哪个线程就持有该方法所属对象的锁（Lock），但 在静态方法上加synchronized关键字，表示锁定.class类，类一级别的锁（独占.class类）。  
　　**wait/notify：** 线程的等待/唤醒机制、使用wait 和 notify必须配合synchronized关键字使用，且wait方法释放锁，notify方法不释放锁   
　　**Volatile：** 用来修饰被不同线程访问和修改的变量达到线程之间的通讯目的，jvm在运行时刻有一个内存区域是jvm虚拟机栈，每一个线程运行时都有一个线程栈，线程栈保存了线程运行时候变量值信息。当线程访问某一个对象时候值的时候，首先通过对象的引用找到对应在堆内存的变量的值，然后把堆内存变量的具体值加载到线程本地内存中，建立一个变量副本，之后线程就不再和对象在堆内存变量值有任何关系，而是直接修改副本变量的值， 在修改完之后，自动把线程变量副本的值回写到对象在堆中变量。在加载之后，如果主内存数据变量发生修改之后，线程工作内存中的值由于已经加载，不会产生对应的变化，所以会产生安全问题。为了防止同一变量被并发修改，保证每个线程读到的都是最新资源需要在线程间进行通信，最简单的方式就是修改时对执行代码块加synchronize 锁，方式二是通过volatile关键字来保证共享数据在线程见的通信。Volatile具有比synchronize更高的执行效率。当然Volatile是不具备原子性的，如果想保证数据原子性请使用atomic相关类(仅保证整体原子性)
![img](/img/blogarticles/concurrent/volatile.png)　 

## 同步类容器 
古老的`Vector`、`HashTable`。这些容器的同步功能由JDK的`Collections.synchroonized()`工厂方法去创建实现的。其底层的机制无非是用传统的synchronized关键则对每一个公用的方法都进行同步，是的每一次只要一个线程访问容器的状态。并不能满足高并发需求，再保证线程安全的同时，也必须要有足够好的性能
　　并发类容器：`jdk5.0`以后提供了多种并发类容器来替代同步类从而改善性能。同步类容器的状态都是串行化的。他们虽然实现了线程安全，但是严重降低了并发性，在多线程环境时，严重降低了应用的吞吐量。
并发类容器时专门针对并发设计的，使用`ConcurrentHashMap`来代替给予散列的传统的`HashTable`，而且在`ConcurrentHashMap`中，添加了一些常见复合操作的支持。以及使用了`CopyOnWriteArryList`代替`vector`，并发的`CopyonWriteArraySet`以及并发的`queue`，`ConcurrentLinkedQueue`和`LinkedBlockingQueue`，前者是高性能的队列，后者是以阻塞形式的队列，具体实现Queue还有很多，例如`ArrayBlockingQueue`、`PriorityBlockingQueue`、`SynchronousQueue`等。  

　　在具体介绍各个容器的特点时我们试着用前面讲到的wait/notify来简单的模拟一下Queue，BlockingQueue是一个消息队列，并且支持阻塞机制，阻塞的放入和得到的数据。这里简单模拟LinkedBlockingQueue下面的put和take方法。
- Put(Obect):把 Object加入到BlockingQueue中，如果BlockQueue没有空间，则调用此方法的线程被阻塞，直到BlockingQueue里面有空间再继续添加
- Take:取走BlockingQueue里面排在首位的对象，若BlockingQueue为空，阻断进入等待状态直到BlockingQueue中有新的数据被加入。

```java
public class MyQueue {
	//需要一个集合
	private final LinkedList<Object> list = new LinkedList<Object>();
	//需要一个计数器 
	private AtomicInteger count = new AtomicInteger(0);
	//指定容器上限/下限
	private final int minSize = 0;
	private final int maxSize;
	public MyQueue(int size){
		this.maxSize = size;
	}
	public void put(Object obj) throws Exception {
		synchronized (this) {
			if(count.get() == this.maxSize) {
				this.wait();
			}
			list.add(obj);	//加入元素
			count.incrementAndGet();	//计数器累加
			System.out.println("新加入的元素为："+obj);
			this.notify();	//唤醒
		}
	}
	public Object take() throws Exception {
		Object ret = null;
		synchronized (this) {
			while (count.get() == this.minSize) {
				this.wait();
			}
			ret = list.removeFirst();		//移除元素
			count.decrementAndGet();	//计数器--
			System.out.println("移除的元素为:" + ret);
			this.notify();		//唤醒
		}
		return ret;
	}
	
	public int getSize(){
		return this.count.get();
	}
}
```    

再来瞅瞅用来测试的主函数  

```java
public static void main(String[] args) throws Exception {
	final MyQueue mq = new MyQueue(2);
	mq.put("a");
	mq.put("b");
	System.out.println("当前容器的长度:"+mq.getSize());
	Thread t1 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				mq.put("c");
				mq.put("d");
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	},"t1");
	t1.start();
	Thread t2 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				mq.take();
				TimeUnit.SECONDS.sleep(1);
				mq.take();
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	},"t2");
	TimeUnit.SECONDS.sleep(2);
	t2.start();
}
```  

执行结果
![img](/img/blogarticles/concurrent/simulation-queue.png)  
设置最大容量为2，向队列中首先添加数据a,b 此时队列已满，线程`t1`向队列中添加数据进入阻塞状态，2S后`t2`线程开始执行从队列中取走a数据，此时队列有空余空间`t1`向队列插入c元素，此时队列满进入阻塞状态，1S后`t2`取走b，线程`t1`插入元素d。

现在对Queue和wait/notify有进一步了解了，那么下面就来瞅瞅JUC包下的并发集合。

#### ConcurrentMap    
  

> `ConcurrentHashMap` (可以理解为HashMap/HashTable)  
> `ConcurrentSkipListMap`(支持并发排序功能)(可以理解为TreeMap)   

`ConcurrentHashMap`和JDK6不同的是，JDK7中除了第一个`Segment`之外，剩余的`Segments`采用的是延迟初始化的机制。每次`put`之前都需要检查`key`对应的`Segment`是否为`null`，如果是则调用`ensureSegment()`以确保对应的Segment被创建。内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的`HashTable`,它们有自己的锁。只要多个修改操作发生在不同的段上，他们就可以并发进行。把一个整体分成了16个小段(Segment)。也就是最高支持16个线程的并发修改操作。这也是在多线程场景时减小锁的粒度从而降低锁竞争的一种方案。并且代码中大多共享变量使用volatile关键字声明，目的是第一时间获取修改的内容，
　　在JDK1.8中摒弃了Segment（锁段）的概念，而是启用了一种全新的方式实现,利用`CAS算法`。它沿用了与它同时期的HashMap版本的思想，底层依然由 `数组”+链表+红黑树`的方式思想。这篇文章讲的较为透彻推荐阅读！ [ConcurrentHashMap总结](http://www.importnew.com/22007.html)   

#### CopyOnWrite容器（COW）

> CopyOnWriteArrayList
> CopyOnWriteArraySet**

`CopyOnWrite`容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对`CopyOnWrite`容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以`CopyOnWrite`容器也是一种读写分离的思想，读和写不同的容器。

`CopyOnWriteArrayList`的实现原理
在使用CopyOnWriteArrayList之前，我们先阅读其源码了解下它是如何实现的。以下代码是向ArrayList里添加元素，可以发现在添加的时候是需要加锁的，否则多线程写的时候会Copy出N个副本出来
![img](/img/blogarticles/concurrent/cow-add.png)　 
读的时候不需要加锁，如果读的时候有多个线程正在向ArrayList添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的 
![img](/img/blogarticles/concurrent/cow-set.png)　  

#### 并发Queue  
在并发队列上JDK提供了两套实现，一个是以`ConcurrentLinkedQuee`为代表的高性能队列，一个是以`BlockingQueue`接口为代表的阻塞队列，无论哪种都继承自Queue
![img](/img/blogarticles/concurrent/queue-class.png)　 
`ConcurrentLinkedQuee`是一个适用于高并发场景下的队列，通过无锁方式，实现了高并发状态下的高性能，通常`ConcurrentLinkedQueue`性能好于`BlockingQueue`。它是一个基于链接节点的无界线程安全队列。该队列的元素遵循先进先出的原则。头是最先加入的，尾是最近加入的，不允许null元素。  

**重要方法：**
- add()/offer()都是取头元素节点(ConcurrentLinkedQueue中无任何区别)
- poll/peek区别在于前者会删除元素，后者不会

#### BlockingQueue接口 
- **ArrayBlockingQueue：**基于数组的阻塞队列实现，在`ArrayBlockingQueue`内部，维护了一个定长的数组，以便缓存队列中的数据对象，其内部没有实现读写分离，也就意味着生产和消费不能完全并行，长度是需要定义的，可以指定先进先出或者先进后出，也叫有界队列，在很多场合非常适合使用
- **LinkedBlockingQueue：**基于链表的阻塞队列，同`ArrayListBlockingQueue`类似，其内部也维持着一个数据缓冲队列（该队列由一个链表构成），当生产者往队列中放入一个数据时，队列会从生产者手中获取数据，并缓存在队列内部，而生产者立即返回；只有当队列缓冲区达到最大值缓存容量时（LinkedBlockingQueue可以通过构造函数指定该值），才会阻塞生产者队列，直到消费者从队列中消费掉一份数据，生产者线程会被唤醒，反之对于消费者这端的处理也基于同样的原理。而`LinkedBlockingQueue`之所以能够高效的处理并发数据，还因为其对于生产者端和消费者端分别采用了独立的锁来控制数据同步，以此来提高整个队列的并发性能。它是一个无界队列。
- **PriorityBlockingQueue：**基于优先级的阻塞队列（优先级的判断通过构造函数传入的Compator对象来决定），但需要注意的是`PriorityBlockingQueue`并不会阻塞数据生产者，而只会在没有可消费的数据时，阻塞数据的消费者。因此使用的时候要特别注意，生产者生产数据的速度绝对不能快于消费者消费数据的速度，否则时间一长，会最终耗尽所有的可用堆内存空间。在实现`PriorityBlockingQueue`时，内部控制线程同步的锁采用的是公平锁。它是一个无界的队列。注意：在take时候排序
- **DelayQueue：**带有延迟时间的Queue，其中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。`DelayQueue`中的元素必须实现Delayed接口，是一个没有大小限制的队列，应用场景有很多，比如对缓存超时的数据进行移除、任务超时处理、空闲连接的关闭等等。
- **SynchronousQueue：**一种没有缓冲的队列，生产者的数据直接会被消费者获取并消费

## JDK 多任务执行框架  
为了更好的控制多线程，JDK提供了一套线程框架Executor，帮助开发人员有效的进行线程控制。他们都在`ava.util.concurrent`包中，是JDK并发包的核心。其中有一个比较重要的类Executors，他扮演这线程工厂的角色，可以通过Executors创建特定功能的线程池。 

#### Executors创建线程池主要方法 
> `NewFixedThreadPool()`方法，该方法返回一个固定数量的线程池，该方法的线程数始终不变，当有一个任务提交时，若线程池中空闲，则立即执行，若没有，则会被暂缓在一个任务队列中等待有空闲的线程去执行。  

![img](/img/blogarticles/concurrent/fixed-threadpool.png)　 

> `NewSingleThreadExecutor()`方法，创建一个线程的线程池，若空闲则执行，若没有空闲线程则暂缓在任务队列中。   

![img](/img/blogarticles/concurrent/single-threadpool.png)　 

> `NewCachedThreadPool()`方法，返回一个可根据实际情况调整的线程个数的线程池，不限制最大的线程数量，若用空闲的线程则执行任务，若无任务则不创建线程。并且每一个空闲线程都会在60秒后自动回收。  

![img](/img/blogarticles/concurrent/cache-threadpool.png)　 

> `NewScheduledThreadPool()`方法，该方法返回一个SchededExecutorService对象，但该线程池可以指定线程的数量    

![img](/img/blogarticles/concurrent/scheduled1.png)
![img](/img/blogarticles/concurrent/scheduled2.png)
![img](/img/blogarticles/concurrent/scheduled3.png)

由上可知，线程池是通过`new ThreadPoolExecutor`的实例对象来创建， 观察Executors的构造方法
![img](/img/blogarticles/concurrent/thread-pool.png)　 

- `int corePoolSize：`当新任务在方法 execute(java.lang.Runnable) 中提交时，如果运行的线程少于 corePoolSize，则创建新线程来处理请求
- `int maximumPoolSize：`如果运行的线程多于 corePoolSize 而少于 maximumPoolSize，则仅当队列满时才创建新线程。如果设置的 corePoolSize 和 maximumPoolSize 相同，则创建了固定大小的线程池。如果将maximumPoolSize 设置为基本的无界值（如 Integer.MAX_VALUE），则允许池适应任意数量的并发任务。在大多数情况下，核心和最大池大小仅基于构造来设置，不过也可以使用 setCorePoolSize(int) 和 setMaximumPoolSize(int) 进行动态更改
- `long keepAliveTime：`如果池中当前有多于 corePoolSize 的线程，则这些多出的线程在空闲时间超过 keepAliveTime 时将会终止。
- `TimeUnit unit：`指定keepAliveTime时间单位
- `BlockingQueue<Runnable> workQueue：`所有 BlockingQueue 都可用于传输和保持提交的任务。如果运行的线程少于 corePoolSize，则 Executor 始终首选添加新的线程，而不进行排队。 如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程。 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。 
- `RejectedExecutionHandler handler：`拒绝策略当 Executor 已经关闭，并且 Executor 将有限边界用于最大线程和工作队列容量，且已经饱和时，在方法 execute(java.lang.Runnable) 中提交的新任务将被拒绝。在以上两种情况下，execute 方法都将调用其` RejectedExecutionHandler `的 `rejectedExecution`方法。  
下面提供了四种预定义的处理程序策略：   
	1. `AbortPolicy：`直接抛出异常组织，系统正常工作
	2.	`CallerRunsPolicy：`只要线程池未关闭，该策略直接在调用者线程中，执行当前被丢弃的任务
	3.	`DiscardOldestPolicy：`丢弃最老的一个请求，尝试再次提交任务。
	4.	`DiscardPolicy：`丢弃无法处理的任务，不给予任何处理。 
	了解完这些参数的作用后我们就可以自定义线程池了

#### 自定义线程池 

![img](/img/blogarticles/concurrent/my-threadpool1.png)　 
执行main方法  
![img](/img/blogarticles/concurrent/my-threadpool2.png)　 
自定义拒绝策略需要实现RejectedExecutionHandler接口 
![img](/img/blogarticles/concurrent/my-threadpool3.png)　  
MyTask实现Runnable接口 包含属性 ID 和 name  
![img](/img/blogarticles/concurrent/my-threadpool4.png)　 
执行结果  
![img](/img/blogarticles/concurrent/my-threadpool5.png)　 

当任务开始执行，初始化线程coreSize=1，故1号任务开始执行，有界队列ArrayBlockingQueue大小为3，任务2，3，4进入队列中，任务5由于队列大小以达上限，且MaxSize=2，所以系统分配一个线程直接执行，当任务6进入，由于已达最大线程数且队列已达上限故任务六被执行拒绝策略，首先1、5执行，执行完毕后2、3执行后4执行。

值得注意的是submit和execute的区别：
> submit可以传入实现Callable接口的实例对象  
> submit方法有返回值

## Concurrent.util工具类 

#### CyclicBarrier
CyclicBarrier（多个线程等待，最后一个线程调await方法时同时唤醒继续执行）：假设有只有的一个场景：每个线程代表一个跑步运动员，当运动员都准备好后，才一起出发，只要有一个人没有准备好，大家都等待  

```java
static class Runner implements Runnable {  
    private CyclicBarrier barrier;  
    private String name;  
    
    public Runner(CyclicBarrier barrier, String name) {  
        this.barrier = barrier;  
        this.name = name;  
    }  
    @Override  
    public void run() {  
        try {  
            Thread.sleep(1000 * (new Random()).nextInt(5));  
            System.out.println(name + " 准备OK.");  
            barrier.await();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        } catch (BrokenBarrierException e) {  
            e.printStackTrace();  
        }  
        System.out.println(name + "Go!!");  
    }  
}

```  

main 方法  

```java
public static void main(String[] args) throws IOException, InterruptedException {  
        CyclicBarrier barrier = new CyclicBarrier(3);  // 3 
        ExecutorService executor = Executors.newFixedThreadPool(3);  
        
        executor.submit(new Thread(new Runner(barrier, "1号运动员")));  
        executor.submit(new Thread(new Runner(barrier, "2号运动员")));  
        executor.submit(new Thread(new Runner(barrier, "3号运动员")));  
  
        executor.shutdown();  
    }  

```  

执行结果：
![img](/img/blogarticles/concurrent/run.png)　 

#### CountDownLacth  
CountDownLacth （一个线程等待，其他多个线程通知，主线程继续执行）：
经常用于监听某些初始化操作，等初始化执行完毕后，通知主线程继续工作  

```
CountDownLatch countDown = new CountDownLatch(2)
```  
在代码中使用
`countDown.await() `进行阻塞
当使用`countDown.countDown() `进行2次通知后阻塞方法继续执行

```java 
final CountDownLatch countDown = new CountDownLatch(2);
Thread t1 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				System.out.println("进入线程t1" + "等待其他线程处理完成...");
				countDown.await();
				System.out.println("t1线程继续执行...");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	},"t1");

Thread t2 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				System.out.println("t2线程进行初始化操作...");
				Thread.sleep(3000);
				System.out.println("t2线程初始化完毕，通知t1线程继续...");
				countDown.countDown();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	});

Thread t3 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				System.out.println("t3线程进行初始化操作...");
				Thread.sleep(4000);
				System.out.println("t3线程初始化完毕，通知t1线程继续...");
				countDown.countDown();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	});  
t1.start();
t2.start();
t3.start();  
```  

![img](/img/blogarticles/concurrent/countdown.png)　 
执行线程`t1、t2、t3`，`t1`输出文字后阻塞，`t2`输入文字并休眠3S后调用`countDown()`方法通知线程由于`new CountDownLatch(2)`的参数设置为2故需要两次countDown的调用才能唤醒线程`t1`，当`t3`执行完毕后再次调用该方法，线程`t1`被唤醒继续执行。

#### Callable 和 Future 
Future模式非常适合在处理耗时很长的业务逻辑时使用，可以有效的减小系统的响应时间，提高系统的吞吐量。  

```java
public class UseFuture implements Callable<String>{
	private String para;
	
	public UseFuture(String para){
		this.para = para;
	}
	
	/**
	 * 这里是真实的业务逻辑，其执行可能很慢
	 */
	@Override
	public String call() throws Exception {
		//模拟执行耗时
		Thread.sleep(5000);
		String result = this.para + "处理完成";
		return result;
	}
	
	//主控制函数
	public static void main(String[] args) throws Exception {
		String queryStr = "query";
		//构造FutureTask，并且传入需要真正进行业务逻辑处理的类,该类一定是实现了Callable接口的类
		FutureTask<String> future = new FutureTask<String>(new UseFuture(queryStr));
		//创建一个固定线程的线程池且线程数为1,
		ExecutorService executor = Executors.newFixedThreadPool(1);
		//这里提交任务future,则开启线程执行RealData的call()方法执行
		//submit和execute的区别： 第一点是submit可以传入实现Callable接口的实例对象， 第二点是submit方法有返回值
		// f.get()当执行完毕后返回null , future.get()返回执行结果
		Future f = executor.submit(future);		//单独启动一个线程去执行的
		
		System.out.println("请求完毕");
		
		try {
			//这里可以做额外的数据操作，也就是主程序执行其他业务逻辑
			System.out.println("处理实际的业务逻辑...");
			Thread.sleep(1000);
		} catch (Exception e) {
			e.printStackTrace();
		}
		//调用获取数据方法,如果call()方法没有执行完成,则依然会进行等待
		System.out.println("数据：" + future.get());
		executor.shutdown();
	}
}

```  
当调用submit方法时开启线程执行FutureTask对象中实现了Callable接口的call方法，当调用get方法时线程阻塞等待返回数据的数据。

#### 锁的高级深化  
在java中，可以使用sycnchronized关键字来实现线程间的同步互斥工作，那么其实还有一种更优秀的机制去完成这个工作，Lock对象，这里主要阐述两种所，重入锁和读写锁。
> `ReentrantLock`（重入锁） 

jdk的concurrent包提供的一种独占锁的实现。它继承自Dong Lea的 `AbstractQueuedSynchronizer`（同步器），确切的说是ReentrantLock的一个内部类继承了AbstractQueuedSynchronizer，ReentrantLock只不过是代理了该类的一些方法  

```java
Lock lock = new ReentrantLock(boolean isFair); 
lock.lock(); 
lock.unlock();（finally代码块）
```  

使用sycnchronized时，需要多线程协同工作则需要wait()/ notify() / notifyAll() 方法进行配合工作，同样，在使用Lock的时候，可以使用一个新的等待/通知类`Condition`，Condition一定是针对具体某一把锁，也就是只有锁的基础上才会产生Condition。 

```java
Condition c = lock.newCondition()
c.await();
c.signal(); 
```

> `ReentrantReadWriteLock`（读写锁）  

其核心就是实现读写分离的锁，在高并发访问下，尤其是读多写少的情况下，性能要远高于重入锁，在了解synchronized、ReentrantLock时在同一时间，只能有一个线程进行访问被锁定的代码，那么读写锁本质是分为两个所，读锁和写锁。在读锁下，多个线程可以并发的访问，但是在写锁时候只能顺序访问，读读共享，写写互斥，读写互斥。  

```java
private ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
private ReadLock readLock = rwLock.readLock();
private WriteLock writeLock = rwLock.writeLock();
readLock.lock();
writeLock.lock();
```