### 线程池

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015095326131.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




#### 结构

##### Executor

Executor 提供了一种将“任务提交”与“任务执行”分离开来的机制，它仅提供了一个 `#execute(Runnable command)` 方法，用来执行已经提交的 Runnable 任务。

```java
public interface Executor {
    void execute(Runnable command);  
}
```



##### ExcutorService

ExcutorService 拓展了一些方法，为了让提交执行模式更加完善，比如提交任务的：submit、invokeAll、invokeAny，但拓展这些提交方法最终调用的还是 excute 方法

```java
public interface ExecutorService extends Executor {

    /**
     * 关闭请求，会执行完以前提交的任务，但不接受新任务
     */
    void shutdown();

    /**
     * 关闭请求，试图停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待执行的任务列表
     */
    List<Runnable> shutdownNow();

    /**
     * 如果此执行程序已关闭，则返回 true。
     */
    boolean isShutdown();

    /**
     * 如果关闭后所有任务都已完成，则返回 true
     */
    boolean isTerminated();

    /**
     * 请求关闭、发生超时或者当前线程中断，无论哪一个首先发生之后，都将导致阻塞，直到所有任务完成执行
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    // ========== 提交任务 ==========

    /**
     * 提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future
     */
    <T> Future<T> submit(Callable<T> task);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    Future<?> submit(Runnable task);

    /**
     * 执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    /**
     * 执行给定的任务，当所有任务完成或超时期满时（无论哪个首先发生），返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 执行给定的任务，如果某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    /**
     * 执行给定的任务，如果在给定的超时期满前某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```



##### AbstractExecutorService

抽象类，实现 ExecutorService 接口，提供部分默认实现

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015095353345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




#### Future 体系

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015095307623.png#pic_center)



##### Future

异步计算的顶层接口，Future 对具体的 Runnable 或者 Callable 任务提供了三种操作：

- 执行任务的取消
- 查询任务是否完成
- 获取任务的执行结果

```java
public interface Future<V> {

    /**
     * 试图取消对此任务的执行
     * 如果任务已完成、或已取消，或者由于某些其他原因而无法取消，则此尝试将失败。
     * 当调用 cancel 时，如果调用成功，而此任务尚未启动，则此任务将永不运行。
     * 如果任务已经启动，则 mayInterruptIfRunning 参数确定是否以中断的方式停止任务
     */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * 如果在任务正常完成前将其取消，则返回 true
     */
    boolean isCancelled();

    /**
     * 如果任务已完成，则返回 true
     */
    boolean isDone();

    /**
     * 等待计算完成，然后获取其结果
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * 最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```



##### RunnableFuture

继承 Future、Runnable 两个接口，为两者的合体，即所谓的 Runnable 的 Future 。因此就可以作为 Thread 的 Runable 的参数了

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    //在未被取消的情况下，将此 Future 设置为计算的结果
    void run();
}
```



##### FutureTask

Future 体系的核心

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015095411671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)
