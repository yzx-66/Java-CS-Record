#### ScheduledThreadPoolExecutor

虽然 Timer 与 TimerTask 可以实现线程的周期和延迟调度，但是对于定期、周期执行任务的调度策略存在一些缺陷，所以一般都是推荐ScheduledThreadPoolExecutor 来实现



##### 构造方法

都是利用ThreadLocalExecutor来构造的，但是

* 所使用的阻塞队列变成了 DelayedWorkQueue
  * DelayedWorkQueue 为 ScheduledThreadPoolExecutor 中的内部类，它其实和阻塞队列 DelayQueue 有点儿类似。
* 所有的最大线程数都变成  Integer.MAX_VALUE ，因为实际上在提交延时任务后会直接加入阻塞队列，然后会去判断是否要增加工作线程，实际上工作线程个数永远超不过 corePoolSize

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue());
}

public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue(), threadFactory);
}

public ScheduledThreadPoolExecutor(int corePoolSize,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue(), handler);
}


public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue(), threadFactory, handler);
}
```



##### Executors 创建

###### ScheduledThreadPool

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```



##### ScheduledExecutorService

ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor 之后，多拓展了这个接口的方法

###### 定义

```java
// 创建并执行 在给定延迟后启用的 一次性操作。
<V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit)

// 创建并执行 在给定延迟后启用的 一次性操作。
ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit)

// 创建并执行 一个在给定初始延迟后 首次启用的定期操作，后续操作具有给定的周期；
// 也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。
// 即 本次任务开始 和 下一次任务开始 的间隔为 period
ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)

// 创建并执行 一个在给定初始延迟后 首次启用的定期操作，随后，在每一次执行终止 和 下一次执行开始 之间都存在给定的延迟。
// 即 本次任务结束 和 下一次任务开始 的间隔为 delay
ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)
```



###### scheduleWithFixedDelay

四个方法的源码，会发现其实他们的处理逻辑都差不多，所以我们就挑scheduleWithFixedDelay方法来分析

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit) {
    // 校验，如果参数不合法则抛出异常
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    
    // 构造一个task，该task为ScheduledFutureTask
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      // 注意是 -delay ，为了和 scheduleAtFixedRate 区分
                                      unit.toNanos(-delay));
    
    // 构造一个 Future 作为定时任务的回调对象（留给子类实现的，默认是直接向上转型）
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    
    // 执行这个定时任务
    delayedExecute(t);
    return t;
}

private void delayedExecute(RunnableScheduledFuture<?> task) {
    // 如果线程池关闭
    if (isShutdown())
        reject(task);
    else {
        // 任务直接加入阻塞队列
        // 由此可见和 ThreadPoolExcecutor 的区别就是 task 的 run 方法
        super.getQueue().add(task);
       
        // 如果线程池关闭了 或者 任务异常
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            // futureTask 的状态设为 cancle
            task.cancel(false);
        else
            // 增加工作线程
            ensurePrestart();
    }
}

/**
 * 判断当前的线程池状态是否可以执行任务
 */
boolean canRunInCurrentRunState(boolean periodic) {
    return isRunningOrShutdown(periodic ?
                               continueExistingPeriodicTasksAfterShutdown :
                               executeExistingDelayedTasksAfterShutdown);
}

/**
 * ThreadPoolExecutor#remove
 */
public boolean remove(Runnable task) {
    boolean removed = workQueue.remove(task);
    tryTerminate(); // In case SHUTDOWN and now empty
    return removed;
}

/**
 * ThreadPoolExecutor#ensurePrestart
 */
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    if (wc < corePoolSize)
        addWorker(null, true);
    else if (wc == 0)
        addWorker(null, false);
}
```





##### ScheduledFutureTask

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015095625887.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




###### 属性

```java
/** 任务被添加到ScheduledThreadPoolExecutor中的序号 */
private final long sequenceNumber;

/** 任务要执行的具体时间 */
private long time;

/**任务的间隔 */
private final long period;
```



###### 构造方法

```java
ScheduledFutureTask(Runnable r, V result, long ns) {
    super(r, result);
    this.time = ns;
    this.period = 0;
    this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Runnable r, V result, long ns, long period) {
    super(r, result);
    this.time = ns;
    this.period = period;
    this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Callable<V> callable, long ns) {
    super(callable);
    this.time = ns;
    this.period = 0;
    this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Callable<V> callable, long ns) {
    super(callable);
    this.time = ns;
    this.period = 0;
    this.sequenceNumber = sequencer.getAndIncrement();
}
```



###### compareTo

用于构建最小堆

```java
// 首先按照time排序，time小的排在前面，大的排在后面，如果time相同，则使用sequenceNumber排序，小的排在前面，大的排在后面。
public int compareTo(Delayed other) {
    if (other == this) // compare zero if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
    return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
}
```



###### run

核心，ScheduledThreadPoolExecutor 就是通过 run 方法对任务进行调度和延迟的

```java
public void run() {
    boolean periodic = isPeriodic();
    // 当前线程池状态是否可以执行任务
    if (!canRunInCurrentRunState(periodic))
        // 设置 future 的 state 为 cancle
        cancel(false);
    // 不是周期任务
    else if (!periodic)
        // 调用FutureTask#run 执行任务
        ScheduledFutureTask.super.run();
    // 是周期任务
    // 调用 FutureTask#runAndReset 和 run 的区别是会把状态再次更新为 New，而不是 Complete
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        // outerTask = this
        reExecutePeriodic(outerTask);
    }
}

/**
 * 判断指定的任务是否为定期任务
 */
public boolean isPeriodic() {
    return period != 0;
}

/**
 * 重新计算任务的下次执行时间
 */
private void setNextRunTime() {
    long p = period;
    // 如果是 scheduleAtFixedRate
    if (p > 0)
        // 直接在上一次执行开始上增加
        time += p;
    // 否则是 scheduleAtFixedDelay
    else
        // 在此刻执行完的时间上加
        time = triggerTime(-p);
}
long triggerTime(long delay) {
    // 使用当前的时间加 delay
    return now() +
        ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}

/**
 * 加入阻塞队列，然后增加工作线程
 */
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
    if (canRunInCurrentRunState(true)) {
        super.getQueue().add(task);
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
```
