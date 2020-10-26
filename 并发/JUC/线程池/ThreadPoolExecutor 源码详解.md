#### ThreadPoolExecutor

##### 属性

```java
/**
 * 高 3 位是 线程池状态
 * 后 29 位是 工作线程个数
 */
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
// 获得线程池状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 获得工作线程个数
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 通过线程池状态 和 工作线程个数 得到 ctl
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

有五种状态

* RUNNING：处于RUNNING状态的线程池能够接受新任务，以及对新添加的任务进行处理。
* SHUTDOWN：处于SHUTDOWN状态的线程池不可以接受新任务，但是可以对已添加的任务进行处理。
* STOP：处于STOP状态的线程池不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。
* TIDYING：当所有的任务已终止，ctl记录的"工作线程数量"为 0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现
* TERMINATED：线程池彻底终止的状态。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015095515874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



##### 构造方法

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

**corePoolSize**

* 线程池中核心线程的数量。当提交一个任务时，线程池会新建一个线程来执行任务，直到当前线程数等于corePoolSize。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。

**maximumPoolSize**

* 线程池中允许的最大线程数。线程池的阻塞队列满了之后，如果还有任务提交，如果当前的线程数小于maximumPoolSize，则会新建线程来执行任务。注意，如果使用的是无界队列，该参数也就没有什么效果了。

**keepAliveTime**

* 线程空闲的时间。线程的创建和销毁是需要代价的。线程执行完任务后不会立即销毁，而是继续存活一段时间：keepAliveTime。默认情况下，该参数只有在线程数大于corePoolSize时才会生效。

**unit**

* keepAliveTime的单位。TimeUnit

**workQueue**

* 用来保存等待执行的任务的阻塞队列，等待的任务必须实现Runnable接口。我们可以选择如下几种：
  * ArrayBlockingQueue：基于数组结构的有界阻塞队列，FIFO
  * LinkedBlockingQueue：基于链表结构的有界阻塞队列，FIFO。
  * SynchronousQueue：不存储元素的阻塞队列，每个插入操作都必须等待一个移出操作，反之亦然。
  * PriorityBlockingQueue：具有优先界别的阻塞队列。

**threadFactory**

* 用于设置创建线程的工厂,通过newThread() 方法提供创建线程的功能，newThread() 方法创建的线程都是 “非守护线程” 而且 “线程优先级都是Thread.NORM_PRIORITY ”。

* 默认的 Executors.defaultThreadFactory()，如下：

  ```java
  public static ThreadFactory defaultThreadFactory() {
      return new DefaultThreadFactory();
  }
  ```

  返回的是 DefaultThreadFactory对象，源码如下：

  ```java
  static class DefaultThreadFactory implements ThreadFactory {
      private static final AtomicInteger poolNumber = new AtomicInteger(1);
      private final ThreadGroup group;
      private final AtomicInteger threadNumber = new AtomicInteger(1);
      private final String namePrefix;
  
      DefaultThreadFactory() {
          SecurityManager s = System.getSecurityManager();
          group = (s != null) ? s.getThreadGroup() :
                                Thread.currentThread().getThreadGroup();
          namePrefix = "pool-" +
                        poolNumber.getAndIncrement() +
                       "-thread-";
      }
  
      public Thread newThread(Runnable r) {
          // 线程的名字
          Thread t = new Thread(group, r,
                                namePrefix + threadNumber.getAndIncrement(),
                                0);
          if (t.isDaemon())
              // 非守护线程，即 只有非守护线程都执行完了，jvm 才会退出
              t.setDaemon(false);
          if (t.getPriority() != Thread.NORM_PRIORITY)
              // 优先级
              t.setPriority(Thread.NORM_PRIORITY);
          return t;
      }
  }
  ```

  

**handler**

* RejectedExecutionHandler，线程池的拒绝策略。所谓拒绝策略，是指将任务添加到线程池中时，线程池拒绝该任务所采取的相应策略。当向线程池中提交任务时，如果此时线程池中的线程已经饱和了，而且阻塞队列也已经满了，则线程池会选择一种拒绝策略来处理该任务。 

* 线程池提供了四种拒绝策略： 
  * AbortPolicy：直接抛出异常，默认策略；
  * CallerRunsPolicy：用调用者所在的线程来执行任务；
  * DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
  * DiscardPolicy：直接丢弃任务； 
  * 当然我们也可以实现自己的拒绝策略，例如记录日志等等，实现RejectedExecutionHandler接口即可。



##### Executors创建

###### FixedThreadPool

* 可重用固定线程数的线程池，即 核心线程数 = 最大线程数，队列是无界阻塞

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```



###### **SingleThreadExecutor**

* 只有一个线程的线程池，即 核心线程数 = 最大线程数 = 1，队列是无界阻塞

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```



###### **CachedThreadPool**

* 不限制线程数的线程池，即 核心线程数 = 0 ， 最大线程数是 Integer.MAX，队列是不存储元素的 SynchronousQueue

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```



##### execute流程

因为前面说过了，其他的提交方法 submit、invokeAny 和 invokeAll 最终调用的都是 execute 去执行任务的提交

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    
    // <1> 如果线程池当前线程数小于 corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        // addWorker 创建新 coreszize 区间内的线程执行任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    
    // <2> 到这说明工作线程个数已经达到了 coreSize
    // 如果线程池处于 RUNNING 状态，则尝试加入阻塞队列
    // 注意一下，如果入队失败就直接返回了，不会阻塞等待，可以考虑部分场景任务必须执行，那就重写一下 execute
    if (isRunning(c) && workQueue.offer(command)) {
        // 加入阻塞队列成功，则尝试进行Double Check
        int recheck = ctl.get();
        
        // 再次检查线程池是否是 RUNNING，不是的话从队列中移除
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果是 RUNNING 状态但工作线程个为 0
        else if (workerCountOf(recheck) == 0)
            // 创建 maxSize 区间的线程
            addWorker(null, false);
    }
    // <3> 线程池不是 RUNNING 状态了，或者入队失败
    // 尝试创建 maxSize 区间的线程
    else if (!addWorker(command, false))
        // 拒绝
        reject(command);
}
```



添加线程

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();

        // 获取当前线程状态
        int rs = runStateOf(c);

		
        if (rs >= SHUTDOWN &&  // rs >= SHUTDOWN ，表示当前线程处于SHUTDOWN ，STOP、TIDYING、TERMINATED状态
                ! (rs == SHUTDOWN && 
                        firstTask == null && // rs == SHUTDOWN , firstTask != null时不允许添加线程
                        ! workQueue.isEmpty())) // rs == SHUTDOWN ,firstTask == null，但workQueue.isEmpty() == true，不允许添加线程，
            return false;

        // <1> 内层循环，workerCount + 1
        for (;;) {
            // 线程数量
            int wc = workerCountOf(c);
            // 如果当前线程数大于线程最大上限CAPACITY  return false
            // 若core == true，则与corePoolSize 比较，否则与maximumPoolSize ，大于 return false
            if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize)) 
                return false;
            // worker + 1,成功跳出retry循环
            if (compareAndIncrementWorkerCount(c))
                break retry;

            // CAS add worker 失败，再次读取ctl
            c = ctl.get();

            // 如果状态不等于之前获取的state，跳出内层循环，继续去外层循环判断
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    
    // <2> 创建新线程
    try {

        // 新建线程：Worker
        w = new Worker(firstTask);
        // 当前线程
        final Thread t = w.thread;
        if (t != null) {
            // 获取主锁：mainLock
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {

                // 线程池状态
                int rs = runStateOf(ctl.get());

                // rs < SHUTDOWN ==> 线程处于 RUNNING状态
                // 或者线程池处于 SHUTDOWN 状态，且firstTask == null（即 SHUTDOWN 状态如果队列还有任务，可以创建线程，
                // 但是不可再新增任务）
                if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {

                    // 当前线程已经启动，抛出异常
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();

                    // workers是一个HashSet<Worker>
                    workers.add(w);

                    // 设置最大的池大小largestPoolSize，workerAdded设置为true
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
            // 启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {

        // 线程启动失败
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}

/**
 * 处理工作线程启动失败
 */
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 从HashSet中移除该worker
        if (w != null)
            workers.remove(w);

        // 线程数 - 1
        decrementWorkerCount();
        // 有 worker 线程移除，如果是最后一个线程退出，那么把线程池状态更新为 Terminate
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}


/**
 * 当线程池涉及到要移除 worker 时候都会调用 tryTerminate()，
 * 该方法主要用于判断线程池中的线程是否已经全部移除了，如果是的话则把线程池状态更新为 Terminate
 */
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        
        if (isRunning(c) || // 线程池处于Running状态
                runStateAtLeast(c, TIDYING) || // 线程池已经终止了
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())) // 线程池处于ShutDown状态，但是阻塞队列不为空
            return;

        // 执行到这里，就意味着线程池要么处于 STOP 状态，要么处于 SHUTDOWN 且阻塞队列为空，那么是可以彻底终止的
        // 但是这时如果线程池中还存在线程，则会推进线程的关闭
        if (workerCountOf(c) != 0) {
            // 只中断一个，目的是逐步中断线程池中线程
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 设置线程池状态为 TIDYING，然后执行钩子
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 模板方法（空实现）
                    terminated();
                } finally {
                    // 执行完了线程池终止的钩子方法，那么线程池状态转为彻底关闭 TERMINATED
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
    }
}

/**
 * 终止工作线程，onlyOne == true仅终止一个线程，否则终止所有线程。
 * onlyOne 的目的是为了逐渐关闭线程池，通常出现在
 */
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // for 是为了保证可以关闭一个
        for (Worker w : workers) {
            Thread t = w.thread;
            // 没中断，且没有在执行 task（因为获取了 worker 的 lock）
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    // 设置中断标识
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
##### **Woker**

worker 就是工作线程的 Runnable 参数对象，线程启动后会调用其 run()

```java
// 继承 AQS 主要是为了方便线程的中断处理，即不是为了防止有多个线程来争抢什么资源，
// 而是因为，如果当前线程在 getTask() 获取到 task ，要执行前会 aquire 独占锁，
// 外部要中断这个线程的时候，也会先获得独占锁，那么如果当前线程正在执行某个 task 
// 外部就会获取失败，反之同理，从而就可以起到个 正在执行任务 和 外部中断 互斥的作用（标识作用）。
private final class Worker extends AbstractQueuedSynchronizer
        implements Runnable {
    private static final long serialVersionUID = 6138294804551838833L;

    // task 的thread
    final Thread thread;

    // 运行的任务task
    Runnable firstTask;

    volatile long completedTasks;

    Worker(Runnable firstTask) {

        // 设置 AQS 的同步状态，-1 代表没有线程获取到同步状态，大于 0 代表锁已经被获取
        setState(-1);
        this.firstTask = firstTask;

        // 利用线程池的 ThreadFacotry 创建一个线程，并保存到 worker 中
        this.thread = getThreadFactory().newThread(this);
    }

    // 线程启动后就会执行 run 方法
    public void run() {
        runWorker(this);
    }

}
```



ThreadPoolExecutor#runWorker

```java
final void runWorker(Worker w) {

    // 当前线程
    Thread wt = Thread.currentThread();

    // 要执行的任务
    Runnable task = w.firstTask;

    w.firstTask = null;

    // 释放同步状态，即允许外部获取到同步状态，然后就可以中断当前线程
    w.unlock(); // allow interrupts
    // 这个是为了标识是否正常退出
    // 正常退出指 getTask() 获取不到任务然后退出，非正常退出是抛出异常退出
    boolean completedAbruptly = true;
    try {
        // <1> 获取要执行的 task
        while (task != null || (task = getTask()) != null) {
            // worker 获取同步状态，因为要开始执行任务了，不允许外部获取然后执行中断了
            w.lock();

          
            // <2> 判断是否要设置任务线程的中断标识  
            // 如果线程池状态 >= STOP ,且 当前线程没设置中断状态，则 wt.interrupt()
            if ((runStateAtLeast(ctl.get(), STOP) || 
                    // 如果线程池状态 < STOP，但线程已经中断了，再次判断线程池是否 >= STOP，如果是 wt.interrupt()
                    (Thread.interrupted() && 
                            runStateAtLeast(ctl.get(), STOP))) && 
                    !wt.isInterrupted())
                // 注意：确定当前线程需要中断，只调用了wt.interrupt()，并不会退出循环，该方法只会设置一个线程的标识而已
                // 要退出循环只要 getTask() 返回 null，或者执行 task 时抛出异常
                wt.interrupt();
            
            // <3> 执行 task
            try {
                // 模板方法（默认空实现），前置拓展
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 模板方法（默认空实现），后置拓展
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 完成任务数 + 1
                w.completedTasks++;
                // 释放锁
                w.unlock();
            }
        }
        // 获取不到任务正常退出
        completedAbruptly = false;
    } finally {
        // 处理线程退出
        processWorkerExit(w, completedAbruptly);
    }
}

/**
 * 获取任务
 */
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {

        // 线程池状态
        int c = ctl.get();
        int rs = runStateOf(c);

        // 线程池中状态 >= STOP 或者 线程池状态 == SHUTDOWN 且阻塞队列为空，则 worker - 1，return null
        // return null 之后就会正常退出 while 循环，然后结束该线程
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // 在这执行了 - 1，processWorkerExit() 就不会再 -1 
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 判断是否需要超时控制（即是否大于核心线程数，超时没获取到的话就会返回 null，然后关闭这个线程）
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 如果工作线程数超过最大线程数 或 （有超时控制并已经超时 且 （工作线程 > 1 或者 队列已空）） 
        // 则 工作线程数 -1 返回 null，目的是为了关闭那些 最大线程数范围内的线程
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 从阻塞队列中获取task
            // 如果需要超时控制，则调用 poll()，否则调用 take()
            Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
            
            if (r != null)
                return r;

            // 设置已超时
            timedOut = true;
        } catch (InterruptedException retry) {
            // 到这里说明被线程被中断了，所以被唤醒了
            // 如果是因为线程池关闭而中断，那么重新循环时就会因为线程池关闭而返回 null，之后让 processWorkerExit 移除该线程
            // 设置未超时，如果线程池未关闭，那么可以继续从阻塞队列获取任务
            timedOut = false;
        }
    }
}

/**
 * 处理线程退出
 */
private void processWorkerExit(Worker w, boolean completedAbruptly) {

    // true：用户线程运行异常,需要扣减
    // false：getTask方法中扣减线程数量
    if (completedAbruptly)
        decrementWorkerCount();

    // 获取主锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        // 从HashSet中移出worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    // 有 worker 线程移除，如果是最后一个线程退出，那么把线程池状态更新为 Terminate
    tryTerminate();

    int c = ctl.get();
    // 如果线程池为 running 或 shutdown 状态，即 tryTerminate() 没有成功终止线程池
    // 因为刚才删除了一个 worker，没有彻底终止的话会再判断是否有必要新增一个 worker
    if (runStateLessThan(c, STOP)) {
        // 正常退出，计算min：需要维护的最小线程数量（一般就是核心线程数）
        // 异常退出的话，不用判断是否达比核心线程少了，直接再新增一个线程
        if (!completedAbruptly) {
            // allowCoreThreadTimeOut 默认false：是否需要维持核心线程的数量
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            
            // 如果min ==0 或者workerQueue为空，min = 1
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;

            // 如果线程数量大于最少数量min，直接返回，不需要新增线程
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 说明结束一个线程后，还需要再添加一个线程，那么添加一个没有 firstTask 的 worker
        addWorker(null, false);
    }
}
```



##### 线程池关闭

* 不去中断正在执行 task 的工作线程

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 设置线程池状态为 SHUTDOWN
        advanceRunState(SHUTDOWN);
        // 中断空闲的线程（即没有执行 task 的线程）
        interruptIdleWorkers();
        // 交给子类实现
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}

private void interruptIdleWorkers() {
    // 中断所有没有正在执行  task 的线程
    interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            // 没中断，且没有在执行 task（通过获取 lock 来保证互斥）
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    // 如果没有执行 task，那么设置中断标识
                    // 注意一下，在 getTask 里，如果状态是 SHUTDOWN 但阻塞队列里还有任务的话，那么这个空闲线程先不会退出，
                    // 会再去工作队列获取任务,从而保证工作队列的任务执行完毕
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```



* 去中断所有线程（包括正在执行 task 的线程），并返回没有执行的 task

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 设置线程池状态为 STOP 
        advanceRunState(STOP);
        // 中断所有线程
        interruptWorkers();
        // 返回等待执行的任务列表，因为在 getTask 的时候，如果线程池状态是 STOP 那么直接返回 null，不会再去获取任务执行
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}

private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 这可以看出是不会获取 worker 的 lock 的，所以不在乎是否正执行 task
        for (Worker w : workers)
            // 设置其工作线程的中断标识
            // 注意一下，即使设置了工作线程的中断标识，但是 task 的 run 方法若对中断不敏感
            // 即不去判断是否中断 然后处理的话，那么还是要等该 task 的 run 方法执行完，在下一次 getTask 的时候
            // 因为线程池状态为 STOP 返回 null ，然后再由 processWorkerExit 移除线程
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}

private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    // 把 workQueue 的元素转移到 taskList
    q.drainTo(taskList);
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            // 从 workQueue 中删除该 task
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}
```
