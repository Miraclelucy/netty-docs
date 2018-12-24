
# 线程池是什么

线程大家都清楚是什么？那么线程池是什么？在使用线程的时候有没有考虑过，在平时使用的时候系统中只要你想随处都可以创建线程并且很难管控，基本上一个线程使用完就销毁掉了，要在使用便新建一个线程

> 直接使用线程的缺陷（针对线程池）

- 线程数量无法限制，想创建多少个就多少个
- 线程无法复用，创建启动和销毁线程是会带来一定的开销

线程池的出现主要就是解决2个问题，一个是限制线程的数量和线程复用，在这个扩展上面可以再自行扩展出监控等。

# 线程池的使用
Java的Executors工具类就提供了几种现成的创建线程池实例的方法：
- newFixedThreadPool
- newSingleThreadPool
- newCachedThreadPool
- newScheduledThreadPool
    
他们最终的放回值都是返回一个ExecutorService,且内部实际都是调用了ThreadPoolExecutor不同的构造方法。

# newFixedThreadPool
> 该方法会创建一个固定线程数量的线程池，其核心线程数和最大线程数都是设的值。
```java
class PrintTask implements Runnable{

    private int sequence;
    private long sleepMillis;

    public PrintTask(int sequence, long sleepMillis) {
        this.sequence = sequence;
        this.sleepMillis = sleepMillis;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(sleepMillis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " print: " + sequence);
    }
}

public static void main(String[] args) {
    ExecutorService executor = Executors.newFixedThreadPool(10);
    for (int i = 0; i < 12; i++) {
        executor.execute(new PrintTask(i+1, 3000L));
    }
    executor.shutdown();
    System.out.println("Is shutdown : " + executor.isShutdown());
    System.out.println("Is terminated : " + executor.isTerminated());
    try {
        Thread.sleep(5000L);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println(executor.isTerminated());
}
```
从示例代码中可以看出，首先我创建了一个固定大小的线程池，其固定数量为10，然后execute了12个任务，在执行的过程中当前10个任务没有执行完时，
第11个和第12个任务会被阻塞到当有空余的线程数量时开始执行，从输出的结果上来看，第11个和第12个任务并没有在新创建线程来执行任务而是复用线程来执行，
我全部execute掉了以后我尝试调用了shutdown方法来关闭线程池，当然调用了以后并没有真正的关闭线程池，它会等待线程池中所有的任务
（包括阻塞队列中没有执行完的任务）都执行完在关闭，通过isShutdown可以获取到是否调用过shutdown方法，通过isTerminated获取线程池是否已经终止，
也就是停止掉了，此时返回的是false因为线程池中的任务还没有执行完，当过了5秒以后再次执行isTerminated方法返回true因为线程池已经完全shutdown。

# newSingleThreadPool

> 该方法创建只有一个线程的线程池，线程由于只有一个，所以execute的任务会一个一个的执行
```java
public static void main(String[] args) {
    ExecutorService executor = Executors.newSingleThreadExecutor();
    for (int i = 0; i < 12; i++) {
        executor.execute(new PrintTask(i+1, 3000L));
    }
    executor.shutdown();
}
```
# newCachedThreadPool

> 该线程池创建时在你的感知上是没有数量限制，因为它的实现，给予的最大空闲数量为Integer.MAX_VALUE,在执行任务时，如果没有多余的空闲线程执行该任务，
就会创建一个新的线程来执行这个任务
```java
public static void main(String[] args) {
    ExecutorService executor = Executors.newCachedThreadPool();
    for (int i = 0; i < 12; i++) {
        executor.execute(new PrintTask(i+1, 3000L));
    }
    executor.shutdown();
}
```
# newScheduledThreadPool
```java
public static void main(String[] args) {
    ScheduledExecutorService executor = Executors.newScheduledThreadPool(10);
    executor.schedule(new PrintTask(1, 3000L), 1L, TimeUnit.SECONDS);
    executor.scheduleAtFixedRate(new PrintTask(2, 2000L), 1, 1, TimeUnit.SECONDS);
    executor.scheduleWithFixedDelay(new PrintTask(3,3000L), 1, 1, TimeUnit.SECONDS);
}
```
首先创建了一个核心线程数量10的线程池，其最大空闲数量背后的实现仍旧是Integer.MAXVALUE，然后调用了schedule该方法是首先是只调度一次任务，
第一个参数为要调度的任务，第二参数为时间，第三个参数为时间单位，在接着使用了scheduleAtFixedRate这是相对频率重复调度任务的方法，
第一个参数为要调度的任务，第二个参数为首次开始的延时时间，第三个参数为相对上一个任务开始的延迟执行时间，第四参数为时间单位，
scheduleWithFixedDelay第一个参数为要调度的任务，第二个参数为首次延迟执行的时间，第三个参数为上一个任务执行结束后执行的间隔时间，第四个参数为时间单位


# 详解ThreadPoolExecutor
> 简要介绍重要的成员变量和几个方法
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    private final BlockingQueue<Runnable> workQueue;

    private final ReentrantLock mainLock = new ReentrantLock();

    private final HashSet<Worker> workers = new HashSet<>();

    private final Condition termination = mainLock.newCondition();

    private int largestPoolSize;

    private long completedTaskCount;

    private volatile ThreadFactory threadFactory;

    private volatile RejectedExecutionHandler handler;

    private volatile long keepAliveTime;

    private volatile boolean allowCoreThreadTimeOut;

    private volatile int corePoolSize;

    private volatile int maximumPoolSize;

    //...........省略多行代码........... //
    
    public void execute(Runnable command) {
    //...........外层用于往线程池提交线程的方法........... //
    }
    
    private boolean addWorker(Runnable firstTask, boolean core) {
    //...........真正添加线程的方法........... //
    }
    
    final void runWorker(Worker w) {
    //...........真正执行线程的方法........... //  
    }
    
}
```
