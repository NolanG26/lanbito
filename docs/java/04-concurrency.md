# Java 并发编程完整篇

> 学完基础语法、类和集合后再读。本章先回答“任务在哪个线程执行、什么时候完成、共享数据会不会改错”，再介绍并发工具。

## 本章怎么学

第一次先读 **第 1～4 节、第 7.1 节、第 8.1～8.3 节**：能启动和等待任务，能解释计数为什么会丢失，能看懂异步结果的组合就够了。`volatile`、高级锁、CAS、虚拟线程和排障内容可以第二遍再看。

本章使用 **JDK 21，不需要预览功能**。带有 `main` 的完整程序可以单独保存、编译并运行，例如 `javac ThreadDemo.java`、`java ThreadDemo`。仅有类定义的示例可以编译，需要另写 `main` 创建对象或调用方法。其余代码会注明放在 `main` 中还是类中；不同示例中的同名变量和同名类不要直接拼到一起。

并发示例中的 `main` 有时写成 `throws Exception`，是为了先集中理解线程：它允许等待操作的受检异常向上传播，**没有把异常处理掉**。在实际服务中应按调用边界处理失败和中断。

---

## 1. 并发模型对比：Node.js vs Java

先分清三个词：

| 概念 | 含义 | 例子 |
|------|------|------|
| 并发 | 多个任务在一段时间内交错推进 | 等数据库时处理另一个请求 |
| 并行 | 多个任务在同一时刻执行 | 两个 CPU 核心分别计算 |
| 异步 | 发起任务后，先继续做别的事，稍后取得结果 | 注册回调，收到结果再处理 |

异步不等于并行，多线程也不保证更快。增加线程会带来调度、内存和数据协调成本。

### Node.js：单线程事件循环

普通 JavaScript 回调通常在同一个事件循环线程中执行；I/O 可以由操作系统或 libuv 配合完成，也可以显式使用 Worker Threads。

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0); // 注册回调，稍后执行
console.log('3');
// 输出：1、3、2
```

`0` 不表示立刻执行，当前同步代码先执行完。单线程回调通常不会同时修改同一变量，但跨 `await` 的“先检查再修改”仍可能出现业务竞态；使用 Worker 和共享内存时也需要同步。Node.js 中的请求不会自动拥有各自独立的内存。

### Java：多线程模型

一个 Java 进程可以有多个线程。它们有各自的方法调用栈，也可以访问同一个堆对象，因此“两个任务都改同一个对象”需要额外考虑。

下面这行放在普通程序的 `main` 中：

```java
System.out.println(Thread.currentThread().getName()); // 通常是 main
```

`Thread.currentThread()` 取得**正在执行这一行的线程**，`.getName()` 取得线程名称。打印线程名只是观察当前线程，并没有创建另一个线程。

### 核心区别总结

| 问题 | Node.js 常见写法 | Java 常见写法 |
|------|----------------|--------------|
| 等待 I/O | Promise / async-await | 阻塞调用、异步 API、虚拟线程等 |
| 同时处理多个任务 | 事件循环，也可用 Worker | 多个线程、线程池、异步 API |
| 共享数据 | 模块变量也会跨请求共享 | 对象和静态字段也可能共享 |
| 是否需要防止改错数据 | 需要考虑异步竞态与共享内存 | 还需考虑多线程可见性、原子性等 |

---

## 2. 线程基础

先跑一个最小例子，观察 `start()` 和 `join()`。

**完整文件 `ThreadDemo.java`：**

```java
public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            System.out.println("工作线程：处理订单");
        }, "order-worker");

        worker.start();                       // 请求启动新线程，执行上面的任务
        System.out.println("主线程：继续做事");
        worker.join();                        // 当前主线程等待 worker 结束
        System.out.println("主线程：确认任务结束");
    }
}
```

读代码时按这四步走：

1. `new Thread(任务, 名字)` 创建线程对象，此时任务还没开始。
2. `() -> { ... }` 表示一个没有参数的任务；大括号内是它启动后要执行的代码。
3. `start()` 后，工作线程和主线程都能继续推进。因此前两行输出的顺序**不确定**。
4. `join()` 等工作线程结束才返回。因此“确认任务结束”一定在工作线程那行输出之后。

`join()` 阻塞当前线程；JavaScript 的 `await` 会暂停当前异步函数并让出事件循环，两者机制不同。

### 2.1 创建线程的三种方式

#### 方式一：继承 Thread 类

下面的 `MyThread` 单独放在 `MyThread.java`；使用部分放在另一个类的 `main` 中。

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("执行任务");
    }
}
```

```java
MyThread worker = new MyThread();
worker.start();
worker.join();
```

`extends Thread` 表示继承线程类，`@Override` 表示重写父类的方法。线程启动后会执行重写的 `run()`。

**不要把 `start()` 换成 `run()`**：直接调用 `run()` 只是当前线程执行普通方法；也不要对同一个线程对象调用两次 `start()`。

#### 方式二：实现 Runnable 接口（推荐）

`Runnable` 表示“可以执行、没有返回值的任务”，它本身不是线程。把任务和执行任务的线程分开，后面才能把同一个任务交给线程池。

下面两个任务是同一种逻辑的不同写法，放在 `main` 中：

```java
// 匿名类写法：创建一个实现 Runnable 的对象
Runnable traditionalTask = new Runnable() {
    @Override
    public void run() {
        System.out.println("执行任务");
    }
};

// Lambda 写法：编译器知道这里要提供 Runnable 的 run() 方法
Runnable shortTask = () -> System.out.println("执行任务");

Thread worker = new Thread(shortTask);
worker.start();
worker.join();
```

`new Runnable() { ... }` 后面的类体就是接口实现；Lambda 省略了这部分样板。两种写法任选一种即可，不需要把任务执行两遍。

#### 方式三：实现 Callable 接口（有返回值）

任务需要返回结果时，用 `Callable<T>`。`T` 是结果类型，例如 `Callable<String>` 表示任务完成后返回字符串。

下面代码放在 `main` 中，文件顶部添加这两个 `import`，`main` 声明 `throws Exception`：

```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;
```

```java
Callable<String> task = () -> "计算结果";
FutureTask<String> future = new FutureTask<>(task);
Thread worker = new Thread(future);
worker.start();

String result = future.get(); // 等待结果；失败时抛出异常
System.out.println(result);   // 计算结果
```

`FutureTask` 把 `Callable` 包装成既能交给线程执行、又能保存结果的对象。`Future<String>` 可以理解为“将来能拿到字符串结果的凭据”；`get()` 可能等待，因此不是单纯读一个字段。

### 2.2 线程生命周期

Java 的 `Thread.State` 有以下六种状态，**没有单独的 `RUNNING` 枚举值**：

| 状态 | 可以怎样理解 |
|------|--------------|
| `NEW` | 创建后还没启动 |
| `RUNNABLE` | 可运行，可能正在用 CPU，也可能等待 CPU 等资源 |
| `BLOCKED` | 等待进入另一个线程持有的 `synchronized` 监视器 |
| `WAITING` | 无超时地等待，例如 `join()`、`Object.wait()` |
| `TIMED_WAITING` | 有时间限制地等待，例如 `sleep()`、带超时的 `join()` |
| `TERMINATED` | 线程的 `run()` 已结束 |

下面放在 `main` 中，声明 `throws InterruptedException`：

```java
Thread worker = new Thread(() -> System.out.println("执行任务"));
System.out.println(worker.getState()); // NEW：还没有 start
worker.start();
System.out.println(worker.getState()); // 不固定：此时甚至可能已经结束
worker.join();
System.out.println(worker.getState()); // TERMINATED：join 已确认结束
```

`getState()` 是某一瞬间的观察，不要依赖连续两次状态读取来协调任务。I/O 等待也不能简单地全部归为 `WAITING`。

### 2.3 常用线程方法

| 方法 | 作用 | 容易误解的地方 |
|------|------|----------------|
| `worker.start()` | 启动线程 | 不保证立即获得 CPU |
| `worker.join()` | 当前线程等待 worker 结束 | 不负责启动 worker |
| `worker.join(1000)` | 最多等待约 1000 毫秒 | 返回时 worker 可能仍在运行，需检查 `isAlive()` |
| `worker.interrupt()` | 发出中断请求 | 不等于强制停止线程 |
| `worker.isAlive()` | 是否已启动且尚未结束 | 一次查询不能锁定后续状态 |
| `worker.setName("worker-1")` | 设置调试名称 | 不影响任务逻辑 |
| `Thread.sleep(1000)` | 让当前线程等待一段时间 | 不会释放已经持有的 `synchronized` 锁 |
| `Thread.currentThread()` | 取得当前线程 | 静态方法，与某个 worker 对象无关 |

`setPriority()` 和 `yield()` 只是调度提示，不应靠它们保证执行顺序。线程遇到中断时，需要检查中断标志，或处理 `InterruptedException`，才可能配合退出；中断也不代表业务副作用已经撤回。

---

## 3. 线程池详解

### 3.1 为什么不要手动创建线程？

学习时 `new Thread()` 很直观。但如果每次请求都创建一个平台线程，数量、内存和失败处理会很难管理。线程池负责复用一组工作线程，并管理任务排队与拒绝。

可以把它理解为：`Runnable` / `Callable` 是待办事项，线程是工作人员，线程池是安排工作的调度员。虚拟线程使用不同的资源模型，见第 9 节，不必把所有任务都塞进固定线程池。

### 3.2 ExecutorService 基础用法

**完整文件 `PoolDemo.java`，JDK 21：**

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class PoolDemo {
    public static void main(String[] args) throws Exception {
        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            executor.execute(() -> System.out.println("发送通知"));
            Future<Integer> total = executor.submit(() -> 10 + 20);
            System.out.println("计算结果：" + total.get());
        }
        System.out.println("线程池中的任务已结束");
    }
}
```

逐步看：

- `newFixedThreadPool(2)` 创建最多用两个工作线程执行任务的线程池。
- `execute()` 提交没有返回值的任务；`submit()` 返回 `Future`，可以取得结果或观察失败。
- `Future<Integer>` 表示结果是整数；`total.get()` 得到 `30`，必要时会等待。
- JDK 21 中 `ExecutorService` 支持 `try-with-resources`。离开 `try` 代码块时调用 `close()`，停止接收新任务并等待已提交任务结束。

“发送通知”和“计算结果”两行的先后不固定；最后一行一定在任务结束后输出。没有 `try` 管理资源的长期线程池，则应在它所属的应用或组件结束时关闭。

**关闭、等待、停止任务是三件事。** 下列片段是另一种关闭方式，不要接在上面已关闭的线程池后运行；放在已经创建并使用了 `executor` 的方法中，该方法需处理 `InterruptedException`：

```java
executor.shutdown(); // 发起有序关闭：不接新任务，已提交的继续执行；此方法不等待
boolean ended = executor.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
if (!ended) {
    executor.shutdownNow(); // 尝试中断正在执行的任务，返回尚未开始的任务
}
```

`awaitTermination()` 超时返回 `false`，并没有自动取消任务。`shutdownNow()` 也不保证正在运行的任务立即停止；是否退出取决于任务能否响应中断。

### 3.3 四种常用线程池

这些是创建方式对比；**每个示例的执行器都需要关闭**，不是让你同时创建四个池。相关类型来自 `java.util.concurrent` 包。

#### ① Fixed Thread Pool（固定大小）

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
```

最多四个工作线程执行，其余任务排队。默认队列没有容量上限，提交速度长期大于处理速度时，排队仍可能耗尽内存。CPU 密集型任务的线程数通常从可用核心数附近开始测量，不存在通用的“核心数 + 1”最佳公式。

#### ② Cached Thread Pool（动态大小）

```java
ExecutorService executor = Executors.newCachedThreadPool();
```

优先复用空闲线程，没有可用线程时可能继续创建；默认空闲一段时间后回收。它不是自动限流工具，大量慢任务可能导致线程数失控。

#### ③ Single Thread Executor（单线程）

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```

一次执行一个任务，适合必须串行处理的工作。它也使用无界队列；“只有一个线程”不代表提交再多任务都安全。

#### ④ Scheduled Thread Pool（定时任务）

下面放在 `main` 中，文件顶部 `import java.util.concurrent.*;`，`main` 声明 `throws Exception`：

```java
try (ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1)) {
    ScheduledFuture<?> scheduled = scheduler.schedule(
        () -> System.out.println("延迟通知"),
        1,
        TimeUnit.SECONDS
    );
    scheduled.get(); // 等待这一次任务执行完，输出“延迟通知”
}
```

`1` 和 `TimeUnit.SECONDS` 合起来是“延迟约一秒后具备执行条件”，不保证精确一秒执行。

周期任务使用 `scheduleAtFixedRate(task, 初始延迟, 周期, 单位)`，返回的 `ScheduledFuture` 可以 `cancel(false)` 来停止后续调度。同一个周期任务不会与自身重叠；任务运行过慢会影响后续开始时间，未处理的异常会阻止后续周期执行。周期任务也要明确取消和执行器关闭的时机，避免程序结束后仍有线程等待调度。

### 3.4 自定义线程池（进阶：先理解参数）

下面是一个**参数演示**，不是可以照搬到任意服务的容量配置。放在方法中，添加 `import java.util.concurrent.*;`，使用后按第 3.2 节关闭：

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                                       // corePoolSize
    8,                                       // maximumPoolSize
    60,                                      // keepAliveTime
    TimeUnit.SECONDS,                        // 上一个数字的单位
    new LinkedBlockingQueue<>(100),          // 最多排队 100 个任务
    Executors.defaultThreadFactory(),        // 创建工作线程的工厂
    new ThreadPoolExecutor.AbortPolicy()      // 满载时显式拒绝
);
```

| 参数 | 这里的含义 |
|------|------------|
| 核心数 `4` | 通常先按需创建到四个核心线程 |
| 最大数 `8` | 核心线程忙、队列也满时，才继续增加到最多八个 |
| 空闲时间 `60 秒` | 默认主要控制超过核心数量的空闲线程何时回收 |
| 队列容量 `100` | 核心线程都忙时，新任务先进入队列 |
| 线程工厂 | 负责创建线程，也可以配置名称等属性 |
| 拒绝策略 | 达到线程和队列限制，或已关闭时，怎样处理新任务 |

这里的顺序是“到核心数 → 排队 → 队列满后扩到最大数 → 再满就拒绝”，并不是一开始启动八个线程。

常见拒绝策略：`AbortPolicy` 抛 `RejectedExecutionException`；`CallerRunsPolicy` 在池未关闭时让提交者自己执行，可能阻塞请求线程；`DiscardPolicy` 静默丢弃；`DiscardOldestPolicy` 尝试丢弃队列最前的等待任务再提交。后两种可能导致工作丢失，应明确业务是否允许，不能靠它们隐藏过载。

---

## 4. synchronized 关键字

### 4.1 为什么需要同步？

假设两个线程都要让 `count` 加一。`count++` 在逻辑上包含“读取、加一、写回”三步，可能发生：

| 步骤 | 线程 A | 线程 B |
|------|--------|--------|
| 1 | 读到 `0` | |
| 2 | | 也读到 `0` |
| 3 | 写回 `1` | |
| 4 | | 也写回 `1` |

做了两次加一，最终却只有 `1`。这叫**丢失更新**。它不一定每次出现，因此“运行一次结果正确”不能证明线程安全。

需要保护的是“共同访问同一个计数器”的操作，而不只是某一行看起来够不够短。

### 4.2 三种锁的粒度

#### 方法锁（对象级别）

先用完整示例解决上面的计数问题。**文件 `CounterDemo.java`：**

```java
public class CounterDemo {
    static class Counter {
        private int count = 0;

        public synchronized void increment() {
            count++;
        }

        public synchronized int getCount() {
            return count;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        };
        Thread first = new Thread(task);
        Thread second = new Thread(task);
        first.start();
        second.start();
        first.join();
        second.join();
        System.out.println(counter.getCount()); // 2000
    }
}
```

`static class Counter` 是定义在外部类中的辅助类，让示例能放在一个文件里。两个线程共享同一个 `counter`。实例方法上的 `synchronized` 使用这个对象自身，也就是 `this`，作为锁：同一时刻，只有一个线程能进入由**同一把锁**保护的代码。方法正常返回或抛出异常时，锁都会自动释放。

`join()` 确认两次循环都完成后才读结果。如果两个线程各自创建一个 `Counter`，它们锁住的是不同对象，就不是共同维护这个计数器了。

#### 对象锁（同步代码块）

下面是另一种 `Counter` 定义，单独放在 `Counter.java`；不要和上一个类重名拼接：

```java
public class Counter {
    private int count = 0;
    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {
            count++;
        }
    }

    public int getCount() {
        synchronized (lock) {
            return count;
        }
    }
}
```

`lock` 是大家约定使用的锁对象，`final` 防止它被换成另一把锁。所有共享读写都使用同一个锁；把不访问共享状态的工作移到块外，可以减少占用锁的时间，但不保证一定更快。

#### 类锁（静态方法/类对象）

下面两个方法是**两种写法**，类成员片段：

```java
private static int instanceCount = 0;

public static synchronized void incrementInstance() {
    instanceCount++;
}

public static void incrementInstance2() {
    synchronized (Counter.class) {
        instanceCount++;
    }
}
```

将它们放在名为 `Counter` 的类中。`static` 字段属于类，两个方法都锁 `Counter.class`；它和某一个实例的 `this` 锁是不同的。若要并发读取这个字段，读取也应采用相应的同步方式。

### 4.3 死锁示例与避免

死锁可以先看等待关系，不必先运行一个会挂住的程序：

```text
线程 A：已持有 lockA → 等待 lockB
线程 B：已持有 lockB → 等待 lockA
```

下面是**风险结构示意**，省略线程创建，不是让你直接复制执行：

```java
// 线程 A 中
synchronized (lockA) {
    synchronized (lockB) { /* 处理共享数据 */ }
}

// 线程 B 中：获取锁的顺序相反
synchronized (lockB) {
    synchronized (lockA) { /* 处理共享数据 */ }
}
```

只有在特定交错下才会卡住，不能说每次执行都一定死锁。最直接的改法是让所有路径都先获取 `lockA` 再获取 `lockB`；也可减少嵌套，或设计获取失败后的退出策略。后面 `tryLock()` 的超时可以帮助退出等待，但工具本身不自动证明程序没有死锁。

---

## 5. volatile 关键字

### 5.1 可见性问题

`volatile` 适合“一个线程修改标志，另一个线程据此行动”的简单状态通信。先看**完整文件 `StopDemo.java`**：

```java
public class StopDemo {
    private static volatile boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            while (running) {
                // 仅为演示：实际任务在这里工作，不应长期空转占用 CPU
            }
            System.out.println("工作线程退出");
        });
        worker.start();
        running = false;
        worker.join();
        System.out.println("主线程确认结束");
    }
}
```

`running = false` 是发出停止信号；工作线程重新读取后退出循环；`join()` 才是确认线程结束。示例输出依次为“工作线程退出”“主线程确认结束”，但并不保证工作线程一定先进入过循环。

去掉 `volatile` 后，这种没有其他同步的读写就没有可靠的可见性保证，工作线程可能持续读不到变化。原因涉及 Java 内存模型和编译器优化，不能简单理解成“所有读写都绕开缓存，直接访问物理内存”。

### 5.2 volatile 的限制

`volatile` 保证相应读写的可见性和内存顺序，但不会把多个步骤合成一个原子操作。下面是**不安全的类成员片段**：

```java
private volatile int count = 0;

public void increment() {
    count++; // 仍是读、加一、写回；多个线程可能互相覆盖
}
```

因此：停止标志可以考虑 `volatile boolean`；多个线程加一用锁或第 7 节的原子类。对对象引用加 `volatile` 也不会自动保护对象内部所有字段。

---

## 6. Lock 接口

### 6.1 ReentrantLock vs synchronized

`Lock` 提供显式的“加锁、解锁”操作。先理解这个结构，放在独立的 `Counter.java` 中：

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private int count;
    private final Lock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

`lock.lock()` 成功返回后才进入 `try`；`finally` 确保业务代码即使抛异常也会解锁。`synchronized` 自动释放锁，`Lock` 则需要自己保证这一点。若增加读取方法，也要按同一个同步约定访问 `count`。

“可重入”指同一个线程已经持有这把锁时，可以再次获取它；每次获取都需要对应释放。它不表示不同线程能同时进入。

### 6.2 ReentrantLock 的高级特性

下面是**互相独立的控制流程片段**；假设 `lock` 是上一节那样的字段，相关类型从 `java.util.concurrent` / `.locks` 导入。

```java
if (lock.tryLock()) { // 立即尝试，得到 true 才表示拿到锁
    try {
        System.out.println("已拿到锁，处理任务");
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("没有拿到锁，稍后重试或返回失败");
}
```

不要在 `else` 中解锁，因为当前线程没有获得这把锁。需要最多等待五秒时，可用 `lock.tryLock(5, TimeUnit.SECONDS)`；它会抛 `InterruptedException`，调用方法必须处理或声明。

```java
try {
    lock.lockInterruptibly(); // 等待锁期间允许响应中断
    try {
        System.out.println("执行任务");
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断标志，向上层保留信号
    // 这里结束本次操作，不继续执行需要锁的业务
}
```

恢复中断标志不等于自动结束整个线程，后续流程仍需要按约定退出或向上传播失败。

### 6.3 读写锁（ReadWriteLock）

需要保护一个“读多、写少”的共享 `Map` 时，可以让多个读者同时读，而写入时独占。下面是**独立文件 `Cache.java`**：

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache {
    private final Map<String, String> data = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    public String get(String key) {
        readLock.lock();
        try {
            return data.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(String key, String value) {
        writeLock.lock();
        try {
            data.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

两把锁来自同一个 `rwLock`，所以能互相协调。调用 `put("name", "Alice")` 后再调用 `get("name")` 得到 `"Alice"`；查不到则是 `null`。不要绕过这些方法直接访问内部 `data`。读写锁有维护成本，是否比普通锁合适需要结合竞争和测量。

### 6.4 何时用 ReentrantLock vs synchronized？

| 需求 | 可以考虑 |
|------|----------|
| 简单保护几行共享操作 | 先用 `synchronized` |
| 尝试获取、超时等待、可中断等待 | `ReentrantLock` |
| 读多写少，需要多个读者并行 | 评估读写锁 |
| 希望更偏向按等待顺序分配锁 | 评估 `new ReentrantLock(true)` |

公平锁不是绝对的线程调度顺序保证，也可能降低吞吐。先确定需要哪种行为，再选工具。

---

## 7. Atomic 原子类

### 7.1 AtomicInteger

如果只要安全地加一，不需要保护一整段业务流程，可以用 `AtomicInteger`。下面代码放在 `main` 中，文件顶部添加 `import java.util.concurrent.atomic.AtomicInteger;`：

```java
AtomicInteger counter = new AtomicInteger(0);
System.out.println(counter.incrementAndGet()); // 1：先加一，再返回新值
System.out.println(counter.getAndIncrement()); // 1：先返回旧值，同时加一
System.out.println(counter.get());             // 2
System.out.println(counter.addAndGet(5));      // 7
System.out.println(counter.compareAndSet(7, 10)); // true：当前是 7，改为 10
System.out.println(counter.compareAndSet(7, 20)); // false：当前已是 10，不修改
System.out.println(counter.get());             // 10
```

这里每一步的结果都能算清楚。把第 4 节示例的 `Counter` 改为 `AtomicInteger`，两个线程都调用 `incrementAndGet()`，等两次 `join()` 后结果同样是 `2000`。

原子方法只保证**这一次操作**不可分割；“先 `get()` 检查，再单独修改”不一定作为整体原子，也不能自动维护多个字段之间的业务约束。

### 7.2 AtomicReference

`AtomicReference<T>` 原子地更新的是**对象引用**，不是对象内所有属性。下面放在 `main` 中，顶部添加 `import java.util.concurrent.atomic.AtomicReference;`：

```java
String initial = new String("初始值");
AtomicReference<String> ref = new AtomicReference<>(initial);

System.out.println(ref.compareAndSet(initial, "新值")); // true
System.out.println(ref.compareAndSet(initial, "另一个值")); // false
System.out.println(ref.get()); // 新值
```

第一个参数是“我预期当前引用仍指向哪个对象”。这里比较引用身份，不调用 `equals()`；即使另造一个内容相同的字符串，也不能当作同一个预期引用。

### 7.3 CAS 原理简介

CAS 是 Compare And Set/Swap，可以理解成一次不可分割的条件更新：

```text
如果当前值仍等于我预期的旧值：改成新值，返回 true
否则：什么也不改，返回 false
```

这只是语义说明，自己用普通 `if` 写出相同文本并不会获得原子性。原子类由 JVM 与底层平台机制提供这种保证；有竞争时可能需要重试，因此也不是“永远没有等待成本”。

进阶问题：某引用经历 `A → B → A`，只检查当前引用的 CAS 无法发现中间变化，称为 ABA 问题。如果业务在意变化历史，可以给状态加版本号。

下面放在 `main` 中，顶部添加 `import java.util.concurrent.atomic.AtomicStampedReference;`：

```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
int[] stampHolder = new int[1];
String oldValue = ref.get(stampHolder); // 同时读取引用与版本，版本写入数组第 0 项
int oldStamp = stampHolder[0];
boolean changed = ref.compareAndSet(oldValue, "B", oldStamp, oldStamp + 1);
System.out.println(changed); // 没有其他线程修改时为 true
```

`int[1]` 在这里用来带回第二个结果，不是存业务列表。CAS 要求引用和版本都匹配；是否需要这类工具取决于业务，第一次学习不用自己实现无锁算法。

---

## 8. CompletableFuture 完整教程

`CompletableFuture<T>` 表示一个将来可能成功或失败的结果，并允许注册后续处理。它可以类比 Promise 来理解链条，但线程和阻塞行为不同。

### 8.1 创建 CompletableFuture

先看**完整文件 `FutureDemo.java`**：

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FutureDemo {
    public static void main(String[] args) {
        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            CompletableFuture<Integer> total = CompletableFuture.supplyAsync(
                () -> 10 + 20,
                executor
            );
            System.out.println("主线程可以先做其他事");
            System.out.println("结果：" + total.join()); // 结果：30
        }
    }
}
```

`CompletableFuture<Integer>` 中的 `Integer` 是最终结果类型。`supplyAsync()` 把计算交给执行器；创建 Future 本身不等于计算已经结束。主线程先打印提示，然后 `join()` 等待结果。

| 方法 | 用途 |
|------|------|
| `supplyAsync(() -> 值, executor)` | 异步任务有结果 |
| `runAsync(() -> { 动作; }, executor)` | 异步任务没有结果，得到 `CompletableFuture<Void>` |
| `completedFuture(值)` | 创建已经成功完成的结果，不启动异步任务 |
| `future.get()` | 阻塞等待；以受检异常形式报告中断、任务失败等 |
| `future.join()` | 阻塞等待；任务失败通常抛 `CompletionException` |

不传执行器的 `supplyAsync()` 通常使用公共 `ForkJoinPool`；不要因此把无限阻塞的 I/O 都交给公共池。`Supplier` 是提供返回值的函数接口，它的回调不允许直接抛出 `InterruptedException`，所以不能直接在里面写未处理的 `Thread.sleep()`；完整的处理方式见第 8.6 节。

### 8.2 链式处理

下面放在 `main` 中，顶部添加 `import java.util.concurrent.CompletableFuture;`：

```java
CompletableFuture<String> result = CompletableFuture
    .completedFuture("hello")
    .thenApply(text -> text.toUpperCase())
    .thenApply(text -> text + "!");

System.out.println(result.join()); // HELLO!
```

按值的变化读这条链：`"hello" → "HELLO" → "HELLO!"`。`text` 是上一阶段的结果，`thenApply()` 把它转换成下一阶段的普通值。

```java
CompletableFuture<Void> printed = result.thenAccept(
    text -> System.out.println("结果：" + text)
);
printed.join(); // 确认打印动作也已完成

CompletableFuture<Void> finished = result.thenRun(
    () -> System.out.println("任务完成")
);
finished.join();
```

`thenAccept()` 用到上一步的结果，但不产生业务结果；`thenRun()` 连上一步的值也不接收。两者都返回 `CompletableFuture<Void>`，仍然可以继续串接或等待。

**没有 `Async` 后缀的回调不保证换线程**：可能由完成前一步的线程执行，也可能在注册回调时由当前线程执行。需要提交给指定执行器时，用对应的 `thenApplyAsync(..., executor)` 等方法。

### 8.3 组合多个 Future

先分清“回调返回普通值”与“回调又返回一个 Future”：

```java
CompletableFuture<Integer> length = CompletableFuture
    .completedFuture("Alice")
    .thenApply(name -> name.length()); // 返回普通整数，最终是 5

CompletableFuture<String> greeting = CompletableFuture
    .completedFuture("Alice")
    .thenCompose(name -> CompletableFuture.completedFuture("你好，" + name));
System.out.println(greeting.join()); // 你好，Alice
```

`thenCompose()` 的回调返回下一项异步任务，它等待这项任务并展开结果。若这里改成 `thenApply()`，类型会变成 `CompletableFuture<CompletableFuture<String>>`，外层完成不等于内层任务完成。

实际用途是“先查用户 ID，再凭 ID 查资料”，后一步依赖前一步，不能直接同时启动。

两项任务互不依赖时，先各自启动，再用 `thenCombine()` 合并。下面放在 `main` 中，顶部添加 `import java.util.concurrent.*;`：

```java
try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
    CompletableFuture<String> name = CompletableFuture.supplyAsync(() -> "Alice", executor);
    CompletableFuture<Integer> points = CompletableFuture.supplyAsync(() -> 100, executor);
    CompletableFuture<String> summary = name.thenCombine(
        points,
        (userName, userPoints) -> userName + "：" + userPoints + " 分"
    );
    System.out.println(summary.join()); // Alice：100 分
}
```

两个值按参数位置传给合并函数，不按谁先完成排序。`thenCombine()` 负责等两项都成功；能否并行取决于任务如何启动、执行器和运行资源，并不是它自动把任意代码变成并行。

### 8.4 allOf 和 anyOf

`allOf()` 适合等待一组任务；它本身没有结果列表。下面用已经完成的结果来先看清类型和输出，放在 `main` 中：

```java
CompletableFuture<String> first = CompletableFuture.completedFuture("结果1");
CompletableFuture<String> second = CompletableFuture.completedFuture("结果2");
CompletableFuture<Void> all = CompletableFuture.allOf(first, second);
all.join();
System.out.println(first.join() + ", " + second.join()); // 结果1, 结果2
```

实际使用时可把 `first`、`second` 换成提前提交的异步任务。`allOf()` 等所有任务都完成；若有任务失败，组合结果也会失败，其后的成功回调不会执行。

`anyOf()` 等**任意一个任务完成，包括异常完成**。下面手动完成两个结果，避免把机器调度速度误当成固定规则：

```java
CompletableFuture<String> slow = new CompletableFuture<>();
CompletableFuture<String> fast = new CompletableFuture<>();
CompletableFuture<Object> any = CompletableFuture.anyOf(slow, fast);
fast.complete("先完成");
System.out.println(any.join()); // 先完成
slow.complete("后完成");       // 另一项依然可以继续完成
```

`complete()` 在这里模拟外部事件提供结果，不是创建线程。`anyOf()` 返回 `Object` 是因为输入任务的类型可能各不相同；它不会自动取消其他任务，也不是“等第一个成功结果”。

### 8.5 异常处理

先用确定失败的例子理解异常链，下面放在 `main` 中：

```java
CompletableFuture<String> failed = CompletableFuture.failedFuture(
    new IllegalStateException("服务不可用")
);
CompletableFuture<String> recovered = failed.exceptionally(error -> "暂不可用");
System.out.println(recovered.join()); // 暂不可用
```

`exceptionally()` 只在失败时执行，返回值会把**派生的** Future 恢复为成功。这个例子假设业务允许显示“暂不可用”；创建订单、扣库存等操作不能随意把失败替换成默认值。

| 方法 | 何时调用 | 是否能改变后续结果 |
|------|----------|--------------------|
| `exceptionally(error -> 值)` | 失败时 | 可以把失败恢复成值 |
| `handle((value, error) -> 值)` | 成功和失败时 | 可以转换值或处理失败 |
| `whenComplete((value, error) -> { ... })` | 成功和失败时 | 通常用于观察、记录；回调自身抛错仍会影响派生结果 |

`handle()` 不是 `finally`：它必须返回后续阶段的值。要保证资源关闭，仍然用 `try/finally` 或 `try-with-resources`。

还要分清**等待超时**与**任务停止**：

- `get(1, TimeUnit.SECONDS)`：当前调用最多等一段时间，超时会抛异常，原任务仍可能继续。
- `orTimeout(1, TimeUnit.SECONDS)`：使这个 `CompletableFuture` 在超时后异常完成，不保证停止底层计算或网络请求。
- `completeOnTimeout(默认值, 1, TimeUnit.SECONDS)`：超时后以默认值完成，也不代表底层工作取消。
- `CompletableFuture.cancel(true)` 也不会自动中断 `supplyAsync()` 中的计算；不能把它当成“撤回已经发出的请求”。

如果迟到的任务还会写数据库或更新页面，必须另外设计请求取消、幂等和结果是否仍有效的判断。

### 8.6 完整示例：模拟商品查询

现在把“发起三个任务 → 等全部成功 → 读取结果 → 合并展示”连起来。**完整文件 `ProductService.java`**，模拟函数只返回固定数据，不访问真实数据库：

```java
import java.math.BigDecimal;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ProductService {
    private final ExecutorService executor;

    public ProductService(ExecutorService executor) {
        this.executor = executor;
    }

    private static void pause(long milliseconds) {
        try {
            Thread.sleep(milliseconds);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new CompletionException(e);
        }
    }

    public CompletableFuture<String> getProduct(long id) {
        return CompletableFuture.supplyAsync(() -> {
            pause(100);
            return "商品:" + id;
        }, executor);
    }

    public CompletableFuture<Integer> getStock(long id) {
        return CompletableFuture.supplyAsync(() -> {
            pause(80);
            return 100;
        }, executor);
    }

    public CompletableFuture<BigDecimal> getPrice(long id) {
        return CompletableFuture.supplyAsync(() -> {
            pause(120);
            return new BigDecimal("99.9");
        }, executor);
    }

    public CompletableFuture<String> getProductDetail(long id) {
        CompletableFuture<String> product = getProduct(id);
        CompletableFuture<Integer> stock = getStock(id);
        CompletableFuture<BigDecimal> price = getPrice(id);

        return CompletableFuture.allOf(product, stock, price)
            .thenApply(ignored -> String.format("%s, 库存:%d, 价格:%.1f",
                product.join(), stock.join(), price.join()));
    }

    public static void main(String[] args) {
        try (ExecutorService executor = Executors.newFixedThreadPool(3)) {
            ProductService service = new ProductService(executor);
            String detail = service.getProductDetail(1L).join();
            System.out.println(detail);
        }
    }
}
```

先不要逐字符背，按调用链看：

1. `main` 创建三个工作线程的池，并把它传给 `ProductService`；创建它的人负责关闭。
2. `getProductDetail(1L)` 连续提交商品、库存、价格三个独立任务，中间没有逐项等待。
3. `allOf()` 在三个任务全部成功后执行 `thenApply()`。`ignored` 表示这里不需要 `Void` 类型的值。
4. 此时三个任务已经成功完成，回调中的 `join()` 直接取得它们的结果，再拼成一行。
5. 最外层 `join()` 等最终字符串完成；离开 `try` 时关闭执行器。

价格用基础章介绍过的 `BigDecimal` 表示。`String.format()` 中的 `%s`、`%d`、`%.1f` 分别放入字符串、整数、保留一位小数的数字。典型默认数字格式环境下输出 `商品:1, 库存:100, 价格:99.9`。三个模拟等待有机会重叠，但总耗时还受调度等影响，不能保证正好 120 毫秒。

`sleep()` 抛出 `InterruptedException` 时会清除当前线程的中断标志，所以 `pause()` 用 `interrupt()` 恢复这个信号；再把受检异常包装成 `CompletionException`，让 Future 以失败结束。这解决了 `Supplier` 不能直接抛受检异常的问题，也没有把中断误当成正常查询结果；真实接口还需要自己的超时和取消约定。

---

## 9. Java 21 虚拟线程（Virtual Threads）

### 9.1 什么是虚拟线程？

平台线程通常对应操作系统线程；虚拟线程由 JVM 调度，多个虚拟线程可以复用平台线程作为“载体”。它主要帮助应用用较直观的阻塞代码处理大量等待型任务。

例如一个请求等待远端接口时，支持卸载的阻塞操作可以让载体去运行其他虚拟线程。它不会让 CPU 计算本身更快，也不意味着所有阻塞都能卸载。线程可承载数量和内存成本取决于实际程序，不宜记成固定的“每个 1KB、一定能创建百万个”。

### 9.2 使用虚拟线程

单个虚拟线程的用法和普通线程很像，下面放在 `main` 中，声明 `throws InterruptedException`：

```java
Thread worker = Thread.ofVirtual().start(() -> {
    System.out.println(Thread.currentThread().isVirtual()); // true
});
worker.join();
```

`ofVirtual()` 选择虚拟线程构建方式；这里的 `start()` 已经创建并启动线程，不要再对 `worker` 调一次 `start()`。

多个任务可以使用每任务一个虚拟线程的执行器。下面放在 `main` 中，顶部添加 `import java.util.concurrent.*;`，`main` 声明 `throws Exception`：

```java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> first = executor.submit(() -> {
        Thread.sleep(100); // Callable 允许抛受检异常，用等待模拟 I/O
        return "任务 A 完成";
    });
    Future<String> second = executor.submit(() -> "任务 B 完成");
    System.out.println(first.get());
    System.out.println(second.get());
}
```

两项任务提前提交；打印时先读取 `first`，所以输出顺序为 A、B，不能据此反推 A 先完成。`get()` 也能让任务失败被观察到；离开 `try` 时关闭执行器并等待任务退出。

### 9.3 虚拟线程最佳实践

- 适合大量等待 HTTP、数据库等 I/O 的任务；CPU 密集型工作仍要限制并行度。
- 虚拟线程执行器不是并发上限，数据库连接、远端接口容量仍需要连接池或限流等保护。
- **在 JDK 21 中**，虚拟线程在 `synchronized` 内执行某些阻塞操作可能固定载体线程。优先避免持锁做慢 I/O；确有需要时再评估改用 `ReentrantLock`，不必机械替换所有锁。
- Spring Boot 3.2+ 有虚拟线程相关配置，但 `spring.threads.virtual.enabled=true` 不代表项目内所有自定义线程池都会自动转换。

第一次读到这里，只要记住“虚拟线程让等待更容易扩展，不会让下游容量无限增长”即可。

---

## 10. 并发常见坑

### 10.1 竞态条件（Race Condition）

“先判断没有，再创建”的操作也可能被两个线程同时执行。下面是**错误结构片段**，假设在一个名为 `Singleton` 的类中：

```java
private static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```

两个线程都可能通过 `if`，最终创建多个对象。进阶资料常给出“双重检查锁”：外层快速检查、进入锁后再检查，并把字段声明为 `volatile` 来保证安全发布。它需要同时理解锁和内存模型，不适合先背模板。

先看更容易解释的**完整文件 `Singleton.java`**：

```java
public final class Singleton {
    private Singleton() {} // 外部不能随意 new

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

首次使用 `Holder` 时 JVM 初始化它的静态字段，这个初始化过程有线程安全保证。在普通调用中两次 `Singleton.getInstance()` 得到同一引用。实际 Spring 应用通常让容器管理服务实例，不需要给每个类手写单例。

### 10.2 死锁排查

对于本地或有权限检查的 Java 进程，终端命令：

```bash
jstack <pid>
```

`<pid>` 是目标 Java 进程的真实进程号，不要原样输入尖括号。输出线程转储，查看是否报告死锁，并检查哪些线程持有什么锁、等待什么锁；一次快照没有死锁报告不代表没有其他阻塞或历史上从未死锁。

### 10.3 内存泄漏

`ThreadLocal<T>` 给每个线程保存一份值。线程池会复用线程，任务结束却没清理时，旧值可能留存或被后续任务误用。

下面是**完整类定义，保存为 `RequestContext.java`**：

```java
public class RequestContext {
    private static final ThreadLocal<String> userId = new ThreadLocal<>();

    public static void setUserId(String id) { userId.set(id); }
    public static String getUserId() { return userId.get(); }
    public static void clear() { userId.remove(); }
}
```

下面是另一个方法中的调用片段：

```java
try {
    RequestContext.setUserId("demo-user");
    System.out.println(RequestContext.getUserId()); // demo-user
} finally {
    RequestContext.clear();
}
```

`finally` 让正常和异常路径都清理。它只清理**当前线程**的值；把任务交给另一个线程并不会自动传递上下文，应遵循框架或项目的上下文传播方案。

### 10.4 线程池任务异常被吞掉

最常见的坑是：`submit()` 把任务失败存进 Future，却没人读取结果。`execute()` 的未捕获运行时异常通常会交给线程的未捕获异常处理器，并非天然静默吞掉。

下面放在 `main` 中，顶部添加 `import java.util.concurrent.*;`，`main` 声明 `throws InterruptedException`：

```java
try (ExecutorService executor = Executors.newFixedThreadPool(1)) {
    Future<?> future = executor.submit(() -> {
        throw new IllegalStateException("任务失败");
    });
    try {
        future.get();
    } catch (ExecutionException e) {
        System.err.println("失败原因：" + e.getCause().getMessage());
    }
}
```

`Future<?>` 表示这里不关心成功返回值的具体类型，但仍要观察失败。`get()` 把原始任务异常包在 `ExecutionException` 中，`getCause()` 取得里面的真实原因。打印日志只是这里演示如何观察异常，业务还需要决定如何向调用者报告失败，不能打印后假装成功。

### 10.5 快速参考：并发工具选型

| 需求 | 可以考虑 | 前提 |
|------|----------|------|
| 单个计数器加一 | `AtomicInteger` | 不涉及多个字段联动 |
| 保护一组共享操作 | `synchronized` / `ReentrantLock` | 所有相关路径遵循同一把锁 |
| 异步计算和结果组合 | `CompletableFuture` | 明确执行器、异常与任务生命周期 |
| 等一组任务完成 | `allOf()` | 不会自动收集结果，也不自动取消其他任务 |
| 延迟或周期执行 | `ScheduledExecutorService` | 管理周期取消与异常 |
| 大量等待型任务 | 虚拟线程 | 继续限制下游资源与并发量 |
| 在线程间传递数据 | `BlockingQueue` | 用队列协调生产者、消费者 |
| 等待一批工作到齐 | `CountDownLatch` | 计数到零后不重置 |
| 多轮到齐后再继续 | `CyclicBarrier` | 需要处理某一方失败的情况 |
| 限制同时进入的任务数 | `Semaphore` | 获取与释放许可必须配对 |

后三种协调工具先知道用途即可，遇到实际需求后再展开，避免一次背太多 API。

---

## 总结

读并发代码时，依次问：任务在哪个线程执行？有没有共享可变数据？谁等待结果、处理失败并关闭资源？超时后底层任务还会不会继续？

能不共享状态就尽量不共享；必须共享时，先明确哪些操作要作为一个整体，再决定使用锁、原子操作或其他协调机制。

### 自测：能解释这些就可以继续

1. **把 `start()` 改成 `run()` 会怎样？** 不启动新线程，由当前线程执行普通方法。
2. **两个线程各自加一千次，为什么要先 `join()` 再读总数？** 必须确认两项工作都结束；还要用锁或原子类避免丢失更新，单独等待不能修复不安全的加一。
3. **`volatile int count` 能安全执行 `count++` 吗？** 不能，读改写不是一个原子操作。
4. **回调返回字符串用什么？返回另一个 Future 呢？** 分别用 `thenApply()` 和 `thenCompose()`。
5. **`orTimeout()` 触发后，数据库查询一定停止了吗？** 不一定。它改变 Future 的完成状态，底层查询取消需要另行处理。
6. **换成虚拟线程后可以无限并发访问数据库吗？** 不能。数据库连接与服务容量仍然有限。
