# 线程池 ThreadPoolExecutor

其实之前就看过、用过线程池，也看过 AsyncTask 源码，有的时候长时间不用又记忆模糊，这里趁着最近重新看过，做下记录。

### 构造函数
<code>ThreadPoolExecutor(int corePoolSize,  maxPoolSize, keepAliveTime, unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory)</code>

corePoolSize:   核心线程数
maxPoolSize:   最大线程数
keepAliveTime:  非核心线程最多空闲的时间，超过这个时间线程就销毁
unit:   上一个参数（时间）的单位
workQueue:  任务队列
threadFactory: 线程工厂

主要看 <code>corePoolSize、maxPoolSize、workQueue</code> 三个参数。
当往线程池里提交新任务时，先判定线程池中核心线程的数量：
1、核心线程数量 < coreSize, 直接起新线程来执行新的任务；
2、核心线程数量 == coreSize，再看 workQueue 是否装满：
   a) 如果队列未满，新任务直接进入到 workQueue；
   b) 如果 workQueue 满了，判断线程池中线程数量：如果线程数量 < maxPoolSize，起新线程执行新任务；如果线程数量 == maxPoolSize, 则 Reject 异常


shutDown()：不再接受新的任务，并且会中断所有等待执行的任务。

     /** <p>This method does not wait for previously submitted tasks to
     * complete execution.  Use {@link #awaitTermination awaitTermination}
     * to do that.
     */
	public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
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


shutDownNow(): 不再接受新的任务，并且会中断线程池里的所有任务（包含正在执行的）

     /** <p>This method does not wait for actively executing tasks to
     * terminate.  Use {@link #awaitTermination awaitTermination} to
     * do that.
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }

     /**
     * Interrupts all threads, even if active. Ignores SecurityExceptions
     * (in which case some threads may remain uninterrupted).
     */
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                try {
                    w.thread.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        } finally {
            mainLock.unlock();
        }
    }


### Executors 四种线程池

1、线程数量固定，所有线程都是核心线程；任务队列是无界队列（unbounded queue），核心线程满了后其他任务都进入队列等待

    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

2、只有一条线程的线程池；任务队列为无界队列

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

3、没有核心线程，如果线程池里没有可用线程，直接起新线程执行提交的任务；适合处理耗时比较小的任务较多的情况

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

4、能执行延时任务或周期性任务的线程池

    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }



### AsyncTask

最后再来看看 AsyncTask，它内部使用了两个 Executor：SERIAL_EXECUTOR（串行） 和 THREAD_POOL_EXECUTOR（并行）。默认情况下 sDefaultExecutor 为前者，这种情况下实际是单线程在执行，如果需要任务并行，execute 时指定用后一个 Executor。再看一下线程池的几个核心参数：

    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);

最大线程数和队列的容量都不大，所以对于高并发的情况，AsyncTask 并不是一个合适的选择。
另外，因为非静态内部类持有外部类的引用，我们在 activity 里使用 AsyncTask 时，容易引起内存泄露问题，所以 AsyncTask 不适合用来处理耗时比较长的任务。
