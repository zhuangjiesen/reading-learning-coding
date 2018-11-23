# ThreadPoolExecutor.java 源码解析


### 用途

```
 * <p>Thread pools address two different problems: they usually
 * provide improved performance when executing large numbers of
 * asynchronous tasks, due to reduced per-task invocation overhead,
 * and they provide a means of bounding and managing the resources,
 * including threads, consumed when executing a collection of tasks.
 * Each {@code ThreadPoolExecutor} also maintains some basic
 * statistics, such as the number of completed tasks.

 翻译：
 线程池解决2个问题：
 1.在执行大量异步任务时，提升了性能，减少了每个任务调用的负载；
 2.线程池是当执行一系列任务时，用来限制和管理资源，包括多线程的一种手段；
 每个线程池都保存一些基本信息，比如完成的任务数

```

### 关键点概念：
```
1.ctl(AtomicInteger): 线程池状态管理，是一个原子的整型变量
它能表示2种东西，通过位运算 
    1.当前运行的线程数 后29位 最大5个亿(2的29次方 - 1)
    2.当前线程池的状态 前3位 
2.workQueue(BlockingQueue<Runnable>): 用来保存队列的，当任务数大于 corePoolSize 时,直接通过 workQueue.offer() 方法加入到队列；
3.workers(HashSet<Worker>): 保存线程池中的worker 的集合，；
4.corePoolSize(int): 核心线程池大小；
5.maximumPoolSize(int): 线程池最大容量，最大值为 2的29次方 - 1；
6.handler(RejectedExecutionHandler):RejectedExecutionHandler接口，当任务数量大于 maximumPoolSize 时触发,默认 AbortPolicy.class 直接抛异常；
7.threadFactory(ThreadFactory): 通过实现ThreadFactory接口，创建线程的接口，可以用来设置Thread的属性；
8.keepAliveTime(long): 线程worker执行完任务后 去queue.poll(keepAliveTime) 获取任务，超时就把worker关闭；

```


### Worker.java 
#### 线程池中用来执行任务的线程对象，继承了AQS（AbstractQueuedSynchronizer），实现了线程锁的功能

```

 private final class Worker
 		/** 继承AQS 实现了线程锁 */
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        /** 当前worker真正运行的线程，如果是null，表示threadFactory创建失败 */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        /** 初始化Worker的时候，第一个任务，之后就是null了 */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
    
    
    
..... 省略部分代码

    
```

#### runWorker() 方法
```


    /**
     * Main worker run loop.  Repeatedly gets tasks from queue and
     * executes them, while coping with a number of issues:
     *
     * 1. We may start out with an initial task, in which case we
     * don't need to get the first one. Otherwise, as long as pool is
     * running, we get tasks from getTask. If it returns null then the
     * worker exits due to changed pool state or configuration
     * parameters.  Other exits result from exception throws in
     * external code, in which case completedAbruptly holds, which
     * usually leads processWorkerExit to replace this thread.
     *
     * 2. Before running any task, the lock is acquired to prevent
     * other pool interrupts while the task is executing, and then we
     * ensure that unless pool is stopping, this thread does not have
     * its interrupt set.
     *
     * 3. Each task run is preceded by a call to beforeExecute, which
     * might throw an exception, in which case we cause thread to die
     * (breaking loop with completedAbruptly true) without processing
     * the task.
     *
     * 4. Assuming beforeExecute completes normally, we run the task,
     * gathering any of its thrown exceptions to send to afterExecute.
     * We separately handle RuntimeException, Error (both of which the
     * specs guarantee that we trap) and arbitrary Throwables.
     * Because we cannot rethrow Throwables within Runnable.run, we
     * wrap them within Errors on the way out (to the thread's
     * UncaughtExceptionHandler).  Any thrown exception also
     * conservatively causes thread to die.
     *
     * 5. After task.run completes, we call afterExecute, which may
     * also throw an exception, which will also cause thread to
     * die. According to JLS Sec 14.20, this exception is the one that
     * will be in effect even if task.run throws.
     *
     * The net effect of the exception mechanics is that afterExecute
     * and the thread's UncaughtExceptionHandler have as accurate
     * information as we can provide about any problems encountered by
     * user code.
     *
     * @param w the worker
     大致翻译：
     	我们可以开启一个初始化任务，而不需要重新初始化线程，只要线程池在运行，就可以从getTask() 方法中获取任务；
    	每个任务都会触发beforeExecute()；
     	每个任务都会触发 afterExecute(task, thrown) 方法；
     */
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                    	//执行线程主体任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }




    /**
     * Performs blocking or timed wait for a task, depending on
     * current configuration settings, or returns null if this worker
     * must exit because of any of:
     * 1. There are more than maximumPoolSize workers (due to
     *    a call to setMaximumPoolSize).
     * 2. The pool is stopped.
     * 3. The pool is shutdown and the queue is empty.
     * 4. This worker timed out waiting for a task, and timed-out
     *    workers are subject to termination (that is,
     *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
     *    both before and after the timed wait, and if the queue is
     *    non-empty, this worker is not the last thread in the pool.
     *
     * @return task, or null if the worker must exit, in which case
     *         workerCount is decremented

     翻译：
     阻塞或停留时间从queue中获取任务，取决于当前设置
     当getTask() 返回空时，表示这个worker需要停止
     原因：
     1.当前worker总数大于 maximumPoolSize
     2.线程池停止
     3.线程池停止，队列为空
     4.queue.poll() 获取超时，没有获取到任务


     */
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

```


#### execute(Runnable) 执行任务



```


    /**
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@code RejectedExecutionHandler}.
     *
     * @param command the task to execute
     * @throws RejectedExecutionException at discretion of
     *         {@code RejectedExecutionHandler}, if the task
     *         cannot be accepted for execution
     * @throws NullPointerException if {@code command} is null

     大致翻译：
     给定的任务之后会被执行，任务在一个新的线程或者已经存在线程池中的线程中执行。
     任务无法执行，或者线程池已经停止，或者已经达到线程容量，会触发 RejectedExecutionHandler
     抛空指针异常当 command(传入的线程) 为空
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.

         大致翻译：
         1.小于 corePoolSize 时，创建新的线程执行 ，调用 addWorker()方法
         2.当大于等于 corePoolSize 时，添加到队列中，如果成功，作 double-check ，判断运行状态
         	1.当线程池停止运行，删除队列元素
         	2.线程池中没有worker，创建一个新的空firstTask 的worker,它会直接去queue的任务
         3.如果无法添加到队列，并小于 maximumPoolSize 时，直接创建新的线程worker 
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
        	//直接创建线程 ， 第二个参数是方法addWorker() 中用来判断core
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
            	//直接创建线程 ， 第二个参数是方法addWorker() 中用来判断max 
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }




    /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful

     大致翻译：
     当小于core 或 maximum 时，新的worker 被添加到线程池中，workerCount变量也会相应的改变。
     新的worker被创建并且运行firstTask作为worker的第一个任务，如果这个addWorker() 方法返回false
     说明线程池已经停止或者结束。或者当 threadFactory创建失败时，也会返回false.
     参数 boolean core 用来判断当前worker数量跟 corePoolSize 还是 maximumPoolSize 比较
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            /*
	        判断线程池状态 ，队列状态
	        */
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            for (;;) {
                int wc = workerCountOf(c);
                /*
	            判断线程worker数量，
	            1.大于 CAPACITY(5亿多) 
	            2.或者大于 corePoolSize 或 maximumPoolSize (取决于core的值)
	            返回false
	            addWorker() 失败，可能进workQueue(workQueue.offer())或者可能接着 addWorker()
	            */
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                /*
                更新workerCount数量，跳出循环
                */
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
        	//创建worker对象
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    //判断线程池状态
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //添加到worker列表中
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                	//执行任务
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }


```



#### 总结流程
```
1.实行任务 execute(Runnable command)
2.判断小于corePoolSize 
    1.小于 ->直接创建线程并执行 addWorker(command , true); - worker运行完直接挂起，阻塞调用workerQueue.poll()方法，等待任务
    2.大于 -> workerQueue.offer()方法(boolean返回值，当queue满了，插入不成功)，进入等待队列，之后判断线程池状态，还判断了worker 数量，创建新线程或者直接删除队列元素返回
    - 当queue插入不成功，调用addWorker() 判断maxPoolSize 如果小于max 直接new worker创建
    

```

