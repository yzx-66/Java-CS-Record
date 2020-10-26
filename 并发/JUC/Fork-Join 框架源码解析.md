### Fork / Join

#### 使用

- ForkJoinTask：我们要使用 ForkJoin 框架，必须首先创建一个 ForkJoin 任务。它提供在任务中执行 fork() 和 join() 操作的机制，通常情况下我们不需要直接继承 ForkJoinTask 类，而只需要继承它的子类，Fork/Join 框架提供了以下两个子类：
  - RecursiveAction：用于**没有返回结果**的任务。
  - RecursiveTask ：用于**有返回结果**的任务。
- ForkJoinPool ：ForkJoinTask 需要通过 ForkJoinPool 来执行，任务分割出的**子任务会添加到当前工作线程所维护的双端队列（数组）**中，**进入当前工作线程队列的头部**。**当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务**。

```java
/**
 * 实例：并行求和
 */
class SumTask extends RecursiveTask<Long> {
    static final int THRESHOLD = 500;
    long[] array;
    int start;
    int end;

    SumTask(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // 如果任务足够小,直接计算:
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += this.array[i];
            }
            return sum;
        }
        
        // 任务太大,一分为二:
        int middle = (end + start) / 2;
        System.out.println(String.format("split %d~%d ==> %d~%d, %d~%d", 
                                         start, end, start, middle, middle, end));
        
        // 创建子任务
        SumTask subtask1 = new SumTask(this.array, start, middle);
        SumTask subtask2 = new SumTask(this.array, middle, end);
        
        // 执行子任务
        subtask1.fork();
        subtask2.fork();
  
        // 获得子任务结果（很像 FutureTask.get()）
        Long subresult1 = subtask1.join();
        Long subresult2 = subtask2.join();
        
        // 合并结果
        Long result = subresult1 + subresult2;
        System.out.println("result = " + subresult1 + " + " + subresult2 + " ==> " + result);
        
        return result;
    }
}


public class Main {
    public static void main(String[] args) throws Exception {
        // 创建2000个随机数组成的数组:
        long[] array = new long[2000];
        for (int i = 0; i < array.length; i++) {
            array[i] = random();
        }
        
        // fork/join:
        ForkJoinTask<Long> task = new SumTask(array, 0, array.length);
        // 启动新线程执行 Fork/Join
        Long result = ForkJoinPool.commonPool().invoke(task);
    }

    static Random random = new Random(0);
    static long random() {
        return random.nextInt(10000);
    }
}
```



##### 源码

###### fork()

```java
public final ForkJoinTask<v> fork() {  
    // 把当前 task 提交到当前线程的任务队列里
    ((ForkJoinWorkerThread) Thread.currentThread()).pushTask(this);     
    // 返回当前任务，原理相当于 FutrueTask，返回一个可以获取结果回调，
    // 但实际上不返回也可以，因为返回的就是任务调用对象
    return this; 
} 


/**
 * ForkJoinWorkerThread#pushTask
 */
final void pushTask(ForkJoinTask t) {
   ForkJoinTask[] q; int s, m;
   if ((q = queue) != null) {    // ignore if queue removed
       long u = (((s = queueTop) & (m = q.length - 1)) << ASHIFT) + ABASE;
       // 把任务加入当前线程的任务队列，其他线程获取任务时都是从另一端获取
       UNSAFE.putOrderedObject(q, u, t);
       queueTop = s + 1;         // or use putOrderedInt
       if ((s -= queueBase) <= 2)
           // 唤醒线程池的一个空闲线程 去执行该线程的任务队列里的任务
           // 这也是所谓的 fork / join 任务窃取的体现，别的线程来执行其他线程任务队列的任务
           pool.signalWork();
   	   else if (s == m)
           // 扩容当前线程任务队列
           growQueue();
    }
}
```



###### join()

```java
public final V join() {
    // 得到当前任务的状态来判断返回什么结果
    // 状态有四种：已完成（NORMAL），被取消（CANCELLED），信号（SIGNAL）和出现异常（EXCEPTIONAL）。
    if (doJoin() != NORMAL)
        // 抛出异常
        return reportResult();
    else
        // 获取结果
        return getRawResult();
}


private int doJoin() {
    Thread t; ForkJoinWorkerThread w; int s; boolean completed;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
       // 首先通过查看任务的状态
       // 如果执行完了，则直接返回任务状态
       if ((s = status) < 0)
			return s;
       
        // 如果没有执行完，则从任务数组里取出任务自己，然后用当前线程执行
       if ((w = (ForkJoinWorkerThread)t).unpushTask(this)) {
           try {
               // 执行该任务
               completed = exec();
           } catch (Throwable rex) {
               // 如果出现异常，则纪录异常，并将任务状态设置为 EXCEPTIONAL
               return setExceptionalCompletion(rex);
           }
           if (completed)
               // 如果任务顺利执行完成了，则设置任务状态为 NORMAL
               return setCompletion(NORMAL);
       }
       return w.joinTask(this);
   }
   else
       return externalAwaitDone();
}


private V reportResult() {
   int s; Throwable ex;
    
    // 状态是被取消，则直接抛出 CancellationException
   if ((s = status) == CANCELLED)
       throw new CancellationException();
    
    // 状态是抛出异常，则直接抛出对应的异常
	if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)
       UNSAFE.throwException(ex);
      
    return getRawResult();
}
```
