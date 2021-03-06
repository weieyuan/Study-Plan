#### Java的多线程
___
* 无状态对象：既不包含任何域，也不包含任何对其他类中域的引用。
* 同步代码代码块包含两个部分：锁对象的引用，由锁所保护的代码块。
* volatile变量，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器中，因此在读取volatile变量时总会返回最新写入的值。

#### 进程和线程
* 进程
	* 程序的执行过程
	* 持有资源和线程
* 线程
	* 执行任务的最小单元
	* 一个进程持有多个线程
	* 线程共享进程的资源

#### 并发容器
* CopyOnWriteArrayList  
> 写入时复制(Copy-on-Write)
> 对修改操作加锁，并且会复制一份底层的数组，在该数组上进行操作，最后将该数组赋值给底层数组，这样保证不会影响读的操作。  
> 该容器适用于大量读操作和少量写操作的场景

* Collections.unmodifiableMap(Map map)  
> 代理map对象，但是不支持任何修改操作
> 但是可以直接修改map对象

* ConcurrentHashMap  
> 采用分段锁，可以保证任意数量的线程并发地读取该容器；读线程和写线程可以并发访问；一定数量的线程可以并发地修改该容器。    
> 并没有实现对Map加锁以提供独占访问，因此在多线程的环境中size和isEmpty的方法返回的并不是准确的值，只是一个估算的值  
> 在需要加锁Map进行独占访问时，这个容器就不适用了，可以采用Hashtable和synchronizedMap

#### 同步工具类
* 闭锁
例如CountDownLatch,FutureTask  
用来确保某些活动直到其他活动完成之后才可以执行，适用的场景例如必须把饭端到桌上，把菜端到桌上，我才能开始吃饭  

```
//关键api
countDown(): 减1
await(): 等待
//示例
public class Test {
	
	private static class Child implements Runnable {
		
		private CountDownLatch countDownLatch;

		public Child(CountDownLatch countDownLatch){
			this.countDownLatch = countDownLatch;
		}
		
		
		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + ": child run begin");
			this.countDownLatch.countDown();
			System.out.println(Thread.currentThread().getName() + ": child run end");
		}
		
	}
	
	private static class Parent implements Runnable {
		
		private CountDownLatch countDownLatch;
		
		public Parent(CountDownLatch countDownLatch){
			this.countDownLatch = countDownLatch;
		}

		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + ": parent run begin");
			try {
                //等待countDownLatch为零
				this.countDownLatch.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + ": parent run end");
		}
		
	}

	public static void main(String[] args){
		int num = 5;
		CountDownLatch countDownLatch = new CountDownLatch(num);
		ExecutorService service = Executors.newFixedThreadPool(num + 1);
		
		service.submit(new Parent(countDownLatch));
		
		for(int i = 0; i < num; i++){
			service.submit(new Child(countDownLatch));
		}
		
		service.shutdown();
	}	
}

```

* 信号量
例如Semaphore，用来控制同时访问某个资源的操作数量。  

```
//关键api:
acquire(): 获取信号量
release(): 释放信号量
//示例
public class Test {
	
	private static class Tourist implements Runnable{

		private Semaphore semphore;
		
		public Tourist(Semaphore semphore){
			this.semphore = semphore;
		}
		
		@Override
		public void run() {
			try {
				System.out.println(Thread.currentThread().getName() + ": 排队进入景区。");
				this.semphore.acquire();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + ": 可以进景区参观了。");
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			this.semphore.release();
		}
		
	}

	public static void main(String[] args) {

		Semaphore semphore = new Semaphore(5);
		ExecutorService service = Executors.newCachedThreadPool();
		for(int i = 0; i < 10; i++){
			service.submit(new Tourist(semphore));
		}
		
		service.shutdown();
		
	}

}
```

* 栅栏
例如CyclicBarrier  
与闭锁的区别是，所有的线程必须都到达栅栏的位置才可以向下执行。例如跑步的时候，必须所有人都在起跑线上排好队才可以开始跑步  

```
//关键api
await(): 等待
//示例
public class Test {

	private static class Child implements Runnable{
		
		private CyclicBarrier cyclicBarrier;
		
		public Child(CyclicBarrier cyclicBarrier){
			this.cyclicBarrier = cyclicBarrier;
		}

		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + ": child run begin");
			try {
                //会在cyclicBarrier上等待，在cyclicBarrier上调用指定次数的await后，后面代码会接着执行
				this.cyclicBarrier.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			} catch (BrokenBarrierException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + ": child run end");
		}
	}
	
	private static class Parent implements Runnable{

		@Override
		public void run() {
            //当在cyclicBarrier上调用指定次数的await后，代码会被执行
			System.out.println(Thread.currentThread().getName() + ": parent run start");
			System.out.println(Thread.currentThread().getName() + ": parent run end");
		}
		
	}
	
	public static void main(String[] args) {
		int num = 5;
		CyclicBarrier cyclicBarrier = new CyclicBarrier(num, new Parent());
		ExecutorService service = Executors.newFixedThreadPool(num + 1);
		for(int i = 0; i < num; i++){
			service.submit(new Child(cyclicBarrier));
		}
		
	}

}
```

#### 定时任务
* Timer  
> TimerTask抛出未受检的异常时将终止定时线程，这种情况下Timer也不会恢复线程的执行，而是错误地认为整个Timer都被取消了。
> 建议使用ScheduledExecutorService来执行定时任务

#### 线程池
* 概念
	* 线程池的基本大小(Core Pool Size):没有线程执行时线程池的大小,只有在工作队列满的情况下才会创建超过这个数量的线程
	* 最大大小(Maximum Pool Size):可同时活动的线程数量的上限
	* 存活时间：某个线程的空闲时间大于存活时间时，那么将会标记为可回收，并且当线程池的当前大小超过了基本大小时，这个线程将会被终止
	* 线程池是通过ThreadFactory来创建线程的，可以指定自定义的ThreadFactory来自定义线程的创建
* newFixedThreadPool
> 提交一个任务就会创建一个线程，直到达到线程池的最大限制，这是线程池的规模不会发生变化，当某个线程down掉后，会补充一个新的线程
* newCachedThreadPool
> 可缓存的线程池，当缓存的线程数超过任务数时，会回收空闲的线程；当任务数增加时，则可以添加新的线程；线程池的规模不存在限制
* newSingleThreadExecutor
> 单线程，当线程down掉时，会创建一个新的线程来代替
* newScheduledThreadPool
> 创建固定长度的线程池，可以延时或者定时执行任务。

线程池的创建：  
```
//Executors
ExecutorService newFixedThreadPool(int nThreads);
ExecutorService newSingleThreadPool()
ExecutorService newCachedThreadPool()

ScheduledExecutorService newScheduledThreadPool(int corePoolsize)
```

**#线程池的饱和策略**  
* 概念
> 当有界队列被填满或者向已经关闭的Executor提交任务都会触发饱和策略。可以通过ThreadPoolExecutor.setRejectedExecutionHandler()来修改饱和策略
* AbortPolicy，这是线程池的默认策略。抛出未检查的RejectedExecutionException
* CallerRunsPolicy,调用者运行策略，会将调用者提交的任务返还给调用者，在调用者线程中运行该任务
* DiscardPolicy，抛弃
* DiscardOldestPolicy，抛弃最旧的，也就是执行优先级最高的

#### 工作队列
* LinkedBlockingQueue
* ArrayBlockingQueue
* PriorityBlockingQueue
* SynchronousQueue
> 常用的场景是生产线程放置数据时，需要等待消费者线程来获取数据；反之亦然。

#### 线程的中断
* interrupt方法能够中断线程
* isInterrupted方法能够返回线程的中断状态
* interrupted方法将清除当前线程的中断状态，并返回它之前的值
* 一般阻塞的方法都能抛出中断异常

#### Fork/Join
* Fork/Join框架涉及到的类：  
ForkJoinTask: 任务描述类，有两个抽象子类，RecursiveTask用于描述有结果返回的任务，RecursiveAction用于描述无结果返回的任务，在使用时，根据需要继承这两个类，需要实现compute方法，在compute方法中需要对任务进行判断，如果任务比较大需要将任务分割为小任务，否则直接执行任务。可以通过invoke/invokeAll将拆分的子任务提交给线程池，或者调用fork、join方法。   
ForkJoinPool：任务执行的线程池，继承于AbstractExecutorService类。建议使用ForkJoinPool.commonPool()方法来获取ForkJoinPool对象，因为这个线程池是预定义好的，可以达到公用的目的。execute方法用于提交没有返回值的任务，submit用于提交有返回值的任务。  

```
///compute方法实现
protected Integer compute()
    {
        int sum = 0;
        if (end - start + 1 <= threshold)
        {
            System.out.println(Thread.currentThread() + ",start: "
                + start + ",end: " + end);
            for (int i = start; i <= end; i++)
            {
                sum += i;
            }
        }
        else
        {
            System.out.println("begin to fork task");
            int middle = (end + start) / 2;
            CalcNumForkJoinTask leftTask = new CalcNumForkJoinTask(start,
                middle, threshold);
            CalcNumForkJoinTask rightTask = new CalcNumForkJoinTask(middle + 1,
                end, threshold);
            leftTask.fork();
            rightTask.fork();
            int leftRes = leftTask.join();
            int rightRes = rightTask.join();
            sum = leftRes + rightRes;
        }
        return sum;
    }
///提交主任务给线程池
 CalcNumForkJoinTask oCalcNumForkJoinTask = new CalcNumForkJoinTask(1,
            100, 20);
 ForkJoinPool pool = new ForkJoinPool();
 Future<Integer> res = pool.submit(oCalcNumForkJoinTask);
```

#### ThreadLocal
ThreadLocal类似于一个管理类，其中最重要的两个api是：
T get()//设置变量
void set(T)//获取变量
在get/set方法中通过Thread.currentThread()取得当前线程对象，线程对象中有ThreadLocalMap这个属性，T对象就是保存在ThreadLocalMap对象中的。

```
//set
 public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

//get
 public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

#### Java内存模型与线程
cpu与内存之间使用高速缓存，但是同时带来了缓存一致性的问题，每个计算单元都有自己的缓存，但是又共享一块内存，这样存在缓存和内存数据一致性的问题。

java的内存模型规定所有的变量都存储在主内存中，每个线程都有自己的工作内存，工作内存中保存了被该线程使用到的变量的主内存副本拷贝，线程对变量的操作都必须在工作内存中进行，不能直接操作主内存中的变量，不同的线程之间也无法直接访问对方工作内存中的变量。

java内存模型中定义了8种操作，虚拟机确保了每一种操作都是原子操作：

* lock
* unlock
* read 把变量的值从主内存传输到工作内存
* load 把从read操作中取得的变量值放入到工作内存的副本中
* use
* assign
* store
* write

volatile变量确保了对所有线程的可见性，即当一个线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。可以简单理解为线程在使用volatile变量时都需要从主内存中同步该变量值。

*原子性、可见性、有序性：*

* 原子性：由java内存模型直接保证的原子性变量操作包括read、load、assign、use、store、write。我们大致可以认为基本数据类型的访问读写是具备原子性的。

* 可见性：一个线程修改了共享变量的值，其他线程能够立即得知这个修改。

* 有序性：如果在本线程内观察，所有的操作都是有序的，如果在一个线程中观察另一个线程，所有的操作都是无序的。


*先行发生原则：*
这个原则非常重要，他是判断数据是否存在竞争、线程是否安全的主要依据。先行发生是指两项操作之间的偏序关系，如果操作A先行发生于操作B，其实就是说在发生操作B之前，操作A产生的影响能被操作B观察到。

java的内存模型中“天然具有”的先行发生关系的操作有：

* 程序次序发生：在一个线程内，按照程序代码顺序执行
* 管程锁定规则：一个unlock操作先行发生于后面对同一个锁的lock操作
* volatile变量规则：对volatile变量的写操作先行发生于后面对这个变量的读操作，这里的后面是指时间上的先后顺序
* 线程启动规则
* 线程终止规则
* 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
* 对象终结规则：一个对象的初始化完成先行发生于它的finalize()方法的开始
* 传递性

* 锁的缺陷，操作系统在挂起和恢复线程的过程中存在很大的开销
* 锁的优化
	* 自旋锁和自适应自旋锁，请求锁的线程认为很快就可以获得锁，因此没有必要将线程挂起，只需要让该线程执行一个忙循环(自旋)，这项技术就是自旋锁。
	* 锁消除，虚拟机及时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。
	* 锁的粗化，大部分情况下我们总是将同步块的作用范围限制的尽量小。但是如果虚拟机探测到有这样一串零碎的操作都是对同一个对象加锁，将会把加锁同步的范围扩展到整个操作序列的外部。
	* 轻量锁
	* 偏向锁

#### 其他
* 无状态对象：既不包含任何域，也不包含任何对其他类中域的引用。
* 同步代码代码块包含两个部分：锁对象的引用，由锁所保护的代码块。
* volatile变量，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器中，因此在读取volatile变量时总会返回最新写入的值。
* Future、Callable、FutureTask
> Future描述一个任务的生命周期，方法有cancle/isCancled/isDone/get
> Callable描述一个任务
> FutureTask实现了Future接口和Callable接口，描述一个计算任务
* CompletionService
> 实现类ExcutorCompletionService，这个实现类能够执行任务，并且会把每个任务的结果放置到BlockingQueue中。

**#CAS**  
CAS(Compare and Swap)比较并交换，当内存值V和预期值A相等时，将内存值A更新为B，否则什么也不干。CAS可以保证一次读-改-写是原子操作的。

缺点：  
1. 循环时间长开销大
2. ABA问题
3. 只能保证一个共享变量的原子操作

**#AQS**  
(AbstractQueuedSynchronizer)队列同步器维护一个volatile int state(代表共享资源)和一个FIFO线程等待队列(多线程争用资源被阻塞时会进入此队列)，state访问方式有三种：  
getState()  
setState()  
compareAndSetState()  

AQS定义了两种资源共享方式，独占式，例如ReentrantLock；共享式(多个线程可同时执行)，例如Semaphone和CountDownLatch

#### 锁的分类  
* 公平锁/非公平锁  

* 独享锁/共享锁
synchronized是独享锁。  
ReentrantLock是独享锁，ReentrantReadWriteLock中的读锁是共享的，写锁是独享的

* 乐观锁/悲观锁  
悲观锁认为对同一个数据的并发访问，一定会发生修改，那么不加锁的并发操作一定会出问题。  
乐观锁认为对同一个数据的并发操作，是不会发生修改，那么不加锁的并发操作是不会发生问题的。

* 分段锁
例如ConcurrentHashMap

* 偏向锁
指一段同步代码一直被一个线程所访问，那么该线程会自动获取该锁，降低获取锁的代价。

* 自旋锁
尝试获取锁的线程不会阻塞，而是采用循环的方式去获取锁，好处是减少线程的上下文切换的消耗，缺点是循环会消耗CPU

