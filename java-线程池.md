# java ThreadPool

突然发现好久没有写文章, 罪过罪过. 赶紧上来补作业.
为何突然写这边文章, 原因是今天装了idea的开发插件[阿里巴巴开发手册 idea plugin](https://github.com/alibaba/p3c), 良心推荐哈. 装完以后打开代码, 提示了一个错误. 我原来代码中是这么写的

```
private final static ExecutorService excutorService = Executors.newFixedThreadPool(3);
```
提示推荐使用ThreadPool的写法, 原因是使用newFixedThreadPool可能造成OOM的问题.那好, 我们来复习一下JAVA的线程池的概念和正确的写法姿势.

### 基础概念
ThreadPool顾名思义是一个线程池的管理机制, Executor可以自由的重用线程池中的空闲线程,线程池会回收空闲的线程或是在条件允许的情况下创建worker去执行task.

#### task和执行策略之间的耦合关系

##### 线程饥饿死锁
在线程池中, 如果任务依赖于其他任务, 将可能产生死锁.举个例子, 如果一个任务将另一个任务提交到同一个Executor中, 并等待这个被提交的任务的结果, 那么通常会引发死锁, 第二个任务停留在工作队列中,并等待第一个任务完成, 而第一个任务有无法完成, 因为她在等待第二个任务的结果.在更大的线程池中, 如果所有正在执行的任务的线程都有由于等待其他处于工作队列的任务而阻塞, 那么会发生同样的问题, 这种现象就被成为线程饥饿死锁.举个现实中的例子, 比如我们定义了一个任务, 在这个任务中我们定义了一个Future<T> futureTask, 并执行futureTask.get(). 这里就存在任务在等待子任务的结果的情况, 将发生死锁.所以每当提交一个有依赖性的Executor任务, 要清楚的知道可能会出现线程饥饿死锁, 因此需要在代码或是配置Executor的配置文件中记录线程池的大小或是配置限制.

##### 运行时间较长的任务

如果运行一个任务时间较长, 即便不出现死锁的情况, 也会是整个线程池的响应变得很糟糕, 所以我们需要限定等待资源的时间, 而不能无限制的等待下去. 大多数的可阻塞方法中,都有定义限时版本和不限时版本, 比如: Thread.join, BlockingQueue.put, CountDownLatch.await, Selector.select等, 如果等待超时, 我们可以将其标记为超时失败, 然后终止或是放到队列中以便重试执行.这样, 我们就可以将线程腾出来执行一些耗时更少的任务, 使得整个线程池保持在正常工作的状态下, 如果一个线程池老是充满了阻塞的任务,那么你可以考虑是不是线程池规模太小了?

#### 线程池的大小设置

线程池的大小设置是一个很讲究的事情, 主要取决于被提交的任务类型(CPU密集, IO密集)和系统的硬件设备. 在代码中通常不会固定线程池的大小,而是通过某种配置机制来提供或是根据Runtime.availableProcessors来动态的计算.

在设置线程池大小的时候, 不能过大也不能过小, 过大会导致资源(CPU, 内存)被耗尽, 太小又会导致无法合理的使用CPU,从而吞吐率降低.

我们可以从几个角度去考虑如何设置线程池的大小: 

1. 系统有几个CPU
2. 内存多大
3. 任务是CPU密集型还是IO密集, 还是两者都行
4. 是否是稀缺资源, 如JDBC链接这种?
5. 是不是不同类型的任务, 如果是不同类型的任务, 需要考虑多个线程池分开处理.

一个CPU密集的任务建议是使用N+1的形式, N代表CPU的核数(当线程出现由于页缺失故障或是其他原因导致暂停时, 这个1也能确保CPU的时钟周期不会被浪费). 当如果是IO密集的操作, 由于线程并不会一直执行, 所以线程池的规模需要更大, 我们可以通过某种方法来调节线程池的大小, 在某个基准负载下, 分别设置不同大小的线程池来运行程序, 并观察CPU利用率的水平.有一个大致的公式是:
```
N(线程数) = N(CPU数)*N(CPU)*(1+等待计算时间的系数)
```

通过这个方式动态获取CPU数目:`intN_CPUS=Runtime.getRuntime().availableProcessors();`

### ThreadPoolExecutor实践

ThreadPoolExecutor为Executor提供了基本的实现, 这些Executor是由Executors中的newCachedThreadPool, newFixedThreadPool,newSingleThreadExecutor, newScheduledThreadExecutor等工厂方法返回的.现在我们来看看ThreadPoolExecutor的实现, 首先看看构造方法.

```
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
从这里, 我们可以看到几个参数:

1. 核心线程池大小
2. 最大线程池大小
3. 线程空闲存活时间
4. 任务队列
5. 线程工厂类
6. 拒绝策略实现

在调用ThreadPoolExecutor的execute方法时, 先判定当前线程池中的thread是否小于corePoolSize大小, 如果真, 则创建一个worker去执行task, 如果假, 则将任务加入工作队列,如果工作队列也满了, 则查看当前线程数是否小于最大线程数, 如果真, 则创建worker去执行任务, 如果假, 则执行阻塞规则.所以, 我们会发现, 只有当线程池数量大于等于核心线程池大小且工作队列也满的情况下才会创建超出核心线程池数量的线程.

从上面的流程我们发现线程池的坑位寸土寸金啊,线程如果执行完了task, 他改何去何从呢? 针对空闲线程的回收策略是这样的,当一个线程的空闲时间超过了设置的存活时间, 那么他将被标记为可回收(注意哦, 这里是标记为可回收, 也就是说回收并不是立马执行的, 会产生额外的延迟), 如果线程池当前的线程数量超过了基本大小的时候, 这个空闲线程会被终止掉.通过调节线程池的大小以及存活时间, 可以帮助线程池回收空闲线程占有的资源, 从而使得这些资源可以用于执行其他工作.

#### 任务队列
在上面我们提到了任务队列, 任务队列某种程度上来说是一种缓冲的策略, 当请求大量进来的时候, 如果请求的速率大于服务器处理的速率, 为了避免无限制的创建worker, 将task先放到任务队列中, 以此来缓解任务突增的情况.ThreadPoolExecutor提供了BlockingQueue来保存等待执行的任务, 基本的队列方法有: 无界队列, 有界队列和Synchronous Handoff.

上面我们提到的newFixedThreadPool和newSingleThreadExecutor在默认情况下采用的就是无界队列LinkedBlockingQueue.因为采用无界队列, 这也是为什么文章开头说了使用newFixedThreadPool会导致OOM了, 可想而知, 当任务新增速率大于处理速率的时候, 这个无界队列可能导致超内存了, 因为这个无界队列可以无限制的新增任务.

所以, 更稳妥的方式是采用有界队列这样可以避免资源耗尽的情况.但是又会带来一个新的问题, 如果队列满了怎么办呢? 客官别着急, 我们一步步解释.在使用有界队列的时候, 队列的大小与线程池的大小需要一起进行调节, 线程池较小, 队列大这看起来是一个不错的选择, 因为这样可以减少内存使用量,和CUP进行上下文切换, 但是吞吐率会有些牺牲.

刚刚我们还讲到了newCachedThreadPool. 这个笔者以前有一个误区, 误以为excutor执行一个task的时候newCachedThreadPool就会创建一个worker去处理task,导致OOM.现象是这样的,但是实际上newCachedThreadPool底层是基于Synchronous Handoff的机制, 与无界队列和有界队列不同, SynchronousQueue并不是实际上的队列, 而是线程直接的一个移交机制,将一个元素放入到SynchronousQueue中, 必须有另一个线程接受这个元素, 如果小于最大的线程数,ThreadPoolExecutor就会创建worker去处理task, 如果达到最大线程数了, 那么就会被拒绝掉, 所以任务如果可以被拒绝的话, 才可以考虑使用SynchronousQueue.
另外, 只有当任务互相独立的时候为线程池或队列设置容量才合理, 如果任务之间存在依赖,那可能会导致线程饥饿死锁的问题, 那就要考虑无界限的线程池了newCachedThreadPool.

#### 拒绝策略
前面我们卖了个关子, 讲到队列如果满的情况如何处理, 现在我们就来谈谈拒绝策略.JDK提供了几种不同的RejectedExecutionHandler实现, 每种实现都包含不同的拒绝策略: AbortPolicy, CallerRunsPolicy, DiscardPolicy和DiscardOldestPolicy.下面我们一个个来讲.

Abort: 这是一个默认的拒绝策略, 这个策略会抛出RejectedExecutionExceptioni, 开发者可以捕获这个异常, 然后编写自己的处理代码.
Discard: 顾名思义就是抛弃了
DiscardOldest: 抛弃下一个将被执行的task, 然后再尝试提交到队列中
CallerRuns: 这个机制很有意思, 既不会抛弃, 也不会抛出异常, 而是会回退到调用者, 从而降低新任务的流量, 她不会在线程池的某个线程中执行新的task, 而是在一个调用了execute的线程中执行这个任务.也就是说, 会在调用execute的主线程中去执行.

写了那么多, 我们来举个例子, 定义一个有界的线程池:

```
private final static ThreadPoolExecutor excutorService = new ThreadPoolExecutor(3, 10, 0L,
            TimeUnit.MICROSECONDS, new LinkedBlockingDeque<>(100),
            new ThreadFactoryBuilder().setNameFormat("order-pool-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy());
```
哦, 对了. 刚刚说了AbortPolicy可以抛出异常去处理逻辑, 我们来写个例子:

```
public class TestExecutor{
	private final Executor exec;
	private final Semaphore semaphore;
	
	public TestExecutor( Executor exec, Semaphore semaphore) {
		this.exec = exec;
		this.semaphore =semaphore;
	}
	
	public void submitTask(final Runnable cmd) {
		semaphore.acquire();
		try{
			exec.execute(new Runnable(){
				public void run() {
					try{
						cmd.run();
					} finally {
						semaphore.release();
					}
				}
			});
		} catch(RejectedExecutionException e) {
			semaphore.release();
		}
	}
}
```

#### 线程工厂
我们看构造方法里面还有一个线程工厂, 所谓线程工厂就是线程池每次创建一个线程都是通过线程工厂方法来完成的, 在默认的情况下,线程工厂方法将创建一个新的,非守护线程. 我们可以重新定义线程工厂方法来实现一些定制需求, 如给线程指定一个UncaughtExecptionHandler, 或是实例化一个定制的Thread类用来执行调试信息的记录,简单到给线程取一个名字等等.

### 扩展ThreadPoolExecutor
很多时候我们希望可以监控到我们的线程池的状态, 统计线程池的各种信息, 尝尝用来调优或是做一些报警的工作.

我们可以自定一个MonitorThreadPoolExecutor继承自ThreadPoolExecutor类, ThreadPoolExecutor提供了几个在子类中改写的方法: beforeExecute, afterExecute, terminated. 下面我们一个个解释.

* beforeExecute: 顾名思义, 在执行一个task之前调用. 如果beforeExecute抛出了RuntimeException, 那么任务就不会执行
* afterExecute: afterExecute会在task完成以后(无论成功失败)被调用.配合beforeExecute我们可以统计一个task所花费的时间
* terminated: 在所有的任务都已经完成, 所有的工作线程都关闭的情况下会被调用, terminated用来释放各种资源, 我们可以用他来统计一些信息



### 思考
* 关于线程池大小的设定, 如何调优
* 核心线程池数量和最大线程池数量
* 扩展ThreadPoolExecutor, 监控,调优线程池