
## 问题的发现
### 背景介绍
直接通过`Thread`或者实现`Runnable`接口的方式来创建线程都有一个显著的缺点：不能获取返回值。所以在`JDK1.5`中提供了`Callable#call()`方法来提供返回值，`Future#get()`方法来获取返回值，这样通过`Callabe`和`Future`相互配合的方式就解决了`Thread`和`Runnable`无法获取返回值的问题。但是这样只能解决一些简单场景，如果遇到比较复杂的业务场景，如希望多个线程的返回结果可以进行一定的编排，如结果转换：将上个任务的结果作为下个任务的输入重新计算产生新的结果这样的场景就力不从心了。所以J`DK8`又提供了`CompletableFuture`类来扩展和增强`Future`，同时支持任务的编排能力，如组织任务的运行顺序、规则和方式。

如果想要进一步了解`CompletableFuture`，可以在附录查看。
### 问题的介绍
在`KonaFiber`中使用`CompletableFuture`的`get()`方法时，会发现`KonaFiber8`和`KonaFiber11`的开销并不相同，`Kona8`的开销往往会大一些。本次报告主要描述了我对这种开销差异的分析与思考。

## 测试代码介绍
本次测试代码来源于[TencentKona-8/demo/fiber/mysql_sync_stress_demo at KonaFiber · Tencent/TencentKona-8](https://github.com/Tencent/TencentKona-8/tree/KonaFiber/demo/fiber/mysql_sync_stress_demo)以下为我对该代码的理解和介绍

### 整体分析
`SyncDatabaseDemo`的代码结构如下图所示。
![|500](SyncDatabaseDemo分析/images/SyncDatabaseDemo_Structure.png)

#### main()方法分析
`main()`的主要工作为：根据参数初始化环境和选择何种数据库查询方式(异步or同步)

- 根据接收参数，设置`threadCount`, `requestCount`, `testOption`的值
- `statsInterval = requestCount / 10; `： 将统计间隔`statusInterval`设定为`requestCount`的十分之一
- `initExecutor();`：初始化线程池
- `ConnectionPool.initConnectionPool();` : 初始化数据库连接池
- `if (testOption == useAsync) {...} `： 根据`testOption`的值选择异步查询`testAsyncQuery();`还是同步查询`testSyncQuery();`
- `ConnectionPool.closeConnection();` ： 关闭连接池
设定线程数=`1000`，请求数=`10000`，`testOption() = 0`
```java
public static void main(String[] args) throws Exception {
       threadCount = Integer.parseInt(args[0]);  // 1000
        requestCount = Integer.parseInt(args[1]);  // 100000
        testOption = Integer.parseInt(args[2]);  // 0
        
        statsInterval = requestCount / 10;

        initExecutor();

        ConnectionPool.initConnectionPool();
        if (testOption == useAsync) {
            testAsyncQuery();
        } else {
            testSyncQuery();
        }
        ConnectionPool.closeConnection();
    }
```
### 逐个破解
#### Fields
```java
private static ExecutorService db_executor;  
private static ExecutorService e;  
  
private static int threadCount;  //  线程数量
private static int requestCount;   // 请求梳理
private static int testOption;  // 测试选项，指定测试环境。环境为以下4种：useFiber、useThreadDirect、useThreadAndThreadPool、useAsync
private static int statsInterval;   // 统计间隔



// 4种环境
private static final int useFiber = 0;  // 使用Fiber
private static final int useThreadDirect = 1;  // 直接使用线程
private static final int useThreadAndThreadPool = 2;  // 使用线程池，使用线程
private static final int useAsync = 3;  // 异步执行
```

#### execQuery(String): String
`execQuery(String sql)`的工作为：执行`sql`并将结果以`String`类型返回
具体内容为：
- 从数据库连接池获取连接`node`
- 使用该连接`node`执行·并返回结果`rs`
- 将结果`rs`整理为`String`类型
- 释放连接并返回结果。
```java
public static String execQuery(String sql) throws InterruptedException, ExecutionException {
        String queryResult = "";
        try {
            ConnectionNode node;
            do {
                node = ConnectionPool.getConnection();
            } while (node == null);
            ResultSet rs = node.stm.executeQuery(sql);



            while (rs.next()) {
                int id = rs.getInt("id");
                String hello = rs.getString("hello");
                String response = rs.getString("response");

                queryResult += "id: " + id + " hello:" + hello + " response: "+ response + "\n";
            }

            rs.close();
            ConnectionPool.releaseConnection(node);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return queryResult;
    }
```

#### submitQuery(String): String
`submitQuery(String sql)`的工作为：使用`CompleteFuture`将`sql`异步提交到到`execQuery(String sql)`方法中，并将结果返回。
具体内容为：
- 创建`CompleteFuture`的实例`future`
- 创建一个`Runnable`任务r：用`future`讲`sql`查询任务提交到`future`中执行，并将结果存入到`future`中
- 使用`db_executor`中的线程执行任务`r`
- `return future.get();` 在`get()`处阻塞，直到`future`中有结果。
```java
public static String submitQuery(String sql) throws InterruptedException, ExecutionException {
        CompletableFuture<String> future = new CompletableFuture<>();



        Runnable r = new Runnable() {
            @Override
            public void run() {
                try {
                    future.complete(execQuery(sql));  // *** 见以下讲解
                } catch (Exception e) {

                }
            }
        };
        db_executor.execute(r);

        return future.get();
    }
```
##### CompleteFuture的complete()和get()方法解析
首先是`future.complete(execQuery(sql));`的作用：将`execQuery(sql)`的结果最终通过`cas`操作将结果保存在`future`中
`complete()`的调用过程为：`CompletableFuture#complete() 调用 CompletableFuture#completeValue() 调用 Unsafe#compareAndSwapObject()
具体代码如下所示：`
```java
java.util.concurrent.CompletableFuture#complete

    public boolean complete(T value) {
        boolean triggered = completeValue(value);  // ***
        postComplete();
        return triggered;
    }
```

```java
java.util.concurrent.CompletableFuture#completeValue

    final boolean completeValue(T t) {
        return UNSAFE.compareAndSwapObject(this, RESULT, null,
                                           (t == null) ? NIL : t);  // ***
    }

sun.misc.Unsafe#compareAndSwapObject
    /**
     * Atomically update Java variable to <tt>x</tt> if it is currently
     * holding <tt>expected</tt>.
     * @return <tt>true</tt> if successful
     */
    public final native boolean compareAndSwapObject(Object o, long offset,
                                                     Object expected,
                                                     Object x);
```
#### testAsyncQuery(): void
`testAsyncQuery()`的任务为：测试异步查询时的性能。打印统计信息`"interval <当前已处理请求数> throughput <吞吐量>"`



具体内容为：
- `startSignal`和`doneSigna`l为`CountDownLatch`锁，用于控制输出顺序。实现了当所有`request`都被处理完后，再同步输出`finish <请求数> time <用时> throughput <吞吐量>`
	- `startSignal.await(); ` // 阻塞等待，直至`startSignal`的值变为`0`时
	- `startSignal.countDown();`  `startSignal`的值减`1`
- `count` 和 `statsTimes`为原子类型变量，通过`cas`操作修改值。
	- `count.addAndGet(1);`  `count + 1`并将结果返回
	- `statsTimes.getAndSet(time);` 将原值返回，并将`time`设定为`statsTimes`的新值
	- `statsTimes.set(before);` 将`statsTimes`的值设定为`before`
- `for (int i = 0; i < requestCount; i++) { ...} `依次处理所有请求
	- `cf = CompletableFuture.supplyAsync(...,e) `开启异步任务执行`sql`的查询，并将查询结果result结果返回；
	- `cf.thenAccept(result ->{},e)` 将supplyAsync返回的结果`result`消费掉。这里是用来每处理完一份时，打印`"interval <当前已处理请求数> throughput <吞吐量>"`。
	- 传递的参数`e`是自定义的线程池，用该线程池来执行异步任务。
- `for()`后的代码用于打印`finish <请求数> time <用时> throughput <吞吐量>`以及线程池`e`的收尾工作。
```java
    public static void testAsyncQuery() throws Exception {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(requestCount);
        AtomicLong count = new AtomicLong();
        AtomicLong statsTimes = new AtomicLong();

        for (int i = 0; i < requestCount; i++) {
            // Execute async operation
            CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
                String result = null;
                try {
                    startSignal.await();
                    result = execQuery("select * from hello");
                } catch (Exception e) {
                }

                return result;
            }, e);

            // async operation is done, update statistics
            cf.thenAccept(result -> {
                long val = count.addAndGet(1);
                if ((val % statsInterval) == 0) {
                    long time = System.currentTimeMillis();
                    long prev = statsTimes.getAndSet(time);
                    System.out.println("interval " + val + " throughput " + statsInterval/((time - prev)/1000.0));
                }
                doneSignal.countDown();
            });
        }

        long before = System.currentTimeMillis();
        statsTimes.set(before);
        startSignal.countDown();
        doneSignal.await();

        long after = System.currentTimeMillis();
        long duration = (after - before);
        System.out.println("finish " + count.get() + " time " + duration + "ms throughput " + (count.get()/(duration/1000.0)));

        e.shutdown();
        e.awaitTermination(Long.MAX_VALUE, TimeUnit.NANOSECONDS);
    }
```
#### testSyncQuery(): void
`testSyncQuery()`的工作为：测试同步步查询时的性能。打印统计信息`"interval <当前已处理请求数> throughput <吞吐量>"`。
具体内容(和`testAsyncQuery()`的区别)：
- 定义Runnable任务r：如果`testOption`为`userFiber` 或 `userThreadAndThreadPool`，则通过`submitQuery(sql)`来执行`sql`；	否则直接使用当前`Thread`来处理线程
	- `submitQuery(sql)`中也是通过`CompleteFuture`异步提交`sql`到`execQuery(String sql)`方法
- 使用线程池`e`来执行任务`r`
```java
public static void testSyncQuery() throws Exception {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(requestCount);
        AtomicLong count = new AtomicLong();
        AtomicLong statsTimes = new AtomicLong();

        Runnable r = new Runnable() {
            @Override
            public void run() {
                try {
                    startSignal.await();
                    String sql = "select * from hello";
                    String result;
                    if (testOption == useFiber || testOption == useThreadAndThreadPool) {
                        // submit query to an independent thread pool;
                        result = submitQuery(sql);
                    } else {
                        // execute query direct(use current thread)
                        result = execQuery(sql);
                    }
                    //System.out.println("execute sql result is " + result);

                    long val = count.addAndGet(1);
                    if ((val % statsInterval) == 0) {
                        long time = System.currentTimeMillis();
                        long prev = statsTimes.getAndSet(time);
                        System.out.println("interval " + val + " throughput " + statsInterval/((time - prev)/1000.0));
                    }
                    doneSignal.countDown();
                } catch (Exception e) {

                }
            }
        };

        for (int i = 0; i < requestCount; i++) {
            e.execute(r);
        }

        long before = System.currentTimeMillis();
        statsTimes.set(before);
        startSignal.countDown();  // 和startSignal.await();配合让所有任务同时开始执行。
        doneSignal.await();

        long after = System.currentTimeMillis();
        long duration = (after - before);
        System.out.println("finish " + count.get() + " time " + duration + "ms throughput " + (count.get()/(duration/1000.0)));

        e.shutdown();
        e.awaitTermination(Long.MAX_VALUE, TimeUnit.NANOSECONDS);

        if (testOption == useFiber || testOption == useThreadAndThreadPool) {
            db_executor.shutdown();
        }
    }
```

#### initExecutor(): void
`initExecutor()`的工作为：根据 `testOption` 的值来初始化线程池 `e` 和 `db_executor`，该方法可以初始化不同类型的线程池。
具体内容：
1. `testOption == useFiber` 0
	- 创建虚拟线程`factory`
	- 线程池`e`设置为`FixedThreadPool(threadCount, factory);`
	- 线程池 `db_executor`设置为`FixedThreadPool(Runtime.getRuntime().availableProcessors() * 2);`
2. `testOption == useThreadDirect`1
	- 创建平台线程`factory`
	-  线程池`e`设置为`FixedThreadPool(threadCount, factory);`
	-  `db_executor` 为`null`
3. `testOption ==  useThreadAndThreadPool` 2
	- 创建平台线程`factory`
	- 线程池`e`设置为`FixedThreadPool(threadCount, factory);`
	- 线程池 `db_executor`设置为`FixedThreadPool(Runtime.getRuntime().availableProcessors() * 2);`,即处理器核心数的`2`倍
4. `testOption == useAsync` 3
	- 创建平台线程`factory`
	- `threadCount`设置为处理器核心数的`2`倍
	- 线程池`e`设置为`WorkStealingPool(threadCount);` ，使用`ForkJoinPool`线程池
	- `db_executor` 为`null`


|             | userFiber 0                             | useThreadDirect 1                       | useThreadAndThreadPool 2                | useAsync 3                                                   |
| :---------: | :-------------------------------------- | :-------------------------------------- | :-------------------------------------- | :----------------------------------------------------------- |
|   factory   | `Thread.*ofVirtual*().factory()`        | `Thread.*ofPlatform*().factory()`       | `Thread.*ofPlatform*().factory()`       | `Thread.*ofPlatform*().factory()`                            |
|      e      | `FixedThreadPool(threadCount, factory)` | `FixedThreadPool(threadCount, factory)` | `FixedThreadPool(threadCount, factory)` | `WorkStealingPool(threadCount)` <br />`// threadCount = availableCPU * 2` |
| db_executor | `FixedThreadPool(availableCPU * 2)`     | `null`                                  | `FixedThreadPool(availableCPU * 2)`     | `null`                                                       |


```java
    public static void initExecutor() {
        ThreadFactory factory;
        if (testOption == useFiber) {
            factory = Thread.ofVirtual().factory();
        } else {
            factory = Thread.ofPlatform().factory();
        }


        if (testOption == useAsync) {
            // thread count is equal to available processors when useAsync
            threadCount = Runtime.getRuntime().availableProcessors();
            e = Executors.newWorkStealingPool(threadCount);
        } else {
            e = Executors.newFixedThreadPool(threadCount, factory);
        }

        if (testOption == useFiber || testOption == useThreadAndThreadPool) {
            // an independent thread pool which has 16 threads
            db_executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors() * 2);
        }
    }
```


### 四种场景执行步骤
![](images/SyncDatabaseDemo%20(2).png)

##### 四种场景的性能理论分析
数据库查询是一个典型的IO阻塞型任务，这里首先阐述`e`和`db_executor`分别的作用：
- `e`：每个`request`会从`e`中获得一个线程来处理该请求。
- `db_executor`：执行`sql`查询操作时不使用当前线程查询，而是从`db_executor`获取一个线程去执行查询。
这里给出这四种场景的区别：
1. `useThreadDirect ` 通常会有线程管理问题，如大量线程创建和销毁的开销。在本场景中，每个request会从e中获得一个线程来执行。这里所谓的直接使用线程是指没有使用`db_executo`,当从e中获得的线程在执行查询任务时会阻塞在这里，无法完成其他事情。
2. `useThreadAndThreadPool`使用线程池`db_executor(FixedThreadPool)`执行`execQuery(sql)`查询任务，避免了`useThreadDirect`造成的额外开销。
3. `userAsync` 使用线程池`e(ForkJoinPool)`和`CompleteableFuture`异步执行`execQuery(sql)`查询任务。
4. `useFiber`是使用`VirtulaThread`（用户级线程）来执行任务，，当遇到阻塞时会`yield`出`platformThread`，让其它不阻塞的`virthalThread`装载到`platformThread`上执行。同时默认的线程池也是`ForkJoinPool`，所以具备该线程池的优点。
**根据以上介绍，吞吐量理论上的大小顺序应为：**
 `userAsync` > `userFiber` >`useThreadAndThreadPool` > `useThreadDirect`

##### 四种场景的性能实际表现
`threadCount`设定为固定变量：`1000`。

| requestCount | userFiber ：0      | useThreadDirect： 1 | useThreadAndThreadPool：2 | userAsync ：3      |
| ------------ | ------------------ | ------------------- | ------------------------- | ------------------ |
| 100000       | 55897.149245388486 | 47732.6968973747    | 52328.623757195186        | 56915.196357427434 |

1 finish 100000 time 29892ms throughput 3345.376689415228
2 finish 100000 time 19702ms throughput 5075.626839914729
### 火焰图分析`SyncDatabaseDemo`在`useFiber`场景下的性能
#### 火焰图介绍
火焰图（Flame Graph）是一种可视化工具，用于分析和可视化软件应用程序的性能和调用堆栈信息，可以深入了解应用程序在运行时的性能状况。
火焰图是二维的结构，分为X轴和Y轴：
- X轴：采样次数。一个函数在X轴越长👉采样的次数越多👉该函数执行时间越长。
- Y轴：调用关系：自底向上调用
具体的使用方式可以参考这篇博客：[超好用的自带火焰图的 Java 性能分析工具 Async-profiler 了解一下-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1554194)
#### 分析结果
以下为`KonaFiber8`和`11`下`CompletFuture.get()`的采样次数，可以看到前者占比`5.84%`，后者占比`0.73%`。这说明`get()`在`KonaFiber8`下的执行时间会更长
**KonaFiber8**
![](SyncDatabaseDemo分析/images/SyncDatabaseDemo分析-1.png)



**KonaFiber11**
![](SyncDatabaseDemo分析/images/SyncDatabaseDemo分析.png)


## 问题分析与解决

### KonaFiber 8与11中get()实现对比
每部分代码讲述将会按照
1. 简述功能
2. 代码及注释分析
3. 代码要点梳理
的顺序进行介绍。

#### KonaFiber 8与11中get()实现对比图
如下图所示，KonaFiber8和11的主要差异在非蓝色部分：
- 8采用了自旋等待的方式，通过不释放忙等来达到不释放cpu的目的。
- 11利用了`ForkJoinPool`的`work-steal`算法，用**异步执行其它任务**来代替**无意义的忙等**，更加充分的利用了CPU资源。
![](images/kona8的waitingGet()调用过程.png)



####  Signaller
`Kona8`和`Kona11`中的`waitingGet()`都使用到了`Signaller`内部类，所以先进行简要介绍：
`Signaller`内部类是一个信号，起到辅助唤醒和阻塞线程的作用。
```java
    /**
     * Completion for recording and releasing a waiting thread.  This
     * class implements ManagedBlocker to avoid starvation when
     * blocking actions pile up in ForkJoinPools.
     */
    @SuppressWarnings("serial")
    static final class Signaller extends Completion
        implements ForkJoinPool.ManagedBlocker {  // 注意Signaller实现了ForkJoinPool.ManagedBlocker方法
        long nanos;                    // wait time if timed
        final long deadline;           // non-zero if timed
        volatile int interruptControl; // > 0: interruptible, < 0: interrupted(等待过程中发生了中断) , 0：can't interrupt
        volatile Thread thread;


        Signaller(boolean interruptible, long nanos, long deadline) {
            this.thread = Thread.currentThread();
            this.interruptControl = interruptible ? 1 : 0;
            this.nanos = nanos;
            this.deadline = deadline;
        }
        // 唤醒阻塞线程
        final CompletableFuture<?> tryFire(int ignore) {
            Thread w; // no need to atomically claim
            if ((w = thread) != null) { // ？ 
                thread = null;
                LockSupport.unpark(w);
            }
            return null;
        }
        // 检查waiting thread是否可以release
        public boolean isReleasable() {
            if (thread == null)  // thread为null，表示waiting thread已经被唤醒
                return true;
            if (Thread.interrupted()) { // thread处于中断等待状态：被中断了
                int i = interruptControl; //根据interruptControl判断，> 0: interruptible, < 0: interrupted , 0：can't interrupt
                interruptControl = -1;
                if (i > 0)
                    return true;  // 线程被中断 且 线程中断标志位为true
            }
            if (deadline != 0L &&
                (nanos <= 0L || (nanos = deadline - System.nanoTime()) <= 0L)) {  // thread处于超时等待状态：设置了ddl && 等待时间等于或超过ddl
                thread = null;
                return true;
            }
            return false;
        }
        
        // 将线程阻塞，阻塞成功就返回true
        public boolean block() {
            if (isReleasable())  //thread处于中断等待或超时等待状态时，返回true说明thread正在被阻塞
                return true;  
            else if (deadline == 0L)  // 没有设置ddl
                LockSupport.park(this);
            else if (nanos > 0L)  // waiting time > 0
                LockSupport.parkNanos(this, nanos);
            return isReleasable();  
        }
        // check 线程是否存活
        final boolean isLive() { return thread != null; }
    }

```
要点梳理：
- `tryFire(int ignore)`：唤醒等待线程
- `isReleasable()`：线程处于唤醒、中断等待、超时等待状态时返回true，说明可被释放
- `block()`：阻塞线程，阻塞成功返回true。
	- 和`isReleaseable()`配合实现中断等待和超时等待功能；线程没有设置ddl或等待时间大于0时将线程阻塞。
####  waitingGet()的Kona8的实现
##### waitingGet()
`waitingGet()` 是等待获取线程/虚拟线程阻塞后的结果。如果线程/虚拟线程被中断就返回null。
```java
    private Object waitingGet(boolean interruptible) {
        Signaller q = null;
        boolean queued = false;  // 判断Signaller是否已经入队
        int spins = -1;  // 计数器，自旋等待次数。
        Object r;  // 存储result
        while ((r = result) == null) {  // 获取到结果时退出循环
		        //  自旋等待
            if (spins < 0)
                spins = SPINS;  // 将spins设置为256
            else if (spins > 0) {
                if (ThreadLocalRandom.nextSecondarySeed() >= 0)  // 随机判断是否减少spins的值
                    --spins;
            }
            
            // 当自旋次数用尽时，创建新的Signaller对象
            else if (q == null)  // ：spins == 0 
                q = new Signaller(interruptible, 0L, 0L);
            else if (!queued)  // q 没有入队
                queued = tryPushStack(q);  // 将Signaller对象入队，入队的目的是为了在需要唤醒等待线程时能够找到对应的 `Signaller` 对象，从而进行唤醒操作。
                
            //  中断控制
            else if (interruptible && q.interruptControl < 0) {  // 可中断 && 已经被中断
                q.thread = null;
                cleanStack();
                return null;
            }
            
            //  阻塞等待：通过ForkJoinPool.managedBlock(q)实现
            else if (q.thread != null && result == null) {  // q还在等待 && 没获取到结果
                try {
                    ForkJoinPool.managedBlock(q);  // 阻塞q
                } catch (InterruptedException ie) {  // 发生了中断
                    q.interruptControl = -1;
                }
            }
        }
        
        // 清理q
        if (q != null) {
            q.thread = null;
            if (q.interruptControl < 0) {
                if (interruptible)
                    r = null; // report interruption
                else
                    Thread.currentThread().interrupt();
            }
        }
        postComplete();
        return r;
    }
```
要点梳理：
主要是通过`while ((r = result) == null) { ... } `达到一直尝试获取result的结果，在`{...}`中采取了多种等待策略来优化性能
- 自旋等待机制(原地等待)：自旋锁的实现原理就是自旋等待。这种机制适合于在cpu空转的开销要小于线程切换的开销这类场景。[自旋锁](https://zhuanlan.zhihu.com/p/263762343)
	- 它和核心逻辑是假设==线程切换的成本  > 原地等待的成本==
- 中断控制
- 通过`ForkJoinPool.managedBlock(q)`实现阻塞等待
##### ForkJoinPool.managedBlock(q)
`managedBlock`的主要作用就是管理阻塞
```java
public static void managedBlock(ManagedBlocker blocker)
        throws InterruptedException {
        ForkJoinPool p;
        ForkJoinWorkerThread wt;
        Thread t = Thread.currentThread();
        // 判断当前线程是否是ForkJoinPool中线程的实例
        if ((t instanceof ForkJoinWorkerThread) &&
            (p = (wt = (ForkJoinWorkerThread)t).pool) != null) {
            WorkQueue w = wt.workQueue;
            while (!blocker.isReleasable()) {
                if (p.tryCompensate(w)) {  // 判断当前线程是否可以阻塞
                    try {
                        do {} while (!blocker.isReleasable() &&  // 如果可以阻塞就通过while()的方式来阻塞。
                                     !blocker.block());
                    } finally {
                        U.getAndAddLong(p, CTL, AC_UNIT);
                    }
                    break;
                }
            }
        }
        else {  // 阻塞线程
            do {} while (!blocker.isReleasable() &&
                         !blocker.block());
        }
    }
```

#### waitingGet()的Kona11实现
##### waitingGet
```java
    private Object waitingGet(boolean interruptible) {
        Signaller q = null;
        boolean queued = false;
        Object r;
        while ((r = result) == null) {
		        //  初始化Signaller对象，并且当前线程为ForkJoinThread时，调用ForkJoinPool.helpAsyncBlocker()
            if (q == null) {
                q = new Signaller(interruptible, 0L, 0L);
                if (Thread.currentThread() instanceof ForkJoinWorkerThread)
                    ForkJoinPool.helpAsyncBlocker(defaultExecutor(), q);
            }
            else if (!queued)
                queued = tryPushStack(q);
            //  阻塞等待 + 中断控制：通过ForkJoinPool.managedBlock(q)实现
            else {
                try {
                    ForkJoinPool.managedBlock(q);
                } catch (InterruptedException ie) { // currently cannot happen
                    q.interrupted = true;
                }
                if (q.interrupted && interruptible)
                    break;
            }
        }
        // 清理q
        if (q != null && queued) {  // q对象不为空且已经入队了
            q.thread = null;
            if (!interruptible && q.interrupted)
                Thread.currentThread().interrupt();
            if (r == null)
                cleanStack();
        }
        if (r != null || (r = result) != null) //  获取到result了
            postComplete();
        return r;
    }


```

##### helpAsyncBlocker
`helpAsyncBlocker` 的主要工作就是尝试执行异步任务，直到`blocker`被释放掉。
```java
    static void helpAsyncBlocker(Executor e, ManagedBlocker blocker) {
		    //  初始话WorkQueue w
        if (e instanceof ForkJoinPool) {
            WorkQueue w; ForkJoinWorkerThread wt; WorkQueue[] ws; int r, n;
            ForkJoinPool p = (ForkJoinPool)e;
            Thread thread = Thread.currentThread();
            if (thread instanceof ForkJoinWorkerThread &&
                (wt = (ForkJoinWorkerThread)thread).pool == p)
                w = wt.workQueue;
            else if ((r = ThreadLocalRandom.getProbe()) != 0 &&
                     (ws = p.workQueues) != null && (n = ws.length) > 0)
                w = ws[(n - 1) & r & SQMASK];
            else
                w = null;
             // 调用helpAsyncBlocker(blocker)
            if (w != null)
                w.helpAsyncBlocker(blocker);
        }
    }
```

```java
        final void helpAsyncBlocker(ManagedBlocker blocker) {
            if (blocker != null) {
                int b, k, cap; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
                while ((a = array) != null && (cap = a.length) > 0 &&  		        //blocker被释放就跳出循环
                       top - (b = base) > 0) {
                    t = (ForkJoinTask<?>)QA.getAcquire(a, k = (cap - 1) & b);
                    if (blocker.isReleasable())  //  blocker可以释放就跳出循环
                        break;
                    else if (base == b++ && t != null) {
                        if (!(t instanceof CompletableFuture.
                              AsynchronousCompletionTask))
                            break;
                        else if (QA.compareAndSet(a, k, t, null)) {
                            BASE.setOpaque(this, b);
                            t.doExec();  //  执行ForkJoinTask
                        }
                    }
                }
            }
        }
```

#### KonaFibe8 和 KonaFiber11的区别总结
`KonaFibe8`的`waitingGet()`特点为：
1. 用`spins`做一个较短时间的自旋
2. `managedBlocke()`是根据线程自己的状态来判断是否可以阻塞，如果线程的状态可以阻塞，就阻塞线程。

`KonaFiber11`的`waitingGet()`特点为：
1. 用`helpAsyncBlocker`来代替自旋等待`spins`，`spins`是只忙等，`helpAsyncBlocker`是利用到了`ForkJoinPool`的一些特性，利用这段时间从`WorkerQueue`中窃取任务来异步完成，更加充分利用了CPU资源。
2. 同`8`的`managedBlocke`
3. 简化中断控制的相关代码。

### 定位KonaFiber8的get()开销大的原因
`KonaFiber8`的`waitingGet()`有一个自旋锁逻辑，从理论上分析对于`VirtualThread`来说，自旋锁的设置可能并不合适，会加大开销。
具体原因为：`VirtualThread`的切换开销对比`PlatformThread`会小很多，而自旋锁假设的是线程开销比较大，如果阻塞的时间不长，可以通过在CPU忙等的方式来等待结果，以此避免高额的线程开销。
这就会有一个矛盾点：`VirthalThread`的开销没有那么大，自旋等待造成的开销可能比`VirtualThread`切换的开销更大。这样自旋等待就会造成额外的开销，所以可能应该删掉这段自旋等待的逻辑。


#### 加大自旋Spins对VirtualThread的影响
想证明`spins`可能会造成额外开销，就要尽可能让火焰图上和`spins`相关的部分长度变长。
	我这里采取的方案为增大并发数，即提高`requestCount`的数量级，这样会加大`VirtualThread`间竞争资源的开销，增大其等待获取某种结果的次数，以此来加大`Spins`的影响。

Notes：想要加大Spins的影响除了加大并发数以外，还可以尽可能减少SQL的查询时间，但考虑到本测试的sql已经很简单了，表里只有一张数据，所以从这个角度没有办法进行改进。但同时我也测试了如果增加查询sql的时间会怎样，相关分析可以再附录中看到。
```java
public static void main(String[] args) throws Exception {
        threadCount = 1000;
        requestCount = 10000000;  // 将requestCount 提高两个数量级
        testOption = 2;
        statsInterval = requestCount / 10;

        initExecutor();

        ConnectionPool.initConnectionPool();
        if (testOption == useAsync) {
            testAsyncQuery();
        } else {
            testSyncQuery();
        }
        ConnectionPool.closeConnection();
    }
```
从下面两张火焰图上可以看到当`requestCount`增大后，`waitingGet()`开销显著增高，从约`5.84%`增高到了`12.8%`，并且和`spins`相关的`ThreadLocalRandom.nextSecondarySeed()`代码占比显著降低，在火焰图上几乎看不到了，只有`0.05%`左右。

![](images/CompletableFuture的get()在Kona8和Kona11的差异对比-1.png)
![](images/CompletableFuture的get()在Kona8和Kona11的差异对比.png)

现在分析`ThreadLocalRandom.nextSecondarySeed()`的开销占比显著降低说明了什么？
以极端情况为例：如果`ThreadLocalRandom.nextSecondarySeed()`的开销占比约100%，说明大多数请求都在Spins自旋等待中获取到了结果，相反，若`ThreadLocalRandom.nextSecondarySeed()`的开销特别小，则说明获取结果的大多数开销都在`spins`自旋等待后的阻塞等待中获取到，这就说明了`spins`在`VirtualThread`中会造成额外的开销。

所以我在`Kona8`的源码中删除`spins`的相关代码，重新编译后，再次测试，发现`waitingGet()`的开销从`12.8%`降低到`1.80%`，可以看出开销有明显降低。
其中编译`KonaFiber`的学习过程可以再附录中看到
```java
        while ((r = result) == null) {  // 注释部分为删除的代码
/*            if (spins < 0)
                spins = SPINS;
            else if (spins > 0) {
                if (ThreadLocalRandom.nextSecondarySeed() >= 0)
                    --spins;
            }
            else*/ if (q == null)
                q = new Signaller(interruptible, 0L, 0L);
            else if (!queued)
                queued = tryPushStack(q);
            else if (interruptible && q.interruptControl < 0) {
                q.thread = null;
                cleanStack();
                return null;
            }
            else if (q.thread != null && result == null) {
                try {
                    ForkJoinPool.managedBlock(q);  // to block thread.
                } catch (InterruptedException ie) {
                    q.interruptControl = -1;
                }
            }
        }
```
![](images/CompletableFuture的get()在Kona8和Kona11的差异对比-3.png)
同时我再对比KonaFiber11在同样情况下的开销为1.97%。
![](images/CompletableFuture的get()在Kona8和Kona11的差异对比-4.png)

## 我的改进建议
直接删掉`Spins`是不妥当的，更好的改进方案应该是让原有的`PlatformThread`保留`Spins`相关的自旋等待逻辑，而让`VirtualThread`执行删掉`Spins`后的代码。
修改样例如下
```java
private Object waitingGet(boolean interruptible) {

        Signaller q = null;
        boolean queued = false;
        int spins = -1;
        Object r;
        boolean isVT = Thread.currentThread().isVirtual();  // 增加isVT判断当前是否是VirtualThread
        while ((r = result) == null) {  
                        if (!isVT) {  // 是platformThread就执行spins
                            if (spins < 0)
                                spins = SPINS;
                            else if (spins > 0) {
                                if (ThreadLocalRandom.nextSecondarySeed() >= 0)
                                    --spins;
                            }
                        }
            else if (q == null)
                q = new Signaller(interruptible, 0L, 0L);
            else if (!queued)
                queued = tryPushStack(q);
            else if (interruptible && q.interruptControl < 0) {
                q.thread = null;
                cleanStack();
                return null;
            }
            else if (q.thread != null && result == null) {
                try {
                    ForkJoinPool.managedBlock(q);  // to block thread.
                } catch (InterruptedException ie) {
                    q.interruptControl = -1;
                }
            }
        }
        if (q != null) {
            q.thread = null;
            if (q.interruptControl < 0) {
                if (interruptible)
                    r = null; // report interruption
                else
                    Thread.currentThread().interrupt();
            }
        }
        postComplete();
                            //boolean falg = spins == 0;  // todo modiff add
        return r;
    }
```

## 附录
### CompletableFuture
#### 从new Thread()讲起
以下会从`new Thread`在某些情况下无法满足需求，为解决这些问题出现了`Callable`、`Future`、`FutureTask`以及`CompletableFuture`。

##### Thread
首先是线程的创建有2种方式：
```java
// 方式一
new Thread().start();


//方式二
new Thread(new Runnable() {
		@Override
		public void run() {
				// do my work
		}
}).start();
```
问题：可以发现，这两种方式都没有返回值，如果我们想要获取线程执行完毕的结果，是无法做到的。
##### Callable
解决：此时我们可以使用`Callable`类，通过它的`call()`方法我们可以返回线程执行的结果。
```java
@FunctionalInterface
public interface Callable<V> {
		// Computes a result, or throws an exception if unable to do so.
		//Returns:computed result
    V call() throws Exception;
```
问题：现在我们可以返回线程执行的结果了，但是如果我们还需要限制`main()`线程的执行，让它等待并接收线程执行的结果。

##### Future
解决：此时我们可以使用`Future`类，通过它的`get()`方法我们可以阻塞`main()`线程直至获取到线程的执行结果。
```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();  // 判断任务十分执行完
    V get() throws InterruptedException, ExecutionException;  // 获取结果
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```
问题：可以发现`Future`是一个接口，我们不能直接使用，只能使用它的实现类`FutureTask`

##### FutureTask
解决：以下是`FutureTask`的使用样例
```java
public class FutureTaskDemo {


		public static void main(String[] args) throws ExecutionException, InterruptedException {
				Task task = new Task();
				FutureTask<Integer> futureTask = new FutureTask<>(task);
				// futureTask作为Runnable传入Thread
				new Thread(futureTask).start();
				
				System.out.println("运行结果："+futureTask.get());
		}

		static class Task implements Callable<Integer> {
				@Override
				public Integer call() throws Exception {
					 // do my work
						return result; // result 在futureTask.get()出获取
				}
		}
}
```
注意：==通过这个例子我们可以发现Callable和Future都是一种任务类，并不会作为一个线程去执行。==

问题：`Future`虽然可以解决`Thread`无法获取结果的问题，但是它在并发执行多个任务时，如果阻塞时间长的`future`先`get()`，阻塞时间短的`future`后`get()`，而我想要先获取阻塞时间短的任务结果，此时是无法满足需求的。
```java
// taskB的阻塞时间短，想要先获取taskB的结果，使用futureTask无法满足需求
        public static void main(String[] args) throws ExecutionException, InterruptedException {
            TaskA taskA = new TaskA();
            TaskB taskB = new TaskB();
            FutureTask<String > futureTaskA = new FutureTask<>(taskA);
            FutureTask<String> futureTaskB = new FutureTask<>(taskB);
            // futureTask作为Runnable传入Thread
            new Thread(futureTaskA).start();
            new Thread(futureTaskB).start();

            System.out.println(futureTaskA.get());
            System.out.println(futureTaskB.get());

        }

        static class TaskA implements Callable<String> {

            @Override
            public String  call() throws Exception {
                Thread.sleep(1000);
                return "taskA 完成";
            }
        }
        static class TaskB implements Callable<String> {

            @Override
            public String call() throws Exception {
                Thread.sleep(10);
                return "taskB 完成";  
            }
        }

输出：
taskA 完成
taskB 完成
```
除了该问题，当我们在多任务场景下，只想获取最先完成任务的线程结果，`future`也无法做到。
解决：因此有了`CompletableFuture`

#### CompletableFuture
`CompletableFuture`增强了`Future`的功能，可以对`Future`获取的结果进行复杂操作：结果处理、结果转换、结果消费等功能。同时也控制多任务执行的顺序，即任务编排功能。


这里先简单介绍`CompletableFuture`的创建异步任务，获取结果，结果消费三部分内容
#### 创建异步任务
```java
public static CompletableFuture<Void> runAsync(Runnable runnable)  
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)  
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)  
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```
区别：
- `runAsync()`以`Runnable`为参数，无返回值
- `supplyAsync()`以`Supplier`为参数，有返回值
	- `Supplier`是一个函数式接口，它仅有一个get()方法，返回一个T。
- `Executor executor`
	- 若传递了`executor`，则用`executor`中的线程执行任务
	- 否则默认使用`ForkJoinPool.commonPool()`执行任务。该线程池默认使用CPU核心数


#### 获取结果
阻塞当前线程，直至可以获取到异步任务的执行结果，比如supplyAsync()的返回值
``` java
public T join()
public T get() throws InterruptedException, ExecutionException
```
区别：
- `get()`在线程被中断时会抛出异常，需要手动处理一下
- `join()`不会抛出异常



### KonaFiber的编译过程
#### 准备源码
克隆
 `git clone git@github.com:Tencent/TencentKona-8.git`
 
切换分支
`cd TencentKona-8`
`git checkout Fiber-8.0.13-GA`
#### 准备编译环境
下载jdk7，并将其放在`/usr/lib/jvm`下
[Java Archive Downloads - Java SE 7 --- Java 归档下载 - Java SE 7 (oracle.com)](https://www.oracle.com/java/technologies/javase/javase7-archive-downloads.html#jdk-7u55-oth-JPR)
查看环境
`bash configure --with-boot-jdk=/usr/lib/jvm/jdk1.7.0_80`
![|700](images/在win11上编译jdk-7.png)
#### 编译
`make all`


成功
![|500](images/在win11上编译jdk-6.png)

进入build目录下：
`/home/sunrise/work/java/konaJDK/TencentKona-8/build/linux-x86_64-normal-server-release/jdk/bin`
执行命令`./java -version`，会显示自己编译过的jdk版本

#### 故障排除
##### Cannot find GNU make
典型日志： 
`error: Cannot find GNU make 3.81 or newer! Please put it in the path, or add e.g. MAKE=/opt/gmake3.81/make as argument to configure.`
解决：
`sudo apt update && sudo install make`
参考链接：[openjdk 1.8 源码编译 - 掘金 (juejin.cn)](https://juejin.cn/post/6905377760265863176)


##### Boot_JDK
典型日志：
`error: The path given by --with-boot-jdk does not contain a valid Boot JDK`
或`configure: error: The path of BOOT_JDK, which resolves as "/usr", could not be imported.`
解决：指定jdk版本
`bash configure --with-boot-jdk=/usr/lib/jvm/java-16-openjdk-amd64`

查找本机jdk有哪些：
```sh
ls /usr/lib/jvm # 查找 JDK 安装路径

java -version # 查看 Java 版本

sudo update-alternatives --config java # 切换java版本
```

**下载安装jdk的利器：**
[SDK管理利器：SDKMAN！ - 掘金 (juejin.cn)](https://juejin.cn/post/7044711294984781838)
##### 找不到C编译器
典型日志：
`configure: error: C compiler cannot create executables`
解决：
` bash configure --with-boot-jdk=/usr/lib/jvm/java-16-openjdk-amd64 --build=x86_64-unknown-linux-gnu`

##### x11
典型日志：
`configure: error: Could not find all X11 headers (shape.h Xrender.h Xrandr.h XTest.h Intrinsic.h).`
解决
`sudo apt-get install libx11-dev libxext-dev libxrender-dev libxrandr-dev libxtst-dev libxt-dev`
参考链接：[x11](https://github.com/openjdk/jdk/blob/master/doc/building.md#x11)

##### cups
典型日志：
`configure: error: Could not find cups!`
解决：
` sudo apt install     libcups2-dev`
[cups](https://github.com/openjdk/jdk/blob/master/doc/building.md#cups)

##### fontconfig
典型日志：
`configure: error: Could not find fontconfig!`
解决：
`sudo apt-get install libfontconfig-dev`

##### alsa
典型日志：
`configure: error: Could not find alsa!`
解决：
`sudo apt-get install libasound2-dev`

### 增加查询sql和数据库的时间
本次测试是基于`requestCount = 100000;`
#### 修改数据库和查询语句
新建立一张`hello_bigdata`表，表结构和`hello`相同，但是记录数为`1000`条。
修改`sql`为`“select * from hello_bigdata where id = 500;”`
```java
public static void testSyncQuery() throws Exception {
				...

        Runnable r = new Runnable() {
            @Override
            public void run() {
                try {
                    startSignal.await();
                    //String sql = "select * from hello";
                    String sql = "select * from hello_bigdata where id = 500;"; // 修改sql
		                ...
                } catch (Exception e) {

                }
            }
        };
...
    }
```

#### 火焰图分析
修改数据库后发现`waitingGet()`的开销占比为`0.15%`，删除Spins后的开销占比为`0.03%`。如果用加减法的角度看开销减少了`0.12%`，以乘法的角度看开销较少了`4/5`。能感觉还是有性能提升的。
![](images/CompletableFuture的get()在Kona8和Kona11的差异对比-5.png)



![](images/CompletableFuture的get()在Kona8和Kona11的差异对比-6.png)

