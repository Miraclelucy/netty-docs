
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


# 详解ThreadPoolExecutor成员变量
> 简要介绍重要的成员变量
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
> 成员变量详解

   - ctl

    该变量是一个复合含义的变量，其本身可以看作是一个Integer变量，该变量的高3位代表线程池的状态，
    那么后29位（从低位往高位数）代表该线程池数量

   - COUNT_BITS

    数量的位数

   - COUNT_MASK

    数量位数的掩码

   - RUNNING

    表示运行中的状态标识

   - SHUTDOWN

    表示关闭中的状态标识

   - STOP

    表示已停止的状态标识

   - TIDYING

    表示当前所有任务已经终止，任务数量为0时的状态标识

   - TERMINATED

    表示线程池已经完全终止（关闭），关于线程池的关闭状态

    线程池状态变换

   - workQueue

    用来保存等待任务执行的阻塞队列

   - mainLock

    可重入锁，方法里面会大量使用，很多变量的操作都需要使用该锁

   - workers

    该集合中包含了所有在工作的线程

   - termination

    锁条件队列，主要用于awaitTermination

   - largestPoolSize

    记录线程池最大工作线程的数量（可能是个历史值）

   - completedTaskCount

    完成任务的计时器，仅在中止工作任务时更新

   - threadFactory

    用于创建线程的工厂

   - handler

    > 饱和策略的回调，当队列已满且线程个数达到maximumPoolSize时采取的策略

    有以下几种策略

       - AbortPolicy

       - CallerRunsPolicy

       - DiscardOldestPolicy

       - DiscardPolicy

        分别为：抛出异常、使用调用者当前的线程来执行任务、调用队列的poll丢弃一个任务，执行当前任务、默默丢弃该任务。

   - keepAliveTime

    空闲存活时间，如果线程池中的线程数量比核心线程数量还要多时，并且多出的这些线程都是闲置状态，
    该变量则是这些闲置状态的线程的存活时间啊

   - allowCoreThreadTimeOut

    默认为false，即时是空闲核心线程也会处于活动状态，如果设为true那么核心线程也会遵循keepAliveTime的时间来做闲置处理

   - corePoolSize

    线程池核心线程数量

   - maximumPoolSize

    线程池最大线程数量

# 详解ThreadPoolExecutor中execute方法

```java
public void execute(Runnable command) {
    if (command == null) // 1
        throw new NullPointerException();
    int c = ctl.get(); // 2
    if (workerCountOf(c) < corePoolSize) { // 3
        if (addWorker(command, true)) // 3.1
            return;
        c = ctl.get(); // 3.2
    }
    if (isRunning(c) && workQueue.offer(command)) { // 4
        int recheck = ctl.get(); // 5
        if (! isRunning(recheck) && remove(command)) //6
            reject(command); 
        else if (workerCountOf(recheck) == 0) // 7
            addWorker(null, false);
    }
    else if (!addWorker(command, false)) //8
        reject(command);
}
```
接下来按照注释的序号对其一一解释
  1. 提交的任务如果是个空的则抛出NullPointerException
  2. 获取复合变量（记录了线程池状态和当前线程池线程数量）
  3. 判断当前线程池的线程数量是否在限定的核心线程数量的访问楼内
      a. 如果在那么就直接调用addWorker添加一个核心线程，然后return
      b. 添加失败，重新获取复合变量
  4. 判断线程池是否是运行状态并且添加至阻塞等待队列中
  5. 重新获取状态（可能添加的过程中关闭过线程池之类的并发操作）
  6. 判断线程池是否是运行状态，如果不是将添加的任务删除并采取拒绝措施
  7. 判断线程池中的工作线程数量是否为0，如果为空则添加一个工作线程
  8. 队列添加失败，尝试调用addWorker以非核心线程的方式添加一条非核心线程执行，失败则采用饱和策略拒绝该任务
从上面的源码可以看到，如果核心线程数量未达到限定范围则会优先创建核心线程来执行该任务，否则将其加入阻塞等待队列中，如果添加至阻塞等待队列中失败后，则尝试创建一个非核心线程来执行该任务如果失败则采用饱和策略，该方法大量都与addWorker方法相关。


# 详解ThreadPoolExecutor中addWorker方法

```java

private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) { // 1
        int c = ctl.get(); // 2
        int rs = runStateOf(c); // 3
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty())) // 4
            return false;

        for (;;) { // 5
            int wc = workerCountOf(c); // 6
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize)) // 7
                return false;
            if (compareAndIncrementWorkerCount(c)) // 8
                break retry;
            c = ctl.get(); // 9
            if (runStateOf(c) != rs) // 10
                continue retry;
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask); // 11
        final Thread t = w.thread; // 12
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock; // 13
            mainLock.lock(); // 14 
            try {
                int rs = runStateOf(ctl.get()); // 15

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) { // 16
                    if (t.isAlive()) // 17
                        throw new IllegalThreadStateException();
                    workers.add(w); // 18
                    int s = workers.size(); // 19
                    if (s > largestPoolSize) // 20
                        largestPoolSize = s;
                    workerAdded = true; // 21
                }
            } finally {
                mainLock.unlock(); // 22
            }
            if (workerAdded) {  // 23
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted) // 24
            addWorkerFailed(w);
    }
    return workerStarted; // 25
}
```
以下是对上述代码的分析
  1. 开始自旋
  2. 获取复合状态
  3. 拿到当前线程池运行状态
  4. 判断在必要时检查队列是否为空
      a. 线程池处于SHUTDOWN时并且有第一个任务时
      b. 当前线程池为STOP、TIDYING、TERMINATED
      c. 当前线程池为SHUTDOWN时且任务队列为空时
      d. 以上三种情况都会返回false
  5. 开启第二轮自旋（其实第一轮自旋就只是检测运行状态）
  6. 获取线程数量
  7. 判断当前线程数量是否超出了最大容量限制或判断当前线程数量是否大于核心线程数或者最大线程数，具体判断是判断核心线程数还是最大线程数取决于调用时传入的core是否为true，如果超出了直接返回false
  8. CAS增加当前线程数量，更改成功结束自旋
  9. 重新获取复合状态
  10. 判断当前线程池状态是否还是运行中，如果不是则跳过第一层自旋的第一次自旋开始第二次
  11. 创建工作线程
  12. 获取工作者中的线程对象
  13. 拿到锁变量
  14. 尝试获取锁（在操作队列时采取同步措施）
  15. 获取当前线程池运行状态
  16. 判断线程池是否在运行状态，否则判断是否是SHUTDOWN状态且传入的任务为空（有时是只启动一个工作线程）
  17. 判断线程是否是alive状态
  18. 添加工作线程队列
  19. 取当前工作线程队列的数量
  20. 判断是否大于最大线程数量，如果大于则赋值给它
  21. 设置当前工作者加入线程队列的已添加的标识为true
  22. 释放锁
  23. 判断当前工作线程是否已经加入工作线程队列，如果以加入则启动该工作线程，并设置启动标识为true
  24. 判断工作线程启动标识如果为false则调用addWorkerFailed
  25. 返回启动标识来决定是否添加成功
步骤还挺多的，简单的总结一下，首先自旋的去增加工作者线程的数量，然后创建工作者（工作线程），然后涉及到队列的操作获取到锁然后添加到工作线程队列设置标识，如果未添加到线程队列中，该工作线程也不会启动，如果添加了那么启动该工作线程，然后设置启动标识，最后返回启动标识
   
   

# 详解ThreadPoolExecutor中runWorker方法

从源码中可以看出，将行为委托给了runWorker并将自身实例传递了过去，runWorker是在ThreadPoolExecutor中定义的，其实这里主要是做的职责分离（不理解这个做法也无所谓完全无伤大雅）
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread(); // 1
    Runnable task = w.firstTask; // 2
    w.firstTask = null; // 3
    w.unlock(); // 4
    boolean completedAbruptly = true; // 5
    try {
        while (task != null || (task = getTask()) != null) { // 6
            w.lock(); // 7
            if ((runStateAtLeast(ctl.get(), STOP) || // 8
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task); // 9
                try {
                    task.run();  // 10
                    afterExecute(task, null); // 11
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock(); // 12
            }
        }
        completedAbruptly = false; // 13
    } finally {
        processWorkerExit(w, completedAbruptly); // 14 
    }
}
```
  1. 获取当前线程
  2. 拿到工作线程中的任务
  3. 将工作线程对象中的任务清空
  4. unlock设置锁状态，这里主要还是设置状态为可中断
  5. 设置一个标识，我习惯将这个表示称为“猝死”标识，它主要是标识这个线程是不是正常执行完，是不是意外中断了或者是执行
  6. 自旋判断当前任务是否为空，如果为空，则调用getTask拿去一个任务
  7. 获取锁
  8. 判断状态是否是STOP或者被中断了，如果是则发出中断请求
  9. 方法执行之前的一些Hook函数，说白了该函数啥都没干，留给子类扩展
  10. 运行任务里面的逻辑
  11. 方法执行之后的一些Hook函数
  12. 解锁
  13. 设置“猝死”标识为false没有被猝死233333
  14. 调用processWorkerExit故名思意，其实其背后就是去删除了workers中的当前退出工作线程对象和修改数量



# 详解ThreadPoolExecutor中getTask()方法

在getTask方法中除了从阻塞队列中拿去一个任务以外，还有一个作用就是维持当前线程活下去

```java
private Runnable getTask() {
    boolean timedOut = false; // 1

    for (;;) {
        int c = ctl.get();

        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize; // 2

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) { // 3
            if (compareAndDecrementWorkerCount(c)) // 4
                return null;
            continue; // 5
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take(); // 6
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
  1. 超时标志
  2. 定时标志，首先判断是否允许核心线程超时(默认false)然后判断当前线程池线程数量是否大于核心线程数
  3. 判断当前线程池数是否超过了最大线程数 || 当前线程是否是定时并且已经超时 并且 线程数大于1 或 任务队列为空
  4. 线程数减1，返回null
  5. 跳过本轮自旋
  6. 从队列中拿取任务
其实所谓的核心线程就是保持它启动后保证在核心线程数内的线程不会挂掉一直在自旋，但如果是设置了allowCoreThreadTimeOut标志为true的话那么就意义不大了
