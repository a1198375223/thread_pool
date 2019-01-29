<center>对线程池的简单封装</center>  

<pre><code>public class ThreadPool {

    private static ExecutorService              sPool           = null; // 维护总的线程池
    private static ExecutorService              sIOPool         = null; // 维护io线程池

    private static Handler                      sUiHandler      = null; // 处理在ui线程的任务
    private static HandlerThread                sHandlerThread  = null; // 维护内部worker线程
    private static Handler                      sWorkerHandler  = null; // 处理非ui线程的任务

    private static ScheduledThreadPoolExecutor  scheduledPool = new ScheduledThreadPoolExecutor(1); // 处理周期性任务


    /**
     * 处理当线程池不够用的情况
     */
    private static RejectedExecutionHandler sRejectedHandler = new RejectedExecutionHandler() {
        @Override
        public void rejectedExecution(final Runnable r, final ThreadPoolExecutor executor) {
            ThreadPool.runOnWorker(new Runnable() {
                @Override
                public void run() {
                    try {
                        // 尝试将任务重新添加到队列
                        executor.getQueue().put(r);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    };


    /**
     * 对各种线程进行初始化, 可以在application中进行初始化
     */
    public static void start() {
        // 确保要在ui线程进行初始化操作
        if (!isUiThread()) {
            return;
        }

        ThreadFactory factory = new ThreadFactory() {
            int count = 0;
            @Override
            public Thread newThread(Runnable r) {
                count++;

                Thread thread = new Thread(r, "generic-pool-" + count);
                thread.setDaemon(false);
                thread.setPriority((Thread.NORM_PRIORITY + Thread.MIN_PRIORITY) / 2);
                return thread;
            }
        };

        // 初始化线程池
        int cpu_core = Runtime.getRuntime().availableProcessors(); // 获取进程的数量
        int maxThreads = cpu_core * 32; // 每个进程默认处理32个线程
        sPool = new ThreadPoolExecutor(cpu_core, maxThreads, 15L, TimeUnit.SECONDS,
                new LinkedBlockingDeque<Runnable>(maxThreads), factory, sRejectedHandler);


        sIOPool = new ThreadPoolExecutor(5, 10, 15L, TimeUnit.SECONDS,
                new LinkedBlockingDeque<Runnable>(maxThreads), factory, sRejectedHandler);


        sUiHandler = new Handler(Looper.getMainLooper());

        sHandlerThread = new HandlerThread("internal-work");
        sHandlerThread.setPriority(Thread.NORM_PRIORITY - 1);
        sHandlerThread.start();

        sWorkerHandler = new Handler(sHandlerThread.getLooper());
    }


    /**
     * 关闭线程
     */
    public static void stop(){
        sPool.shutdown();
        sIOPool.shutdown();
        sHandlerThread.quit();
    }


    /**
     * 让任务到ui线程去执行
     * @param runnable 待执行的任务
     */
    public static void runOnUi(Runnable runnable) {
        sUiHandler.post(runnable);
    }

    /**
     * 让任务延迟制定时间到ui线程去执行
     * @param runnable 待之执行任务
     * @param delay 延迟的时间
     */
    public static void runOnUiDelay(Runnable runnable, long delay) {
        sUiHandler.postDelayed(runnable, delay);
    }

    /**
     * 让任务到新的线程（非ui线程）中执行
     * @param runnable 待执行的任务
     */
    public static void runOnWorker(Runnable runnable) {
        sWorkerHandler.post(runnable);
    }


    /**
     * 让任务延迟指定时间后在非ui线程中执行
     * @param runnable 待执行的任务
     * @param delay 延迟的时间
     */
    public static void runOnWorkerDelay(Runnable runnable, long delay) {
        sWorkerHandler.postDelayed(runnable, delay);
    }

    /**
     * 提供方法来返回workerHandler的looper
     */
    public static Looper getWorkerLooper() {
        return sWorkerHandler.getLooper();
    }


    /**
     * 提供方法来返回WorkerHandler
     */
    public static Handler getWorkerHandler() {
        return sWorkerHandler;
    }


    /**
     * 提供方法来返回UiHandler
     */
    public static Handler getUiHandler() {
        return sUiHandler;
    }


    /**
     * 判断当前线程是否是ui线程
     * @return true:是  false:不是
     */
    private static boolean isUiThread() {
        Looper myLooper = Looper.myLooper();
        Looper mainLooper = Looper.getMainLooper();
        return mainLooper.equals(myLooper);
    }


    /**
     * 在普通线程池中执行runnable
     * @param runnable 待处理任务
     * @return 返回结果方便处理异常
     */
    public static Future<?> runOnPool(Runnable runnable) {
        if (null != sPool) {
            return sPool.submit(runnable);
        }
        return null;
    }


    /**
     * 在io线程池中执行runnable
     * @param runnable 待处理任务
     * @return 返回结果方便处理异常
     */
    public static Future<?> runOnIOPool(Runnable runnable) {
        if (null != sIOPool) {
            sIOPool.submit(runnable);
        }
        return null;
    }


    /**
     * 在io线程池处理callable任务
     * @param callable 待处理的任务
     * @return 返回结果方便处理异常
     */
    public static Future<?> callableRunOnIO(Callable<?> callable) {
        if (null != sIOPool) {
            sIOPool.submit(callable);
        }
        return null;
    }

    /**
     * 在延迟initialDelay后开始执行第一个任务, 之后每隔period时间间隔执行一次
     * @param runnable 待处理的任务
     * @param initialDelay 初始延迟时间
     * @param period 每隔period的时间执行一次
     * @return 返回结果方便处理异常
     */
    public static ScheduledFuture<?> scheduleAtFixedTime(Runnable runnable, long initialDelay, long period) {
        return scheduledPool.scheduleAtFixedRate(runnable, initialDelay, period, TimeUnit.SECONDS);
    }

    /**
     * 在延迟initialDelay后开始执行第一个任务, 在每个任务结束后延迟period时间重新执行一次
     * @param runnable 待处理的任务
     * @param initialDelay 初始延迟时间
     * @param period 每隔period的时间执行一次
     * @return 返回结果方便处理异常
     */
    public static ScheduledFuture<?> scheduleWithFixedTime(Runnable runnable, long initialDelay, long period) {
        return scheduledPool.scheduleWithFixedDelay(runnable, initialDelay, period, TimeUnit.SECONDS);
    }


    /**
     * 在延迟delay之后开始执行任务
     * @param runnable 待执行的任务
     * @param delay 延迟的时间
     * @return 返回结果方便处理异常
     */
    public static ScheduledFuture<?> schedule(Runnable runnable, long delay) {
        return scheduledPool.schedule(runnable, delay, TimeUnit.SECONDS);
    }
}</code></pre>
