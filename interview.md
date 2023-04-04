## JUC

#### 1、谈谈对volatile的理解

volatile是java虚拟机提供的**轻量级**同步机制（涉及JMM的可见性、原子性、有序性）

+ 保证可见性


​		线程修改主内存数据后对其他线程可见

+ 不保证原子性

  在写入主内存时被其他线程抢先写入，再写入时会把之前数值覆盖，解决方案：使用原子类Atomic...（CAS）

+ 禁止指令重排

  多线程情况下在不影响结果的时候会进行指令重排，而volatile会禁止指令重排（内存        屏障）

volatile使用场景：懒汉单例模式，双重检查（DCL） + volatile修饰（对象初始化时可能指令重排）

#### 2、CAS知道吗

+ compareAndSwap，比较并交换，真实值（主内存）和期望值（工作内存）一样时才可以修改。  
+ 底层原理：自旋 + unsafe类，CAS是一条CPU的原子指令。  
+ 缺点：1、失败时会一直尝试，给CPU带来很大开销；2、只能保证一个共享变量的原子性；3、ABA问题。  

#### 3、原子类的ABA问题

+ 描述：AB两个线程同时拿到一个值，A线程更新B线程可能更新多次，并且最后一次将值改回原来的值，A再更新时可以成功，但是期间值已经被修改过了。  
+ 解决方案：原子引用AtomicStampedReference + 版本号

#### 4、集合类不安全问题ArrayList

+ 故障现象：java.util.ConcurrentModificattionException
+ 导致原因：多线程add()导致的**并发修改异常**

+ 解决方案：1、Vector()（不推荐...）；2、Collections.synchronizedList()；3、写时复制CopyOnWriteArrayList()，源码如下：

  ```java
  public boolean add(E e) {
          final ReentrantLock lock = this.lock;
          lock.lock();
          try {
              Object[] elements = getArray();
              int len = elements.length;
              Object[] newElements = Arrays.copyOf(elements, len + 1);
              newElements[len] = e;
              setArray(newElements);
              return true;
          } finally {
              lock.unlock();
          }
      }
  ```

#### 5、集合类不安全问题Set

同上，使用CopyOnWriteArraySet()

#### 6、集合类不安全问题Map

同上，使用ConcurrentHashMap()

#### 7、对公平锁、非公平锁、可重入锁、自旋锁的理解，手写自旋锁

+ 公平锁/非公平锁：ReentrantLock()默认为非公平锁，高并发情况下会造成优先级反转和锁饥饿问题，传入参数true时为公平锁，按照线程申请锁顺序排队，非公平锁优点在于吞吐量比公平锁大。

+ 可重入锁（递归锁）：线程可以进入任何一个它已经拥有的锁所同步着的代码块。（ReentrantLock、Synchronized）例如对象锁，对象中两个加锁方法AB，方法A调用方法B，可以直接进入。可重入锁最大的作用是避免死锁。

+ 自旋锁：尝试获取锁的线程不会立即阻塞，而是**采用循环的方式去尝试获取锁**，这样的好处就是减少线程上下文切换的消耗，缺点是循环会消耗CPU。

  ```java
  // 手写自旋锁
  public class SpinLockDemo {
  
      AtomicReference<Thread> atomicReference = new AtomicReference<>();
  
      public void lock() {
          Thread currentThread = Thread.currentThread();
          System.out.println(currentThread.getName() + "线程进入");
          while (!atomicReference.compareAndSet(null, currentThread)) {
              currentThread = Thread.currentThread();
          }
      }
  
      public void unlock() {
          Thread currentThread = Thread.currentThread();
          atomicReference.compareAndSet(currentThread, null);
          System.out.println(currentThread.getName() + "线程解锁");
      }
  
      public static void main(String[] args) {
          SpinLockDemo spinLockDemo = new SpinLockDemo();
          new Thread(() -> {
              spinLockDemo.lock();
              try {
                  Thread.sleep(5000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              } finally {
                  spinLockDemo.unlock();
              }
          }, "A").start();
          new Thread(() -> {
              spinLockDemo.lock();
              try {
                  Thread.sleep(1000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              } finally {
                  spinLockDemo.unlock();
              }
          }, "B").start();
      }
  }
  ```

+ 独占锁/共享锁：独占锁是指该锁一次只能被一个线程所持有，ReentrantLock和Synchronized都是独占锁；共享锁是指该锁可以被多个线程所持有，ReentrantReadWriteLock的读锁就是共享锁，写锁是独占锁，保证了并发读的高效，而读写、写读、写写是互斥的。

#### 8、CountDownLatch、CyclicBarrier和Semaphore使用过吗

+ CountDownLatch：例下班，最后一个人关门

  ```java
  public class CountDownLatchDemo {
  
      public static void main(String[] args) throws InterruptedException {
          CountDownLatch countDownLatch = new CountDownLatch(3);
          for (int i = 0; i < 3; i++) {
              new Thread(() -> {
                  try {
                      System.out.println(Thread.currentThread().getName() + "线程开始执行");
                      Thread.sleep(1000);
                      System.out.println(Thread.currentThread().getName() + "线程执行完毕");
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  } finally {
                      countDownLatch.countDown();
                  }
              }, String.valueOf(i)).start();
          }
          countDownLatch.await();
          System.out.println("所有线程执行完毕");
      }
  
  }
  ```

+ CyclicBarrier：例开会，最后一个人到了才开始

  ```java
  public class CyclicBarrierDemo {
  
      public static void main(String[] args) {
          CyclicBarrier cyclicBarrier = new CyclicBarrier(3, () -> {
              System.out.println("所有线程执行完毕");
          });
          for (int i = 0; i < 3; i++) {
              new Thread(() -> {
                  try {
                      Thread.sleep(1000);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
                  System.out.println(Thread.currentThread().getName() + "线程执行完毕");
                  try {
                      cyclicBarrier.await();
                  } catch (InterruptedException | BrokenBarrierException e) {
                      e.printStackTrace();
                  }
              }, String.valueOf(i)).start();
          }
      }
  
  }
  ```

+ Semaphore：例抢车位

  ```java
  public class SemaphoreDemo {
  
      public static void main(String[] args) {
          Semaphore semaphore = new Semaphore(2);
          for (int i = 0; i < 4; i++) {
              new Thread(() -> {
                  try {
                      semaphore.acquire();
                      System.out.println(Thread.currentThread().getName() + "线程抢到车位");
                      Thread.sleep(1000);
                      System.out.println(Thread.currentThread().getName() + "线程离开");
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  } finally {
                      semaphore.release();
                  }
              }, String.valueOf(i)).start();
          }
      }
  
  }
  ```

#### 9、阻塞队列知道吗

当阻塞队列为空时从队列获取元素的操作被阻塞；当阻塞队列满时往队列添加元素的操作被阻塞。

+ ArrayBlockingQueue：数组结构组成的**有界**阻塞队列

  | 方法类型 | 抛出异常  |  特殊值  |  阻塞  |        超时        |
  | :------: | :-------: | :------: | :----: | :----------------: |
  |   插入   |  add(e)   | offer(e) | put(e) | offer(e,time,unit) |
  |   移除   | remove()  |  poll()  | take() |  poll(time,unit)   |
  |   检查   | element() |  peek()  | 不可用 |       不可用       |

+ SynchronousQueue：不存储元素的阻塞队列，即单个元素的队列

使用场景：数字0，两个线程交替加1减1，执行5次

+ Lock版

  ```java
  /**
   * @description 数字0，两个线程交替加1减1，执行5次
   * 1  线程操作资源类
   * 2  判断干活通知
   * 3  防止虚假唤醒
   */
  public class LockDemo {
  
      public static void main(String[] args) {
          MyData myData = new MyData();
          new Thread(() -> {
              for (int j = 0; j < 5; j++) {
                  myData.add();
              }
          }, "A").start();
          new Thread(() -> {
              for (int j = 0; j < 5; j++) {
                  myData.sub();
              }
          }, "B").start();
      }
  
  } 
  
  class MyData {
      private int num = 0;
      private final Lock lock = new ReentrantLock();
      Condition condition = lock.newCondition();
  
      public void add() {
          lock.lock();
          try {
              // while防止虚假唤醒，if不行
              while (num != 0) {
                  condition.await();
              }
              // 干活
              num++;
              System.out.println(Thread.currentThread().getName() + " " + num);
              // 通知唤醒
              condition.signalAll();
          } catch (InterruptedException e) {
              e.printStackTrace();
          } finally {
              lock.unlock();
          }
      }
  
      public void sub() {
          lock.lock();
          try {
              while (num == 0) {
                  condition.await();
              }
              num--;
              System.out.println(Thread.currentThread().getName() + " " + num);
              condition.signalAll();
          } catch (InterruptedException e) {
              e.printStackTrace();
          } finally {
              lock.unlock();
          }
      }
  
  }
  ```

+ 阻塞队列版：

  ```java
  public class BlockingQueueDemo {
  
      public static void main(String[] args) throws InterruptedException {
          BlockingQueueDemoData blockingQueueDemoData = new BlockingQueueDemoData(new ArrayBlockingQueue<>(10));
          new Thread(() -> {
              try {
                  blockingQueueDemoData.produce();
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }, "A").start();
          new Thread(() -> {
              try {
                  blockingQueueDemoData.consume();
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }, "B").start();
  
          Thread.sleep(5000);
          System.out.println("------stop------");
          blockingQueueDemoData.stop();
      }
  
  }
  
  class BlockingQueueDemoData {
      private volatile boolean FLAG = true; // 默认开启，进行生产+消防
      private final AtomicInteger atomicInteger = new AtomicInteger();
  
      BlockingQueue<String> blockingQueue;
  
      public BlockingQueueDemoData(BlockingQueue<String> blockingQueue) {
          this.blockingQueue = blockingQueue;
          System.out.println(blockingQueue.getClass().getName());
      }
  
      public void produce() throws InterruptedException {
          String data;
          boolean result;
          while (FLAG) {
              data = atomicInteger.incrementAndGet() + "";
              result = blockingQueue.offer(data, 2, TimeUnit.SECONDS);
              if (result) {
                  System.out.println(Thread.currentThread().getName() + "插入队列成功：" + data);
              } else {
                  System.out.println(Thread.currentThread().getName() + "插入队列失败：" + data);
              }
              Thread.sleep(1000);
          }
          System.out.println(Thread.currentThread().getName() + "生产者结束");
      }
  
      public void consume() throws InterruptedException {
          String data;
          while (FLAG) {
              data = blockingQueue.poll(2, TimeUnit.SECONDS);
              if (data != null) {
                  System.out.println(Thread.currentThread().getName() + "消费队列成功：" + data);
              } else {
                  FLAG = false;
                  System.out.println(Thread.currentThread().getName() + "消费队列失败 退出");
                  return;
              }
          }
      }
  
      public void stop() {
          FLAG = false;
      }
  
  }
  ```

  

#### 10、synchronized和lock有什么区别，lock有什么优点（精确唤醒）

+ 原始构成：  

  synchronized是关键字，属于JVM层面，底层通过monitor对象来完成，其实wait/notify等方法也依赖于monitor对象，只有在同步块或者方法中才能调用monitorenter、monitorexit  。

  lock是具体类（juc.locks.Lock）属于api层面的锁。

+ 使用方法：

  synchronized不需要用户去手动释放锁，当synchronized代码执行完后系统会自动让线程释放对所的占用  。

  ReentrantLock则需要用户手动释放锁，不释放可能导致死锁现象，需要lock()和unlock()方法配合try/finally语句块来完成。

+ 等待是否可中断

  synchronized不可以中断，除非抛出异常或者正常运行完成。

  ReentrantLock可以中断，1、超时方法tryLock(time,unit);2、lockInterruptibly()放代码块中，调用interrupt()方法可中断。

+ 加锁是否公平

  synchronized非公平锁。

  ReentrantLock两者都可以，默认非公平锁，构造方法可以传入true实现公平锁。

+ 锁绑定多个条件Condition

  synchronized没有。

  ReentrantLock用来实现分组唤醒需要唤醒的线程，可以**精确唤醒**，synchronized则是随机唤醒一个或者全部唤醒。

  ```java
  /**
   * @description lock精确唤醒
   * ABC三个线程，A先打印2次，B再3次，C再4次，重复5次
   */
  public class LockConditionDemo {
  
      public static void main(String[] args) {
          MyConditionDemoData data = new MyConditionDemoData();
          new Thread(() -> {
              for (int i = 0; i < 5; i++) {
                  data.runA();
              }
          }, "A").start();
          new Thread(() -> {
              for (int i = 0; i < 5; i++) {
                  data.runB();
              }
          }, "B").start();
          new Thread(() -> {
              for (int i = 0; i < 5; i++) {
                  data.runC();
              }
          }, "C").start();
      }
  }
  
  class MyConditionDemoData {
      private int number = 1;
      private final Lock lock = new ReentrantLock();
      Condition conditionA = lock.newCondition();
      Condition conditionB = lock.newCondition();
      Condition conditionC = lock.newCondition();
  
      public void runA() {
          lock.lock();
          try {
              while (number != 1) {
                  conditionA.await();
              }
              for (int i = 1; i <= 2; i++) {
                  System.out.println(Thread.currentThread().getName() + " " + i);
              }
              number = 2;
              conditionB.signal();
          } catch (InterruptedException ignored) {
          } finally {
              lock.unlock();
          }
      }
  
      public void runB() {
          lock.lock();
          try {
              while (number != 2) {
                  conditionB.await();
              }
              for (int i = 1; i <= 3; i++) {
                  System.out.println(Thread.currentThread().getName() + " " + i);
              }
              number = 3;
              conditionC.signal();
          } catch (InterruptedException ignored) {
          } finally {
              lock.unlock();
          }
      }
  
      public void runC() {
          lock.lock();
          try {
              while (number != 3) {
                  conditionC.await();
              }
              for (int i = 1; i <= 4; i++) {
                  System.out.println(Thread.currentThread().getName() + " " + i);
              }
              number = 1;
              conditionA.signal();
          } catch (InterruptedException ignored) {
          } finally {
              lock.unlock();
          }
      }
  
  }
  ```

#### 11、线程池的使用和优势

线程池多的工作主要是控制运行的线程数量，处理过程中将任务放到队列，然后在线程创建后启动这些任务，如果线程数量超过了最大数量，超出数量的线程排队等候，等其他线程执行完毕，再从队列中取出任务来执行。

主要特点：线程复用，控制最大并发数，管理线程。

+ 第一：降低资源消耗，通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
+ 第二：提高响应速度，当任务到达时，任务可以不需要等到线程创建就能立即执行。
+ 第三：提高线程的可管理性，线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

#### 12、Callable接口

有返回值，可以抛出异常，通过适配器模式FutureTask，通过get()获取返回值，尽量放最后防止线程阻塞。

#### 13、线程池的3种常用方式

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        // 一池5个处理线程
        // ExecutorService executor = Executors.newFixedThreadPool(5);
        // 一池1个处理线程
        // ExecutorService executor = Executors.newSingleThreadExecutor();
        // 一池n个处理线程
        ExecutorService executor = Executors.newCachedThreadPool();

        // 模拟10个用户办理业务
        try {
            for (int i = 0; i < 10; i++) {
                executor.execute(() -> System.out.println(Thread.currentThread().getName() + "办理业务"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            executor.shutdown();
        }
    }

}
```

#### 14、线程池7大参数

+ corePoolSize：线程池中的常驻核心线程数
+ maximumPoolSize：线程池中能够容纳同时执行的最大线程数，>=1
+ keepAliveTime：多余空闲线程的存活时间（线程数超过corePoolSize且空间时间达到keepAliveTime时，多余线程被销毁直到线程数量等于corePoolSize为止）
+ unit：keepAliveTime的单位
+ workQueue：任务队列
+ threadFactory：线程工厂
+ handler：拒绝策略

#### 15、线程池底层工作原理

+ 在创建了线程池后，等待提交过来的任务请求。
+ 当调用execute()方法添加一个请求任务时，线程池会做一下判断：
  + 如果正在运行的线程数小于corePoolSize，则马上运行这个任务。
  + 如果正在运行的线程数大于等于corePoolSize，则将任务放入队列。
  + 如果任务队列满了且正在运行的线程数量小于maximumPoolSize，则**创建非核心线程，且立即运行当前任务**。
  + 如果任务队列满了且正在运行的线程数量大于等于maximumPoolSize，则执行线程池的拒绝策略。
+ 当一个线程完成任务时，会从队列中获取写一个任务执行。
+ 当一个线程无事可做超过keepAliveTime时，线程池会做出相应判断：
  + 如果当前线程数量大于corePoolSize，则这个线程会被销毁。
  + 当线程池中所有任务完成后最终线程数量会保持到corePoolSize。

#### 16、线程池的4种拒绝策略

+ AbortPolicy(默认)：直接抛出RejectedExecutionException异常阻止系统正常运行。
+ CallerRunsPolicy：“调用者运行”一种调节机制，该策略既不会抛弃任务，也不会抛出异常，而是将某些任务回退到调用者，从而降低新任务的流量。
+ DiscardOldestPolicy：抛弃队列中等待最久的任务，然后把当前任务加入到队列中尝试再次提交当前任务。
+ DiscardPolicy：直接丢弃任务，不予任何处理也不抛出异常。如果允许任务丢失，这是最好的一种方案。

工作中用哪一个？          答：一个都不用......

#### 17、手写线程池

```java
public class MyThreadPoolExecutorDemo {

    public static void main(String[] args) {

        ExecutorService executor = new ThreadPoolExecutor(
                2,
                5,
                1L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(3),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.DiscardPolicy());
        try {
            for (int i = 0; i < 10; i++) {
                executor.execute(() -> System.out.println(Thread.currentThread().getName() + " thread running"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            executor.shutdown();
        }
    }

}
```

#### 18、线程池合理配置线程数

获取CPU核数：

```java
Runtime.getRuntime().availableProcessors()
```

CPU密集型：配置尽可能少的线程数量

一般公式：CPU核数 + 1个线程的线程池

IO密集型：

+ 由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程。

  参考公式：CPU核数 * 2

+ 任务需要大量的IO，即大量的阻塞。在单线程上运行IO密集型的任务会导致浪费大量的CPU运算能力在等待。所以使用多线程可以大大的加速程序运行，即使在单核CPU上，这种加速主要是利用了被浪费掉的阻塞时间。

  参考公式：CPU核数 / (1 - 阻塞系数)              阻塞系数在0.8~0.9之间

#### 19、死锁编码及定位分析

产生原因：两人都持有枪，互相让对方先放下，一直僵持。

```java
public class DeadLockDemo {

    public static void main(String[] args) {
        Dead dead1 = new Dead();
        new Thread(dead1::m1, "t1").start();
        new Thread(dead1::m2, "t2").start();
    }

}

class Dead {
    Lock lock1 = new ReentrantLock();
    Lock lock2 = new ReentrantLock();

    public void m1() {
        lock1.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "线程持有lock1中");
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getName() + "线程尝试获取lock2");
            lock2.lock();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock1.unlock();
        }
    }

    public void m2() {
        lock2.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "线程持有lock2中");
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getName() + "线程尝试获取lock1");
            lock1.lock();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock2.unlock();
        }
    }
}
```

+ jps -l定位进程号    jstack 进程号
+ jconsole 线程 死锁检查

## JVM + GC

#### 1、谈谈你对GCRoots的理解

用于判断一个对象是否可以被回收，枚举根节点做可达性分析（跟搜索路径）

+ 虚拟机栈（栈帧中的局部变量区，也叫局部变量表）中引用的对象
+ 方法区中的类静态属性引用的对象。
+ 方法区中常量引用的对象。
+ 本地方法栈中JNI（Native方法）引用的对象

#### 2、JVM的标配参数和X参数

+ 标配参数：-version、-help、java -showversion

+ X参数（了解）：-Xint（解释执行）、-Xcomp（第一次使用就编译成本地代码）、-Xmixed(混合模式)

+ XX参数（重点）

  查看是否开启某参数：jinfo -flag xx 进程号 / jinfo -flags 进程号

  + Boolean类型

    公式：-XX:+或者-某个属性     +表示开启；-表示关闭

  + kv设置类型

    公式：-XX:属性key=属性值value

    -Xms等价于-XX:InitialHeapSize、-Xmx等价于-XX:MaxHeapSize

  + 查看JVM初始参数：java -XX:+PrintFlagsInitial

  + 查看JVM修改后的参数：java -XX:+PrintFlagsFinal

    区别“=”和“:=”   后者表示参数修改过

  + 查看默认垃圾回收器：-XX:+PrintCommandLineFlags

#### 3、平时用过的JVM基本配置参数有哪些

+ -Xms（-XX:InitialHeapSize）：初始大小内存，默认物理内存1/64

+ -Xmx（-XX:MaxHeapSize）：最大分配内存，默认物理每次1/4

+ -Xss（-XX:ThreadStackSize）：单个线程栈的大小，不同平台默认值不同（516~1024）

+ -Xmn：设置新生代大小

+ -XX:MetaSpaceSize：设置元空间大小

+ -XX:+PrintGCDetails：输出详细GC日志信息

  ![image-20230331122011936](img\image-20230331122011936.png)

+ -XX:SurvivorRatio：设置新生代eden和S0/S1的比例，默认8:1:1，值为eden区
+ -XX:NewRatio：设置新生代和老年代在堆结构的比例，默认1:2，值为老年代
+ -XX:MaxTenuringThreshold：设置垃圾最大年龄（0-15）
+ tomcat调优参数...

#### 4、强、软、弱、虚引用分别是什么

+ 强引用Reference：即使OOM也不会被回收
+ 软引用SoftReference：内存不足时会被回收
+ 弱引用WeakReference：GC时一定会被回收
+ 虚引用PhantomReference：get()方法返回总是null，不会决定对象的生命周期，必须和引用队列联合使用

##### 软引用和弱引用的使用场景

​		读取大量图片，如果每次读取图片都从硬盘读取会严重影响性能，如果一次性全部加载到内存可能造成内存溢出。

​		设计思路：用一个HaspMap来保存图片的路径和相应图片对象关联的软引用之间的映射关系，在内存不足时，JVM会自动回收这些缓存图片对象所占的空间，从而有效避免OOM问题。

##### 谈谈WeakHashMap

​		当key置空时，GC会移除该entry。

##### 虚引用和引用队列

对象被回收前会先被放到引用队列中

```java
public class PhantomReferenceDemo {

    public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();
        ReferenceQueue<Object> refQueue = new ReferenceQueue<>();
        PhantomReference<Object> phantomReference = new PhantomReference<>(obj, refQueue);

        System.out.println(obj);
        System.out.println(phantomReference.get());
        System.out.println(refQueue.poll());
        System.out.println("--------------------------------");
        obj = null;
        System.gc();
        Thread.sleep(500);
        System.out.println(obj);
        System.out.println(phantomReference.get());
        System.out.println(refQueue.poll());
        /**
         * 输出
         * java.lang.Object@154617c
         * null
         * null
         * --------------------------------
         * null
         * null
         * java.lang.ref.PhantomReference@a14482
         */
    }

}
```

#### 5、谈谈对OOM的认识

+ java.lang.StackOverflow**Error**：栈溢出

  ```java
  public class StackOverflowErrorDemo {
  
      public static void main(String[] args) {
          stackOverflowError();
          // Exception in thread "main" java.lang.StackOverflowError
      }
  
      private static void stackOverflowError(){
          stackOverflowError();
      }
  
  }
  ```

+ java.lang.OutOfMemory**Error**:Java heap space：内存溢出

  ```java
  public class JavaHeapSpaceDemo {
  
      public static void main(String[] args) {
          String str = "start";
          while (true) {
              str += str + new Random().nextInt(123456789) + new Random().nextInt(987654321);
              str.intern();
          }
          // Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
      }
  
  }
  ```

+ java.lang.OutOfMemory**Error**:GC overhead limit exceeded：大量时间都在GC，但是效果不明显。

  ```java
  // 测试发现是java.lang.OutOfMemoryError: Java heap space......
  // -Xms10m -Xmx10m -XX:+PrintGCDetails -XX:MaxDirectMemorySize=5m
  public class GCOverheadDemo {
  
      public static void main(String[] args) {
          int count = 0;
          List<String> list = new ArrayList<>();
          try {
              while(true){
                  list.add(String.valueOf(++count).intern());
              }
          } catch (Throwable e) {
              System.out.println("count:" + count);
              e.printStackTrace();
              throw e;
          }
      }
  
  }
  ```

+ java.lang.OutOfMemory**Error**:Direct buffer memory：本地内存不足（不属于GC管辖，DirectByteBuffer对象们不会被回收，NIO）

  ```java
  // -Xms10m -Xmx10m -XX:+PrintGCDetails -XX:MaxDirectMemorySize=5m
  public class DirectBufferMemoryDemo {
  
      public static void main(String[] args) throws InterruptedException {
          System.out.println("配置的最大本地内存：" + VM.maxDirectMemory() / 1024 / 1024);
          Thread.sleep(3000);
          ByteBuffer.allocateDirect(6 * 1024 * 1024);
          // Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
      }
  
  }
  ```

+ java.lang.OutOfMemory**Error**:unable to create new native thread：无法再创建线程

  原因：进程创建线程过多，超过系统承载极限；服务器不允许程序创建那么多线程，linux默认单个进程可以创建1024个。

  解决方案：降低应用创建线程数；配置服务器限制。

  ```jav
  public class UnableToCreateNewNativeThreadDemo {
  
      public static void main(String[] args) {
          for (int i = 0; ; i++) {
              System.out.println(i);
              new Thread(() -> {
                  try {
                      Thread.sleep(Integer.MAX_VALUE);
                  } catch (InterruptedException exception) {
                      exception.printStackTrace();
                  }
              }, "" + i).start();
          }
          // Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
      }
  
  }
  ```

+ java.lang.OutOfMemory**Error**:MetaSpace：元空间溢出

  Java8之后使用Metaspace替代永久代，Metaspace是方法区在Hotspot中的实现，它与持久代最大的区别在于：Metaspace并不在虚拟机内存中而是使用的本地内存。存放信息如下：

  + 虚拟机加载的类信息
  + 常量池
  + 静态变量
  + 即时编译后的代码

  ```java
  // -XX:MetaspaceSize=8m -XX:MaxMetaspaceSize=8m
  public class MetaspaceOOMDemo {
  
      static class OOM {}
  
      public static void main(String[] args) {
          int i = 0;
          try {
              while (true) {
                  i++;
                  Enhancer enhancer = new Enhancer();
                  enhancer.setSuperclass(OOM.class);
                  enhancer.setUseCache(false);
                  enhancer.setCallback((MethodInterceptor) (o, method, objects, methodProxy) -> methodProxy.invokeSuper(o, objects));
                  enhancer.create();
              }
          } catch (Throwable e) {
              System.out.println(i);
              e.printStackTrace();
          }
          // java.lang.OutOfMemoryError: Metaspace
      }
  
  }
  ```

#### 6、GC垃圾回收算法和垃圾回收器的关系

**GC算法**：引用计数法、复制算法、标记清除、标记整理

垃圾回收器是算法的具体实现

##### 垃圾回收的四种方式：

+ Serial串行：单个垃圾回收器，暂停用户线程
+ Parallel并行：多个垃圾回收器并行，暂停用户线程
+ CMS(ConcMarkSweep)并发：用户线程和垃圾回收线程同时执行
+ G1

#### 7、查看、配置垃圾收集器

**查看**：java -XX:+PrintCommandLineFlags -version

**七种垃圾收集器**：UseSerialGC、UseParallelGC（默认）、UseConcMarkSweepGC、UseParNewGC、UseParallelOldGC、UseG1GC、还有一种弃用了UseSerialOldGC。

![image-20230331164235279](img\image-20230331164235279.png)

![image-20230331153920244](img\image-20230331153920244.png)

**组合的选择**：

+ 单CPU或小内存，单机程序

  UseSerialGC

+ 多CPU，需要最大吞吐量，如后台计算型应用

  UseParallelGC 或者 UseParallelOldGC

+ 多CPU，追求低停顿时间，需快速响应如互联网应用

  UseConcMarkSweepGC 或者 UseParNewGC

#### 8、G1垃圾收集器

主要改变Eden，Survivor和Tenured等内存区域不再是连续的了，而是变成了一个个大小一样的region，每个region从1M到32M不等。一个region有可能属于Eden，Survivor或者Tenured内存区域。

**回收步骤**：

针对Eden区进行收集，Eden区耗尽后会被触发，主要是小区域收集 + 形成连续的内存块，避免内存碎片。

+ Eden区的数据移动到Survivor区，假如出现Survivor区空间不够，Eden区数据会部分晋升到Old区。
+ Survivor区的数据移动到新的Survivor区，部分数据晋升到Old区。
+ 最后Eden区收拾干净了，GC结束，用户的应用程序继续执行。

![image-20230331170833110](img\image-20230331170833110.png)

![image-20230331170909947](img\image-20230331170909947.png)

**与CMS相比的优势**：

+ 不会产生内存碎片
+ 可以精确控制停顿

#### 9、JVMGC + SpringBoot微服务的生产部署和优化调参

+ IDEA微服务开发

+ Maven package

+ 启动微服务，同时配置JVM/GC的调优参数

  + 内部edit configuration

  + 外部（重点）

    公式：java    -server   jvm各种参数   -jar   *.jar/war包

    例：java -server -Xms1024m -Xmx1024m -XX:+UseG1GC -jar test.jar

#### 10、举例栈溢出的情况？（StackOverflowError）

当栈不足以添加栈帧时，如深度较大的递归，可以通过-Xss设置栈大小。

#### 11、调整栈大小就能保证不出现栈溢出吗？

不能，只能延迟栈溢出的时间。

#### 12、分配的栈内存越大越好吗？

不是，栈内存大了意味着其他的内存小了，比如堆内存，会出现OOM。

#### 13、垃圾回收是否会涉及到虚拟机栈？

不会。

#### 14、方法中定义的局部变量是否线程安全

+ 逃逸的局部变量不一定线程安全。
+ 不逃逸的局部变量线程安全。

#### 15、为什么使用PC寄存器记录当前线程的执行地址？

因为CPU需要不停的切换各个线程，这时候切换回来以后需要知道接着从哪里继续执行。JVM的字节码解释器就需要通过改变PC寄存器的值来明确下一条应该执行什么样的字节码指令。

![第04章_PC寄存器](img\第04章_PC寄存器.jpg)

## Linux

#### 1、查看整机系统性能：top

![image-20230331174046905](img\image-20230331174046905.png)

+ load average：系统负载均衡，分别代表1min、5min、15min平均负载值

  如果（a + b + c）/ 3 > 0.6 说明系统负担压力大。

也可使用精简版命令uptime：

![image-20230331174640795](img\image-20230331174640795.png)

#### 2、查看CPU：vmstat

![image-20230331175031558](img\image-20230331175031558.png)

+ procs
  + r：运行和等待CPU时间片的进程数，原则上1核的CPU的运行队列不要超过2，整个系统的运行队列不能超过总核数的2倍。
  + b：等待资源的进程数，比如正在等待磁盘I/O，网络I/O等。
+ cpu
  + us：用户进程消耗CPU时间百分比，us值高，用户进程消耗CPU时间多，如果长期大于50%，优化程序。
  + sy：内核进程消耗的CPU时间百分比。
  + us + sy参考值为80%，大于时说明可能存在CPU不足。
  + id：处于空闲的CPU百分比。
  + wa：系统等待IO的CPU时间百分比。
  + st：来自于一个虚拟机偷取的CPU时间的百分比。

查看所有CPU核信息：mpstat -P ALL 2

每个进程使用CPU的用量分解信息：ps -ef | grep java   +    pidstat -u 1 -p 进程号

#### 3、查看内存：free -m

![image-20230331180641141](img\image-20230331180641141.png)

20% < 应用程序可用内存/系统物理内存 < 70% 说明内存够用

每个进程使用内存的用量分解信息：pidstat -p 进程号 -r 采样间隔秒数

#### 4、硬盘：df -h

![image-20230331181204099](img\image-20230331181204099.png)

#### 5、磁盘IO：iostat -xdk 采样间隔秒数 采样次数

![image-20230331181339228](img\image-20230331181339228.png)

![image-20230331181431474](img\image-20230331181431474.png)

#### 6、网络IO：ifstat

![image-20230331181903781](img\image-20230331181903781.png)

#### 7、生产环境出现CPU占用过高，谈谈分析思路和定位

+ top找出占比高的
+ ps -ef 或者 jps 进一步定位程序
+ 定位到具体线程：ps -mp 进程 -o THREAD,tid,time
  + -m 显示所有线程
  + -p pid进程使用CPU的时间
  + -o 该参数后是用户自定义格式

+ 将需要的线程ID转换为16进制格式（英文小写格式）：printf "%x\n" 线程ID
+ jstack 进程ID | grep tid(16进制线程ID小写英文) -A60
