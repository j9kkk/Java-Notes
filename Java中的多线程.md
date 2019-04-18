# Java中的多线程

## 准备知识

### 进程与线程

> 进程：由操作系统内核创建有独立虚拟地址空间的逻辑运行单元
>
> 线程：运行在进程上下文中的逻辑流，每个独立的线程有独自的Thread ID、栈、栈指针、程序计数器、通用目的寄存器、条件码；同进程中的所有线程共享该进程的整个虚拟地址内存空间，相比IPC，数据共享成本更低

### 并发编程

以典型的客户端-服务端编程模型为例

> 基于进程的并发编程：父进程接收请求，然后创建一个新的子进程来处理实际请求

![屏幕快照 2019-03-31 19.46.50](https://ws3.sinaimg.cn/large/006tKfTcgy1g1m7v10cavj30p40fwdhf.jpg)

> 基于I/O多路复用的并发编程：系统将进程挂起，只有当监听的I/O事件发生后才通知进程进行处理

![屏幕快照 2019-03-31 18.53.05](https://ws2.sinaimg.cn/large/006tKfTcgy1g1m6cfj62qj314609cdhf.jpg)

> 基于线程的并发编程：在同一进程中为每个请求创建独立的线程进行处理

![屏幕快照 2019-03-31 19.52.32](https://ws3.sinaimg.cn/large/006tKfTcgy1g1m80m34j7j30x209y3zp.jpg)

### 线程安全

> 当进行并发编程时，共享的资源对象将因并发读写而产生不一致的问题，这种问题统称为线程安全问题
>
> 解决线程安全问题的手段一般有：互斥同步、非阻塞同步、无同步设计

***互斥同步*** 

同步是指多个线程在并发访问共享数据时，保证共享数据在同一时刻只被一个线程使用；而互斥是指实现同步的手段，比如临界区、互斥量、信号量等。互斥是方法，同步是目的

***非阻塞同步***

互斥同步属于一种悲观锁，这种机制的理念实际认为只要不去做正确的同步措施，就一定会出现线程安全问题。非阻塞同步则属于一种乐观锁，即先进行操作，若有冲突再执行补偿措施如重试等。比较典型的操作有***CAS***(Compare-and-Swap)，这种类型的操作目前都在CPU指令级别支持，保证了操作的原子性

***无同步设计***

要保证线程安全，并不是一定就要进行同步，二者没有因果关系。如果能在设计上避免资源的访问产生冲突，那么就可以实现无同步设计。比如利用值传递函数参数，避免依赖堆上数据或公共资源；或者利用ThreadLocalMap对象将资源定义为本地线程独享来避免其他线程引起的冲突。比较典型的操作有双`buffer list`设计

### 监视器

为界定同步区域而监视的参考对象，使用同一个监视器的不同线程在进入对应同步区域时，必须等待前一个线程释放对监视器的占有权；监视器对象可以是同步资源本身也可以是单纯的无业务意义的任意对象，二者不存在关联关系；由同一个监视器确认的同步区域称为***临界区***

### 锁类型

***互斥锁/排他锁***

是对一种锁的统称，这种类型的锁因为锁定了共享资源，限制同一时刻只允许一个线程访问从而起到互相排斥的效果

***自旋锁和自适应自旋***

使当前等待进入同步区的线程不放弃CPU执行时间而执行一个忙循环，以此避免线程之间的上下文切换带来的性能损失。这类锁的使用前提是预知当前锁可在短时间内获取到，如果锁一直获取不到但是又在不停自旋，这样反而会使线程浪费过多的CPU执行时间，得不偿失

***锁消除***

编译器对不可能产生冲突的资源进行锁消除优化

***锁粗化***

一般情况下锁的粒度越小越好，这表示更快的释放共享资源；但是有些情况下如循环加锁与释放锁，此时就应该将锁的粒度适当放大至循环体外

***轻量级锁***

即乐观锁，先执行后补偿(CAS等)

***偏向锁***

偏表示偏心，即偏向第一个获取它的线程，如果在接下来的执行过程中，该锁没有被其他的线程获取，则持有该锁的线程将永远不再需要进行同步

## Java中的多线程概念

### 线程模型

* 内核线程：***KLT***，就是直接由操作系统内核支持的线程，由内核来完成线程切换，内核通过操作调度器来对线程进行调度，并负责将线程的任务映射到各个处理器上
* 轻量级线程：***LWP***，可以看做是操作系统对内核线程的一种封装，提供给用户使用的高级接口
* 用户线程：***UT***，广义来说，一个线程只要不是内核线程即可认为是用户线程；狭义来说，用户线程是指完全建立在用户空间的线程库上，系统内核不能感知用户线程的存在，用户线程的建立、同步、销毁完全在用户态完成而不需要在内核和用户态之间来回切换

![屏幕快照 2019-04-03 23.34.54](https://ws3.sinaimg.cn/large/006tKfTcgy1g1pvc1rky5j310e0iimzk.jpg)

### 线程状态

* 新建***New***：创建后尚未启动
* 运行***Runnable***：正在执行或等待CPU为其分配执行时间
* 无限期等待***Waiting***：等待被主动唤醒的线程，`Object.wait()`或`Thread.join()`或`LockSupport.park()`都会使线程进入无线等待
* 限期等待***Timed Waiting***：一定时间后被自动唤醒的线程，`Thread.sleep(time)`或`Object.wait(time)`或`Thread.join(time)`或`LockSupport.parkNanos(time)`或`LockSupport.parkUntil(time)`都会使线程进入限期等待
* 阻塞***Blocked***：等待获取到一个排他锁，当线程等待进入同步区域的时候将会进入这种状态
* 结束***Terminated***：已经终止的线程

> 在常见的工作场景中，需要定位性能问题时候可使用jstack工具结合线程状态进行判断

### 内存模型JMM

***操作系统中的内存模型***

> 基于高速缓存的存储交互很好地解决了处理器与外部IO的速度矛盾，但是也引入了更高的复杂度，如：缓存一致性；为了解决这个问题，处理器在访问缓存时都遵循一些协议，在读写时都要根据协议来进行操作，这类协议有MSI、MESI、MOSI、Synapse、Firefly等

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g1m8aaxq4pj311c0giq6i.jpg)

***Java内存模型***

> Java内存模型规定所有的变量都存储在主内存中，但每个对等线程的上下文中都包含对应线程所需使用变量的拷贝，线程无法直接读写主内存或其他线程工作内存中的变量，只能通过主内存进行数据交换。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g1m8mqevbej313m0gejv0.jpg)

> 线程与工作内存和主内存之间存在如下八种操作类型：`lock\unlock、read\load、use\assign、store\write`

- `lock`的对象为主内存变量，从设计上讲`lock`一个工作内存中的变量是没有意义的
- 同一个变量任意时刻只能被一个线程`lock`，但是可以被同一个线程执行N次`lock`操作；对变量执行N次`lock`操作后，只有执行对应的N次`unlock`才能将变量解锁
- 对一个变量执行`lock`后将会清空工作内存中该变量的值，在`use`之前需要重新执行`load\assign`操作初始化变量的值
- 执行`unlock`前必须保证已执行过`store`、`write`
- 不允许对一个没执行过`lock`的变量进行`unlock`，也不允许对其他线程`lock`的变量执行`unlock`
- `read\load`，`store\write`为成对操作，不允许单独出现，即不允许变量被`read`了但是不`load`，也不允许变量被`store`了但是不`write`
- 不允许`store\write`一个没有发生过`assign`的变量

![](https://ws3.sinaimg.cn/large/006tKfTcgy1g1m9k31aokj30ya0fe40t.jpg)

### volatile

在Java中`volatile`关键字有二点重要定义

* 此变量对所有线程均具有可见性
* 禁止指令重排序优化

#### 可见性

没有volatile修饰的变量在多线程环境下可能导致变量状态始终不同步，比如在如下的例子中，`ListenThread`将陷入死循环

```java
import net.sourceforge.groboutils.junit.v1.MultiThreadedTestRunner;
import net.sourceforge.groboutils.junit.v1.TestRunnable;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.lang.invoke.MethodHandles;

public class TestVolatileVisible {
    private static Logger logger = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

    private static int LOOPS = 5;

    //private volatile static int NUMBER = 0;
    private static int NUMBER = 0;

    @Test
    public void validateStatus() throws Throwable {
        TestRunnable[] threads = new TestRunnable[2];
        threads[0] = new ListenThread();
        threads[1] = new ChangeThread();

        MultiThreadedTestRunner runner = new MultiThreadedTestRunner(threads);
        runner.runTestRunnables();
    }

    public class ListenThread extends TestRunnable {
        @Override
        public void runTest() throws Throwable {
            while(NUMBER < LOOPS) {
                // remove this line to see difference
                //logger.info("NUMBER {} less than {}", NUMBER, LOOPS);
            }

            logger.info("shutdown");
        }
    }

    public class ChangeThread extends TestRunnable {
        @Override
        public void runTest() throws Throwable {
            while(NUMBER < LOOPS) {
                NUMBER++;

                try {
                    // remove this line to see difference
                    Thread.sleep(1);
                } catch (Exception ex) {

                }

                logger.info("NUMBER increase to {}", NUMBER);
            }
        }
    }
}
```

#### 有序性

CPU指令在执行的过程中会在不影响程序最终运行结果的前提下对代码执行顺序进行重排，来达到优化执行效率的目的，比如下的代码

```java
int	a = 1;
int b = 2;

a++;
b++;
```

可能会被最终优化为

```java
int a = 1;
a++;

int b = 2;
b++;
```

指令重排序在单线程执行环境下不会影响最终运行结果，但是在多线程环境可能会产生与预期不一致的情况

```java
//线程1:
context = loadContext();   //语句1
inited = true;             //语句2
 
//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```

在如上的示例中，如果在线程1中`inited = true`被重排序在`loadContext()`之前，将会导致线程2中判断失误进而导致`doSomethingwithconfig`方法执行异常；此时使用`volatile`来修饰`inited`会形成一道屏障指示，禁止编译器优化，`volatile`可指示禁止该语句的前部分重排序到该语句之后，同时禁止该语句的后部分重排序到该语句之前

----

> JVM中关于执行顺序定义了一个原则，称为`happens-before`原则或***先行发生***原则，即若操作A***先行发生于***B，则表示A的影响可以被B观察到，这里的影响包括修改变量值、发送消息、调用方法等。具体原则说明如下

- 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作***先行发生于***书写在后面的操作
- 锁定规则：一个`unlock`操作***先行发生于***后面对同一个锁的`lock`操作
- `volatile`变量规则：对一个变量的写操作***先行发生于***后面对这个变量的读操作，这里的后面指的是时间上的先后
- 传递规则：如果操作A***先行发生于***操作B，而操作B又***先行发生于***操作C，则可以得出操作A***先行发生于***操作C
- 线程启动规则：`Thread`对象的`start()`方法***先行发生于***此线程的每个一个动作
- 线程中断规则：对线程`interrupt()`方法的调用***先行发生于***被中断线程的代码检测到中断事件的发生，可以通过`Thread.interrupted()`方法检测到是否有中断发生

- 线程终结规则：线程中所有的操作都***先行发生于***线程的终止检测，我们可以通过`Thread.join()`方法结束、`Thread.isAlive()`的返回值手段检测到线程已经终止执行
- 对象终结规则：一个对象的初始化完成***先行发生于***他的`finalize()`方法的开始

#### 原子性

由Java内存模型和操作规则定义：Java中的基本原子性操作包括`read、load、use、assign、store、write`，当需要更大范围的原子性操作时可使用`lock`（对应字节码`monitorenter`）与`unlock`（对应字节码`monitorexit`）。

***volatile变量的可见性保证了其在不同线程中的安全性，但并不等同于volatile变量的任意操作也是***

```java
private volatile int race = 0;
public void increase() {
    race++;
}

@Test
public void volatileVar() {
    for(int i = 0; i < 10; i++) {
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                for(int j=0; j<10000;j++) {
                    increase();
                }
            }
        });
        
        t.start();
    }
    
    while(Thread.activeCount() > 1) Thread.yield();
    
    Assert.assertEquals(100000, race);
}
```

以上单元测试`volatileVar`将失败，究其原因在于`race++`并非原子性操作

```assembly
public void increase();
    Code:
       0: aload_0
       1: dup
       2: getfield      #3                  // Field race:I
       5: iconst_1
       6: iadd
       7: putfield      #3                  // Field race:I
      10: return
```

`volatile`只能保证在`getfield`时变量在不同线程中的一致性，但是在`iadd`之后，`putfield`之前，变量可能已经被其他线程修改，导致最终计算结果不一致

#### 关于volatile的归纳总结

* ***非`volatile`变量***无法保证其他线程的可见性，即线程A修改后线程B***可能***永远无法感知
* `volatile`变量不可等价于对变量的所有操作都是原子性的，比如`i++`，在这种情况必须使用`synchronized`关键字或其他同步方法
* `volatile`主要的适用场景为单线程写，多线程读，需要保证状态值的变更一定及时通知其他线程

### synchronized

在Java字节指令中`monitorenter`的使用对象必须为`reference`类型，对应的`synchronized`锁定的对象也必须为`reference`类型，`synchronized`一般有如下四种应用模式

| 修饰对象 | 目标对象 | 代码示例                                          |
| -------- | -------- | ------------------------------------------------- |
| 方法     | 实例对象 | `public synchronized doSomething(){ ... }`        |
| 方法     | 类对象   | `public static synchronized doSomething(){ ... }` |
| 代码块   | 类对象   | `synchronized(Some.class) { ... }`                |
| 代码块   | 实例对象 | `synchronized(lockObj){ ... }`                    |

## Java中的多线程工具

### ReentrantLock

***synchronized的不足***

- 只有默认的同步策略—无限期等待
- 同步阻塞的过程中无法被中断

***ReentrantLock的优势***

- 支持设置获取锁的超时时间，当超时后可以暂时执行其他任务节约计算资源
- 支持公平锁(公平锁即按照申请锁的先后顺序来依次获取锁，而非公平锁不保证这一点，当锁被释放后所有等待的线程都有可能获取到锁)
- 方法`lockInterruptibly()`支持在同步阻塞等待的过程中被中断

如下是一个简单示例

```java
try {
  lock.tryLock(1, TimeUnit.SECONDS);
} catch (InterruptedException ex) {
  // handler exception
} finally {
  // 思考这里是否有同步问题？
  if (lock.isLocked()) {
    lock.unlock();
  }
}
```

### Runnable与Thread

实现线程的最直接方式为实现`Runnable`接口并以`Thread`类进行驱动或继承`Thread`类并重写`run`方法

***Runnable与Thread分离设计的思想***

* `Runnable`的语义为执行工作具体内容的实体对象，`Thread`为单纯的线程运行对象

* 在抽象层面我们总是希望定义一个可执行的任务，这个任务包含需要执行的具体业务逻辑，并在类型上与其他任务区别开来，最后我们并不关心执行操作系统相关的线程细节。在线程的角度来看，线程的创建和管理是不与具体的任务相关联的，同时线程的创建代价可能会很高昂。将二者隔离设计既符合面向对象的思想也可以避免不必要的线程开销

***Thread中的异常处理***

默认情况下Thread的run方法导致的异常只能在线程内部处理，一般有以下二种方式处理

* 将`run()`方法内部代码用`try-catch`包裹

* 使用`Thread.unCaughtExceptionHandler`接口实现对`Thread`未捕获的异常的处理，其中可以使用`Thread.setDefaultUnCaughtExceptionHandler`来对默认的异常处理进行统一设置

```java
static class UncaughtExceptionHandler implements Thread.UncaughtExceptionHandler {
	@Override
	public void uncaughtException(Thread t, Throwable e) {
		logger.info("[uncaughtException] caught exception from thread {}, {}", t, e);
		}
}
```

```java
Thread.setDefaultUncaughtExceptionHandler(new UncaughtExceptionHandler());
```

***Thread中的退出处理***

默认情况下当一个线程的`run()`方法执行完毕后线程将会自动退出变为`Terminated`状态，在之后某个时间点被GC回收；对线程执行状态的控制可以通过以下几种方式

* 基于`while`循环 + `volatile`状态指示值

* 基于中断的异常处理`InterruptedException`
* 基于中断的状态判断`Thread.isInterrupted()`

> 注意：中断异常只有在执行了阻塞类的方法的时候才可以捕获到，如`sleep()`；`suspend()、resume()、stop()`因设计缺陷已被建议停止使用

***直接使用Runnable或Thread存在的弊端***

- 不便于有效合理的使用系统资源；无节制的开辟执行线程可能在单核服务器上造成过多上下文切换甚至假死，相反过于保守，在多核服务器上可能无法充分利用现有的计算资源
- 不便于将执行完任务的线程回收重利用；当`Thread`对应的`run`方法执行完毕后进程会进入`Terminated`状态并最终被GC，下次需要线程时又要重新创建线程资源
- 不便于操作退出；没有统一直观的入口对线程进行退出管理

> JDK中包含一个ThreadGroup对象，在初始化Thread对象时可使用；其主要用于构建线程关系树结构，包含组信息、组优先级、组ACL等，一般不会使用到，官方示例也较少。尤其在JDK5之后，其功能基本已被Executor替代。

| 方法名                  | 静态方法 | 语义                                                         | 是否需要锁 | 是否占用锁 |
| ----------------------- | -------- | ------------------------------------------------------------ | ---------- | ---------- |
| Thread.sleep            | 是       | 使当前线程休眠指定时间；既不释放锁也不让出CPU执行时间片      | 否         | 是         |
| Thread.yield            | 是       | 告知当前线程可让出当前CPU执行时间片；调度器可选择是否切换其他线程 | 否         | 是         |
| Object.wait             | 否       | 使锁对象等待被唤醒并***释放对应的锁***；可通过二种方式唤醒：`Thread.interrupted`或`Object.notify/Object.notifyAll` | 是         | 否         |
| Object.notify/notifyAll | 否       | 唤醒在等待的锁对象，notify表示由调度器唤醒一个，notifyAll表示由调度器唤醒所有 | 是         | 是         |

[示例代码 cooperation\ApplicationWaitSimple]()

### Callable与Executor

`Callable<T>`即带有返回值的任务，可以类比为C#中的`Task<T>`

***Executor中的退出***

* `ExecutorService.shutdown()`终止接收新的线程，并等待已有线程自然结束
* `ExecutorService.shutdownNow()`终止接收新的线程，并立即发送中断信号`interrupt`至所有线程
* `ExecutorService.submit()`获取`Future`对象，使用`Future.cancel()`对单个线程发送中断信号`interrupt`，即使是`Runnable`对象也可以调用submit方法

### Semaphore

前文提到的锁都是唯一锁，即任意时刻有且只有一个线程拥有锁；而Java中提供的`Semaphore`则为动态锁，其可实现任意时刻有且最多N个线程拥有锁，其中N可以随意设置并支持动态调整，在一些需要限制并发量的场景下有实际应用，示例代码：[ApplicationSemaphore]()

```java
public class ApplicationSemaphore {
    private static Logger logger = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

    public static void main(String[] args) {
        Pool<Carry> pool = new Pool<>(Carry.class, 3);

        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < 10; i++) {
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    while(true) {
                        Carry item = pool.checkOut();
                        try {
                            TimeUnit.SECONDS.sleep(1);
                        } catch (InterruptedException ex) {

                        }
                        pool.checkIn(item);
                    }
                }
            });
        }
    }

    static class Carry {

    }

    static class Pool<T> {
        private int size;
        private List<T> items;
        private boolean[] states;
        private Semaphore semaphore;

        public Pool(Class<T> clazz, int size) {
            this.size = size;
            semaphore = new Semaphore(size, true);

            items = new ArrayList<>(size);
            states = new boolean[size];
            for (int i = 0; i < size; i++) {
                try {
                    items.add(clazz.newInstance());
                } catch (Exception ex) {
                    logger.info("{}",ex);
                }
            }

            logger.info("items: {}, {}", items, items.size());
        }

        public void checkIn(T item) {
            int i = items.indexOf(item);
            if(i > -1 && !states[i]) {
                semaphore.release();
                logger.info("semaphore lock released...{}", semaphore.getQueueLength());
            }
        }

        public T checkOut() {
            try {
                logger.info("waiting for semaphore lock...");
                semaphore.acquire();

                for (int i = 0; i < size; i++) {
                    if (!states[i]) {
                        return items.get(i);
                    }
                }


            } catch (InterruptedException ex) {

            }
            return null;
        }
    }
}
```

## FAQ

> 为什么我在JUnit单元测试框架下运行多线程的结果与main方式不一致？

答：运行某个`@Test`方法观察到执行的java命令为

```
/Library/Java/JavaVirtualMachines/jdk-10.0.1.jdk/Contents/Home/bin/java -ea -Didea.test.cyclic.buffer.size=1048576 "-javaagent:/Applications/IntelliJ IDEA CE.app/Contents/lib/idea_rt.jar=55038:/Applications/IntelliJ IDEA CE.app/Contents/bin" -Dfile.encoding=UTF-8 -classpath "/Applications/IntelliJ IDEA CE.app/Contents/lib/idea_rt.jar:/Applications/IntelliJ IDEA CE.app/Contents/plugins/junit/lib/junit-rt.jar:/Applications/IntelliJ IDEA CE.app/Contents/plugins/junit/lib/junit5-rt.jar:/Users/joker/Desktop/work/sample/Java/concurrent/target/test-classes:/Users/joker/Desktop/work/sample/Java/concurrent/target/classes:/Users/joker/.m2/repository/junit/junit/4.12/junit-4.12.jar:/Users/joker/.m2/repository/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3.jar:/Users/joker/.m2/repository/org/slf4j/slf4j-api/1.7.25/slf4j-api-1.7.25.jar:/Users/joker/.m2/repository/org/slf4j/slf4j-simple/1.7.25/slf4j-simple-1.7.25.jar" com.intellij.rt.execution.junit.JUnitStarter -ideVersion5 -junit4 TestRunable,simple
```

> 运行的启动类为：`com.intellij.rt.execution.junit.JUnitStarter`

> 其对应的jar包为(MAC OS)：`/Applications/IntelliJ IDEA CE.app/Contents/plugins/junit/lib/junit-rt.jar`

在IDEA中添加对应junit-rt.jar包依赖可进行debug调试，依次调试发现其测试方法调用路径为

```java
com.intellij.rt.execution.junit.JUnitStarter
public static void main(String[] args) throws IOException
↓
private static int prepareStreamsAndStart(String[] args, String agentName, ArrayList listeners, String name)
↓
com.intellij.rt.execution.junit.IdeaTestRunner.Repeater
public static int startRunnerWithArgs(IdeaTestRunner testRunner, String[] args, ArrayList listeners, String name, int count, boolean sendTree)
↓
com.intellij.junit4.JUnit4IdeaTestRunner
public int startRunnerWithArgs(String[] args, String name, int count, boolean sendTree)
↓
org.junit.runner.JUnitCore
public Result run(Runner runner)
↓
org.junit.runners.ParentRunner<T>
public void run(final RunNotifier notifier)
↓
protected Statement childrenInvoker(final RunNotifier notifier)
↓
private void runChildren(final RunNotifier notifier)
↓
protected final void runLeaf(Statement statement, Description description,
            RunNotifier notifier)
↓
org.junit.internal.runners.model.ReflectiveCallable
public Object run() throws Throwable 
```

可追踪到其最后是通过反射来执行对应的`@Test`方法，在执行的过程中并没有对方法内部的异步线程进行特殊处理，在执行完毕后便立即返回了测试的执行状态(成功或失败)，最终将测试状态返回主函数并直接调用JVM退出函数`System.exit(exitCode);`

![屏幕快照 2019-03-30 12.34.10](https://ws2.sinaimg.cn/large/006tKfTcgy1g1kpvlv316j31oo0pkgua.jpg)

所以***测试方法内部异步线程***执行结果是无法直接通过JUnit测试框架观察到的，***无论是否为前台线程还是后台线程***

那么什么是JUnit框架中多线程测试的正确姿势？使用JUnit第三方扩展包 - [GroboUtils](https://community.oracle.com/docs/DOC-982943)

----

> 什么是原子性操作？原子性操作不需要进行同步控制这句话是否正确？试举例

答：原子性操作即不能被线程调度机制打断的操作，一旦一个原子性操作开始执行，那么它一定可以在发生线程上下文切换之前完成；

----

> 试按照先行发生原则分析如下代码，在A线程执行setValue后，B线程调用getValue的返回值是多少？

```java
private int value = 0;

public void setValue(int value) {
  this.value = value;
}

public void getValue() {
  return value;
}
```

答：不满足任意一条先行发生原则，所以无法确定

----

> 先行发生原则是否可以理解为时间上的先后顺序？先行发生原则中的程序次序原则是否与CPU指令重排行为相冲突？

答：先行发生原则描述的是逻辑关系而不是时间关系，所以二者没有必然联系；时间上操作优先的行为并不一定先行发生于时间上操作靠后的行为，如上例。

先行发生原则与CPU指令重排行为不冲突，程序次序原则保证在同一个线程内其执行是按代码先后顺序的，CPU在执行指令重排时会先进性依赖性分析，对于无依赖关系的指令进行重排对同一个线程来说结果一致，所以并无冲突

----

> 以下为Java代码中经典的DCL单例实现，试分析变量volatile关键字的作用，若去掉会有什么副作用？

```java
class Singleton{
    private volatile static Singleton instance = null;
     
    private Singleton() {
         
    }
     
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

答：JDK1.5 new Object()

----

> `ThreadLocal<T>`可以用来存储线程本地变量，其方法`get()`与`set()`是否需要同步？为什么？

答：不需要，ThreadLocal是本地线程专享变量，不存在共享冲突

----

> `Object.wait`与`Object.notify/notifyAll`的行为是否违反Java对同步锁的定义(任意时刻有且只有一个线程占有同一个锁)，为什么？

答：不违反，wait方法调用时会释放锁，notify所在的同步区执行完毕后会释放锁；在任意时刻都只有一个线程占用锁

---

> 试分析下面的代码是否存在死锁风险，为什么？

```java
// Thread 1
synchronized(lockerObject) {
  globalCondition = false;
  lockerObject.notify();
}

// Thread 2
while(globalCondition) {
  // Point A
  synchronized(lockerObject) {
    lockerObject.wait();
  }
}
```

答：存在，假设时刻T1执行到了Point A，之后切换到Thread 1中进行执行并触发了notify，而后回到Thread 2时，由于不再有线程会唤醒locker从而会造成Thread 2无限期wait造成死锁

----

> 假设需求场景为限速消费(比如报警通知，同类型短时间内重复多次会造成不必要的浪费)，需要实现一个内存延迟队列，生产者的数量和速度可变，消费者的数量可变，速度固定为V(可理解为任意二次消费间隔相同时间T)，设计思路如何？

答：DelayQueue<T> 

----

> 假设需求场景为优先消费(比如VIP通道，高等级会员的请求优先处理)，需要实现一个优先消费队列，生产者会生成一些不同优先级的元素，消费者消费时需按优先级从高到低逐个消费，设计思路如何？

答：PriorityBlockingQueue<T>

----

## 资源链接

[Oracle Community - GroboUtils](https://community.oracle.com/docs/DOC-982943)

[How to Create Volatile Problem](https://dzone.com/articles/java-volatile-keyword-0)

[Java Volatile Principle](http://tutorials.jenkov.com/java-concurrency/volatile.html)

[使用Volatile的DCL模式](https://segmentfault.com/a/1190000004559046)

[Close Encounters of The Java Memory Model Kind](https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/#pitfall-release-order-wrong)

[Java 编程思想 - 第四版](https://item.jd.com/10058164.html)

[深入理解Java虚拟机 - 第二版](https://item.jd.com/40201292727.html)

[Oracle Document - java.util.concurrent](https://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/package-summary.html)