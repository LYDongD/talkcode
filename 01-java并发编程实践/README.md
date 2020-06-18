## java 并发编程实践

#### [01 | 可见性、原子性和有序性问题：并发编程Bug的源头](https://time.geekbang.org/column/article/83682)

> 笔记

* 并发安全问题通常是由什么由什么引起的？-> 最终来看是数据结果和预期不一致的问题
    * 可见性 -> 一个线程对另外一个线程的修改是否可见
        * 多核cpu缓存(L1/L2)不一致引起线程不可见
    * 原子性 -> 单个或多个操作的执行不可中断，要么都成功，要么都失败
        * 有的指令不是原子的, 执行过程可以发生线程切换
        * 线程上下文切换导致cpu寄存器指令不一致
    * 有序性 -> 程序按代码顺序执行
        * 编译器优化，编译期在不违法JMM（happens-before) 的原则下进行重排
        * cpu优化，提高cpu利用率
        * 指令重排后，可能引发例如线程拿到未初始化对象之类的问题
            * 如何查看java程序字节码？
                * javap -v xxx.class

* 什么情况下需要注意并发安全问题？
    * 共享变量
    * 发生check-and-do操作
    * 发生read-and-write操作
    * 集合，long型等操作（它们是非原子性操作的时候）
 
> 金句

**在采用一项技术的同时，一定要清楚它带来的问题是什么，以及如何规避**

举个例子，我们为了对系统实施监控，会引入例如pinpoint之类的AMP组件，解决了监控问题
的同时也带来了性能问题，比如对带宽的占用，增加了接口响应的延时等；再比如，微服务
架构是为了解决单体应用灵活性差等问题而出现，同时也带了了架构的复杂度，增加了服务
之间通信，数据隔离等问题。所以，一个技术在解决某个问题的同时可能带来新的问题，这样
我们可能又会为新的问题引入别的技术来处理，这是个不断循环的过程。因此，我们在评估
一项技术的时候，需要充分考虑其负面影响，怎么权衡利弊，实现利益最大化。


#### [02 | Java内存模型：看Java如何解决可见性和有序性问题](https://time.geekbang.org/column/article/84017)

> 笔记

* 什么是JMM 内存模型?
    * 对cpu缓存，编译器优化等按需禁用的方法和规范
        * 方法-> volatile/synchronized/final
        * 规范 -> Happnes-Before
            * 前一个操作的结果对后一个操作是可见的，即使两个操作不在一个线程
    * 避免缓存，编译器优化等导致的并发安全问题，并交给开放人员控制

* Happens-Before包括哪些规则？
    * 程序顺序性规则
        * 单线程来说，前面的操作总是对后面的操作可见
    * volatile 变量规则
        * 对一个volatile变量的写，总是对它后面的读可见
    * 传递性
        * A 对 B可见，B对C可见，则A对C可见
    * 锁规则
        * 锁释放前的操作，总是对下一次获取锁的操作可见
    * 线程start规则
        * start前的操作，总是对被start的线程操作可见
    * 线程join 规则
        * join线程的操作，总是对join之后的线程操作可见

> 金句

**Happens-Before 的语义本质上是一种可见性，A Happens-Before B 意味着 A 事件对 B 事件来说是可见的，无论 A 事件和 B 事件是否发生在同一个线程里**

以前不是很理解为啥要提出来一个happens-before规则，它为了解决什么问题？实际上它是解决了可见性问题，从而避免因为可见性导致的线程安全问题。
由于缓存，线程切换，指令重排等原因，应用层很难保证多线程读写共享变量的一致性，所以才需要JMM这种机制来约束。即使是这样，它也提供了充分的
灵活性，交给开放人员自己决定怎么使用，例如只有在必要的情况下才使用volatile或synchronized等，这样可以更好的协调性能与安全之间的平衡。

#### [03 | 互斥锁（上）：解决原子性问题](https://time.geekbang.org/column/article/84344)

> 笔记

* 锁的作用是什么？
    * 原子性保证 -> 同一时刻只有一个线程执行
    * 可见性保证 -> 锁释放前的操作对下一个获取锁的操作可见
    * 原子性+可见性保证资源的并发安全
	* 对资源的读写操作进行加锁，则读到的资源一定是修改后的资源
	* 如果仅对写加锁，无法保证读到变量的最新值
```
class SafeCalc { 
    
    long value = 0L; 

    //获取锁时可见上一个锁释放前的操作，即可以读到最新的value值
    synchronized long get() { 
	return value; 
    } 
    
    //保证同时只有一个线程更新value
    synchronized void addOne() {
	value += 1; 
    }
}

```

* 锁与资源的关系
    * 资源访问的原子性通过锁来保护，一把锁对应一个或多个资源
    * 如果资源采用不同的锁，将无法保证同时只有一个线程访问该资源
	* 锁对象记录当前获取锁的线程，不同的锁可以记录不同的线程

* 锁的实现
    * 互斥锁 -> synchronize
	* 静态方法 -> 隐式将当前类对象作为锁对象
	* 实例方法 -> 隐式将当前类的实例对象作为锁对象
	* 代码块 -> 自定义对象作为锁对象

> 感想

之前不是很明白为什么要同时对读写加锁，难道不是对写加锁就可以了吗？这个问题其实应该
从可见性去考虑，不加锁的读无法保证它的更新是可见的，即读到的也许是旧值。不过也可
通过volatile 修饰变量保证它立即可见，但是如果读写同时发生，读到的仍然是旧值，因此
此时写可能还没完成或变量还没写入内存。因此如果对资源的实时性要求苛刻，则需要通过
锁来保证可见性和原子性。

#### [04 | 互斥锁（下）：如何用一把锁保护多个资源？](https://time.geekbang.org/column/article/84601)

> 笔记

* 面向资源使用锁
    * 资源不相关 -> 使用不同的锁
	* 保证资源访问可以并发执行
	    * 锁粒度小，提高性能
    * 资源相关 -> 使用相同的锁
	* 保证有相互依赖的资源访问原子性
	    * 相同的锁意味着访问不同资源的所有线程共享一个相同的锁对象
		* 锁对象是不可变的
		* 锁对象是多个线程共享的

> 金句

**关联关系如果用更具体、更专业的语言来描述的话，其实是一种“原子性”特征**

我们说资源是相关的，意思是对它们的操作需要满足原子性，即不能存在中间状态，要么全部成功，要么全部失败。
我们说集合的操作可能不是并发安全的，因为它们的读写存在中间状态，不满足原子性。

#### [05 | 一不小心就死锁了，怎么办？](https://time.geekbang.org/column/article/85001)

> 笔记

* 什么是死锁？
    * 多个线程互相等待对方释放资源，永远阻塞
    * 死锁产生的条件
	* 循环等待
		* 为什么不能共享资源？
			* 互斥 -> 资源同时只能被一个线程持有
	    * 为什么不抢占资源？
			* 不可抢占 -> 其他线程无法抢占当前线程持有的资源
	    * 为什么不释放？
			* 占用并等待 -> 持有资源的线程在等待其他资源的时候不会释放
* 如何避免死锁
    * 如何避免等待
	* 一次性持有所有资源，就不必等待其他资源
	    * 增加锁的粒度
	    * 通过资源管理器锁住一批资源的特定操作
		* 意味着资源的其他操作不会被锁定
    * 如何避免循环？
	* 统一获取锁的顺序，例如根据账户id总是从小到大加锁
    * 如何避免一直占有？
	* 设置超时时间 -> Lock


> 金句

**当我们在编程世界里遇到问题时，应不局限于当下，可以换个思路，向现实世界要答案，利用现实世界的模型来构思解决方案，这样往往能够让我们的方案更容易理解，也更能够看清楚问题的本质**

这就是多模型思维，不要局限于只用一种模型或一个角度去分析问题，避免成为“手里只有铁锤的人”。
往往复杂的问题很难只从一个方向突破，我们要善于采用围攻，多点突入。但是这要求我们具备多模型
的能力，无论是生活经验，还是跨学科的知识储备，只有平时注重积累，耐心学习，才能慢慢养成这种
能够应用多种模型分析并解决复杂问题的能力。

> 死锁模拟与分析

准备2个线程通过两把锁去获取两个资源，访问顺序相反：

```
/**
     *  多线程循环等待资源
     *  1 多个线程访问多个有相互依赖的资源
     *  2 访问顺序不一致
     */
    public void deadLock() throws Exception{

        Thread threadA = new Thread(() -> {
            synchronized (lockA){
                resourceA++;

                try {
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                synchronized (lockB) {
                    resourceB++;
                }
            };
        });

        Thread threadB = new Thread(() -> {
            synchronized (lockB){
                resourceB--;

                try {
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                synchronized (lockA) {
                    resourceA--;
                }
            }
        });

        threadA.start();
        threadB.start();
    }

```

分析死锁：jstack pid > tmp

```
"Thread-1" #10 prio=5 os_prio=31 tid=0x00007fc94197e800 nid=0x4e03 waiting for monitor entry [0x0000700009f57000]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at com.example.demo.concurrent.DeadLockDemo.lambda$deadLock$1(DeadLockDemo.java:51)
    - waiting to lock <0x000000079570e960> (a java.lang.Object)
    - locked <0x000000079570e970> (a java.lang.Object)
    at com.example.demo.concurrent.DeadLockDemo$$Lambda$2/2074407503.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:745)

"Thread-0" #9 prio=5 os_prio=31 tid=0x00007fc94197e000 nid=0x4c03 waiting for monitor entry [0x0000700009e54000]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at com.example.demo.concurrent.DeadLockDemo.lambda$deadLock$0(DeadLockDemo.java:35)
    - waiting to lock <0x000000079570e970> (a java.lang.Object)
    - locked <0x000000079570e960> (a java.lang.Object)
    at com.example.demo.concurrent.DeadLockDemo$$Lambda$1/1149319664.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:745)


```  

* 查看线程状态：线程状态为BLOCKED
* 查看持有的锁和等待的锁：
    * locked/waiting to lock -> 互相等待对方释放锁 
        * 说明发生了死锁

> 数据库死锁

1. A连接读获取共享锁：select *from users where name = 'liam1' lock in share mode;
2. B连接删除，获取排它锁：delete from users where name = 'liam1'; （需要等待共享锁释放资源）
3. A删除：delete from users where name = 'liam1'，等待B释放排他锁

A等B的X锁释放，B等A的share锁释放，形成循环等待

最终现象是A删除成功，B抛出异常： Deadlock found when trying to get lock; try restarting transaction

结论：多个连接获取多个资源(多个锁）形成相互等待时会造成死锁，数据库通过死锁检测自动解锁，会导致其中一个连接执行失败。

#### [06 | 用“等待-通知”机制优化循环等待](https://time.geekbang.org/column/article/85241)

> 笔记

* 等待通知机制解决什么问题？

在check and do 的场景下，条件不满足有2种解决方案：

1. 自旋，主动检查，直到条件满足
2. 阻塞，让出cpu，条件满足时被动通知

等到通知机制就是第二种方法的实现，它让线程waiting，让出cpu资源给其他线程，
能提高cpu的利用率。

* 使用等待通知机制的最佳写法

```

//等待
synchronize(lock) {
    //如果不是while，被唤醒的线程在获取锁之前条件变量可能再次被修改为false导致逻辑错误
    while(!condition) {
        lock.wait()
    }
}

//通知
synchronize(lock) {
    condition = true
    //如果选择notify()，只会唤醒其中一个等待线程，导致有的线程永远等待
    lock.notifyAll()
}

```

> ArrayBlockingQueue中等待通知的实现

* 模型
    * 生产消费模型，生产消费者分别属于不同的线程

* 条件
    * 非空: notEmpty -> 非空时才能消费任务
    * 非满: notFull -> 非满时才能生产任务

* 生产

```
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //非满时才能入队，否则等待非满
            while (count == items.length)
                notFull.await();
            //入队说明非空，需要通知非空
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }

    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        //入队成功后，通知非空，唤醒等待非空的消费者线程
        notEmpty.signal();
    }

```

* 消费

```
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //如果队列为空，需要等待，否则需要等待非空
            while (count == 0)
                notEmpty.await();
            //出队后队列肯定是非满的，需要通知非满
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    private E dequeue() {
        final Object[] items = this.items;
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        //出队成功后，通知等待非满的生产线程
        notFull.signal();
        return x;
    }

```

ArrayBlockingQueue通过等到通知机制 (用Condition实现）实现了阻塞队列，入队或出队时需要获取相同的锁，
因此性能相对于LinkedBlockingQueue要差一些

#### [07 | 安全性、活跃性以及性能问题](https://time.geekbang.org/column/article/85702)

* 笔记

* 并发编程可能出现什么问题？
    * 安全性问题
        * 多线程读写共享变量
        * 竞态条件/数据竞争
            * check and do
            * read and write
        * 对象本身操作存在中间状态
            * 32位系统下处理long
            * 非线程安全的容器
    * 活跃性问题
        * 死锁
            * 循环等待(互斥，不可抢占，占用并等待）
        * 饥饿
            * 非公平锁
            * notify 而不是notifyAll
        * 活锁
            * 线程相互谦让（总是优先让对方执行）导致任务没有进展
    * 性能问题（QPS/RT)
        * 串行导致整体并发性能提升受限
            * 受限于串行度 -> S = 1 / (1 - p) * (p / n )
                * p -> 并行度，1-p表示串行度
                * n -> cpu核心数
                * n趋于无限时，1-p 串行度越大，效率越低
        * 线程阻塞后引起上下文切换


* 如何解决并发编程当中出现的问题？
    * 安全性问题
        * 锁, 保证互斥
        * 原子类型变量
    * 活跃性问题
        * 死锁
            * 避免等待，一次性锁住多个资源
            * 避免循环，按固定顺序获取锁
        * 饥饿
            * 避免使用notify
            * 采用公平锁
    * 性能问题
        * 串行度限制
            * 减少锁粒度，采用细粒度锁，尽快释放
        * 线程阻塞引起上下文切换
            * 使用无锁机制
                * TLS -> ThreadLocal
                * 乐观锁 -> cas等
                * CopyOnWrite -> todo

> 活锁代码示例

* 资源：Resource, 只能被一个worker拥有并执行
* 执行者: worker, 一个线程持有一个worker，可执行拥有的资源
* 谦让: 在active的情况下，总是让其他worker优先执行

DEMO:

```
    public static void main(String args[]) {

        Worker workerA = new Worker(true);
        Worker workerB = new Worker(true);
        Resource resource = new Resource();
        resource.setOwner(workerA);

        Thread threadA = new Thread(() -> {
            workerA.work(resource, workerB);
        });

        Thread threadB = new Thread(() -> {
            workerB.work(resource, workerA);
        });

        threadA.start();
        threadB.start();
    }


    //worker
    public class Worker {

    /**
     *  是否活跃，只有活跃的worker才能执行资源
     */
    private boolean active;

    public Worker(boolean active) {
        this.active = active;
    }

    /**
     *  执行资源
     *  优先谦让，如果其他线程活跃，先让其他线程执行
     */
    public void work(Resource resource, Worker other) {

        //只有活跃且拥有资源执行权才执行
        while (active){

            //没有资源执行权
            if (resource.getOwner() != this) {
                //等待并重试
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                continue;
            }

            //优先让其他线程执行
            if (other.isActive()) {
                log.info("yet to other thread");
                resource.setOwner(other);
                continue;
            }

            //执行
            log.info("process the resource");

            //执行完后让出资源给其他线程
            active = false;
            resource.setOwner(other);
        }
    }

   }

//resource
public class Resource {

    /**
     *  资源拥有者
     */
    private Worker owner;
}

```

执行结果：

在worker同时活跃的情况下，一直发生谦让，无法执行资源。

```
10:54:20.776 [Thread-0] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.785 [Thread-1] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.792 [Thread-0] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.797 [Thread-1] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.802 [Thread-0] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.809 [Thread-1] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.814 [Thread-0] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.820 [Thread-1] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.824 [Thread-0] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
10:54:20.832 [Thread-1] INFO com.example.demo.concurrent.liveLock.Worker - yet to other thread
....


```


> 感想

以往对并发编程问题的理解只有安全性这块，不能同时把活跃性和性能联系起来，这导致我们在
选择方案的时候不够全面，往往只想到一种解决方式，例如比较暴力的使用互斥锁。结合另外
两个问题，我们才会去考虑，怎么减小锁的粒度，能否用无锁方案，如何避免死锁或饥饿等问题，
这样才能保证写出来的程序同时兼具性能和安全上的平衡。

#### []()

> 笔记

* 等待通知机制的实现原理
    * 管程 - MESA 模型
        * 管理共享变量或条件变量以及对它的操作的过程  
        * 等待通知机制本质上是对MESA模型下管程的一种实现
        * 其他模型，从条件变量唤醒来看区别：
            * Hasen 
                * 条件满足后不立即唤醒，需要等当前线程执行完后，在最后唤醒
                    * 保证被唤醒线程和当前线程对条件变量或其他共享变量操作的互斥
                * 缺点 -> 不能立刻唤醒
            * Hoare 
                * 条件满足后立即唤醒，阻塞当前线程，阻塞当前线程
                * 被唤醒线程立刻执行，执行完毕后再通知之前阻塞的线程
                    * 通过阻塞的方式保证只有一个线程执行,保证互斥
                * 缺点 -> 需要阻塞唤醒线程，多阻塞了一次
            * MESA
                * 条件满足后立即唤醒，被唤醒线程不会立即执行，而是加入锁等待队列，重新获取锁后根据条件执行
                    * 唤醒和被唤醒线程通过锁控制，不会同时执行，保证互斥
                * 缺点 -> 被唤醒线程重新获取锁后可能条件变量已经再次被修改，因此需要循环判断
    * 队列 -> 实现互斥和等待
        * 锁等待队列 -> 实现互斥
        * 条件变量等待队列 -> 实现同步
            * 线程被通知唤醒后，从条件变量出队，需要再进入锁等待队列去争抢锁

> 金句

**并发编程的两大问题：互斥和同步**

事实上，跳出语言的框架，无论是go还是java，在进行并发编程的时候都要解决互斥和同步的问题，只是
各自的实现方式不同而已。例如golang可能是通过channel机制实现，而java可以通过synchronize+wait/notify
或Lock/Condiiton等，其背后是特定的管程模式。选择适合的模式，例如MESA并实现它，就能保证互斥和同步，
即实现了所谓的并发编程能力。
