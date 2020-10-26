#### CyclicBarrier

它允许一组线程互相等待，直到到达某个**公共屏障点**，才会进行后续任务

* 内部是使用重入锁 ReentrantLock 及其 Condition 。
* 因为只有一个 await 操作，不像 countDownLatch 和 Semaphore 有释放操作，所以没法调用 release，那么就用 conditon 来阻塞，不用 CLH 队列



##### 属性

```java
/**
 * Generation 是 CyclicBarrier 内部静态类，描述了 CyclicBarrier 的更新换代。在CyclicBarrier中，同一批线程属于同一代 。
 * 当有 `parties` 个线程全部到达 barrier 时，`generation` 就会被更新换代。其中 `broken` 属性，标识该当前 CyclicBarrier 是否已经处于中断状态。
 */
private static class Generation {
    // 默认 barrier 是没有损坏的。
    boolean broken = false;
}


private final ReentrantLock lock = new ReentrantLock();

// 使用 condition 阻塞
private final Condition trip = lock.newCondition();

// 线程数
private final int parties;

// 所有线程到齐要执行的任务（由最后一个到的线程执行）
private final Runnable barrierCommand;

// 分代
private Generation generation = new Generation();

// 剩余线程数
private int count;
```



##### 方法

###### 构造方法

```java
// 创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动，但它不会在启动 barrier 时执行预定义的操作。
public CyclicBarrier(int parties) {
    this(parties, null);
}

// 创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动
// 并在启动 barrier 时执行给定的屏障操作 barrierAction ，该操作由最后一个进入 barrier 的线程执行。
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```



###### await

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);//不超时等待
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
        TimeoutException {
    // 获取锁，因为要使用 lock 的 condition 来阻塞线程
    final ReentrantLock lock = this.lock;
    // 获取锁成功才可以使用 condition
    lock.lock();
    try {
        // 分代，同时用来检查是否被其他线程将 broken 设为 true 了
        final Generation g = generation;
		
        // 当前generation“已损坏”，抛出BrokenBarrierException异常
        // 抛出该异常一般都是某个线程在等待某个处于“断开”状态的 CyclicBarrie
        if (g.broken)
            //当某个线程试图等待处于断开状态的 barrier 时，或者 barrier 进入断开状态而线程处于等待状态时，抛出该异常
            throw new BrokenBarrierException();

        //如果线程中断，终止CyclicBarrier
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        // 进来一个线程 count - 1，当 count 变为 0 的时候，会由那个线程执行 condition.signal
        int index = --count;
        //count == 0 表示所有线程均已到位，触发Runnable任务
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                //触发任务
                if (command != null)
                    command.run();
                ranAction = true;
                //唤醒所有等待线程，并更新generation
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction) // 未执行，说明 barrierCommand 执行报错，或者线程打断等等情况。
                    breakBarrier();
            }
        }
		
        // 走到这就说明 count 没有减为 0，那就要用 condition 来阻塞
        for (;;) {
            try {
                // 如果不是超时等待，则调用 Condition.await()方法等待
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    // 超时等待，调用 Condition.awaitNanos()方法等待，并获得等待的时间
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }

            // 到这说明所有线程都到期了，这个线程进入 CLH 队列后已经成功被唤醒了
            // 虽然 ReentrantLock 是排他锁，但是已经剩下没几步，其他还在 CLH 队列的也等不了多久
            if (g.broken)
                // 如果被 broken 了，那抛异常
                throw new BrokenBarrierException();

            // generation已经更新，返回index
            if (g != generation)
                return index;

            // “超时等待”，并且时间已到,终止CyclicBarrier，并抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        //释放锁
        lock.unlock();
    }
}

// 某个线程检查到被中断会调用这个方法，设置屏障已经破坏
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}


// 当所有线程都已经到达 barrier 处（index == 0），则最后一个到达的线程会执行下面的方法
private void nextGeneration() {
    //唤醒所有线程。
    trip.signalAll();
    //重置 count 。
    count = parties;
    //重置 generation 。
    generation = new Generation();
}
```



###### reset

```java
// 重置 barrier 到初始化状态
// 通过组合 #breakBarrier() 和 #nextGeneration() 方法来实现。
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 这里不会导致其他在 condition 的线程抛出异常，因为它们还在 wait，要唤醒后才会检查是否 borken
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```



###### getNumberWaiting

```java
// 获得等待的线程数
public int getNumberWaiting() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return parties - count;
    } finally {
        lock.unlock();
    }
}
```



###### isBroken

```java
// 判断 CyclicBarrier 是否处于中断
public boolean isBroken() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return generation.broken;
    } finally {
        lock.unlock();
    }
}
```



##### 应用场景

多个线程相互等待的场景

* 例如多线程结果合并的操作，用于多线程计算数据，最后合并计算结果的应用场景。
