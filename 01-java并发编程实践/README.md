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


/发生死锁的话jstack会输出死锁报告
Found one Java-level deadlock:
```  

* 通常会输出死锁提示：Found one Java-level deadlock
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

```

> 感想

以往对并发编程问题的理解只有安全性这块，不能同时把活跃性和性能联系起来，这导致我们在
选择方案的时候不够全面，往往只想到一种解决方式，例如比较暴力的使用互斥锁。结合另外
两个问题，我们才会去考虑，怎么减小锁的粒度，能否用无锁方案，如何避免死锁或饥饿等问题，
这样才能保证写出来的程序同时兼具性能和安全上的平衡。

#### [08 | 管程：并发编程的万能钥匙](https://time.geekbang.org/column/article/86089)

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

#### [09 | Java线程（上）：Java线程的生命周期](https://time.geekbang.org/column/article/86366)

> 笔记

* jvm线程有哪些状态，状态之间如何转化？

| 状态机 | NEW | RUNABLE | BLOCKED | WAITING/TIME_WAITING | TERMINATION |
| :--: |:--: |:--: |:--: |:--: |:--: |
| NEW | x | start | x | x | x |
| RUNABLE | x | x | synchronize获取锁失败 | sleep/wait/lock/join/park | 执行结束 |
| BLOCKED | x | synchronize获取锁成功 | x | x | x |
| WAITING/TIME_WAITING| x | notify/unpark/interrupt | x | x | x |
| TERMINATION | x| x | x| x | x |

注意，这里有几个事件(阻塞）的状态转移需要特别关注：

* 获取锁失败
    * synchronize -> blocked
    * Lock.lock
        * LockSupport.park -> waiting
* 调用阻塞api
    * 例如文件阻塞io -> runnable(系统为sleep)

* jvm线程与系统线程的关系？

jvm线程与系统线程是一一对应关系，状态有区别。

| JVM线程状态 | NEW | RUNABLE | BLOCKED | WAITING/TIME_WAITING | TERMINATION |
| :--: |:--: |:--: |:--: |:--: |:--: |
| 系统线程状态 | x | RUNABLE/RUNNING/SLEEP(阻塞io) | SLEEP | SLEEP | DEAD/ZOMBIE |

* interrupt事件对应的状态转换和行为

| 状态机 | NEW | RUNABLE | BLOCKED | WAITING/TIME_WAITING | TERMINATION |
| :--: |:--: |:--: |:--: |:--: |:--: |
| interrupt | x | 1 设置线程标记位，isInterupt()=true  2 阻塞调用的线程则触发特定异常 | 不可中断 | 触发interruption 异常并清除标志位 | x |

**如何响应interrupt中断信号？**

1. 前提: 线程本身是可中断的，即不能为不可中断状态(进程b状态）或线程的 blocked 状态
2. 主动监测信号 -> 适合Runable线程
	* 查看isInterrupt是否为true, 并进行补偿处理
3. 被动中断 -> 适合waiting的线程
	* 触发Interruption异常，捕获异常进行特定的补偿处理

> 用java.util.concurrent.locks.Lock模拟死锁

```
public class DeadLockDemo2{

    private final Lock lockA = new ReentrantLock();
    private final Lock lockB = new ReentrantLock();
    private Integer resourceA = 0;
    private Integer resourceB = 0;

    /**
     *  多线程循环等待资源
     *  1 多个线程访问多个有相互依赖的资源
     *  2 访问顺序不一致
     */
    public void deadLock() throws Exception{

        Thread threadA = new Thread(() -> {
            lockA.lock();
            try {
                resourceA++;
                try {
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                lockB.lock();
                try {
                    resourceB++;
                }finally {
                    lockB.unlock();
                }
            }finally {
                lockA.unlock();
            }

        });

        Thread threadB = new Thread(() -> {
            lockB.lock();
            try {
                resourceB--;
                try {
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                lockA.lock();
                try {
                    resourceA--;
                }finally {
                    lockA.unlock();
                }
            }finally {
                lockB.unlock();
            }
        });

        threadA.start();
        threadB.start();
    }

    public static void main(String[] args) throws Exception{
        DeadLockDemo2 deadLockDemo = new DeadLockDemo2();
        deadLockDemo.deadLock();
    }

}

```

jstack <pid> 分析线程调用栈：

```
"Thread-1" #10 prio=5 os_prio=31 tid=0x00007f845f878800 nid=0x4f03 waiting on condition [0x000070000a4dc000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x0000000795710588> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2.lambda$deadLock$1(DeadLockDemo2.java:59)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2$$Lambda$2/1989780873.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)

"Thread-0" #9 prio=5 os_prio=31 tid=0x00007f845f878000 nid=0x4d03 waiting on condition [0x000070000a3d9000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000007957105b8> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2.lambda$deadLock$0(DeadLockDemo2.java:37)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2$$Lambda$1/2074407503.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)


Found one Java-level deadlock:
=============================
"Thread-1":
  waiting for ownable synchronizer 0x0000000795710588, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
  which is held by "Thread-0"
"Thread-0":
  waiting for ownable synchronizer 0x00000007957105b8, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x0000000795710588> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2.lambda$deadLock$1(DeadLockDemo2.java:59)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2$$Lambda$2/1989780873.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
"Thread-0":
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000007957105b8> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2.lambda$deadLock$0(DeadLockDemo2.java:37)
	at com.example.demo.concurrent.liveLock.DeadLockDemo2$$Lambda$1/2074407503.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)

Found 1 deadlock.

```

* 输出死锁检测报告，提示互相等待的阻塞线程
* 阻塞线程的状态为waiting, 说明使用Lock获取锁失败后阻塞的状态是waiting
    * synchronize实现方式获取锁失败状态是blocked

#### [10 | Java线程（中）：创建多少线程才是合适的？](https://time.geekbang.org/column/article/86666)

> 笔记

* 为什么多线程可以提高应用性能？
    * 怎么评估应用性能？
        * RT/QPS
        * 提高应用性能指的是增加QPS，减少RT
    * 串行 -> 并行
        * 任务分解成并行的子任务，减少任务执行时间，即减少RT
        * 同时处理多个请求，即增加QPS
        * RT的减少使QPS增加
    * 为什么可以并行？是不是线程越多越好？
        * 并行依赖于系统资源：主要是cpu核心数和内核cpu调度机制
            * 单核cpu，线程等待io时，cpu可以处理其他线程
            * 多核cpu, 每个核心都可以处理线程
        * 线程数相对于cpu核心数过多，会产生过多的上下文切换开销
            * 多少线程才是合适的？

* 怎么设置合理的线程数?
    * 多线程是为了充分使用硬件资源，主要是cpu
        * cpu核心数
            * 核心数越多，可支持的线程越多
        * 线程的cpu（sys+usr)使用时间
            * cpu时间越少，为了充分利用空闲的cpu，尽可能设置更多的线程
    * 计算公式
        * 线程数 = cpu核心数 * （1 + io时间/cpu时间）
            * 例如io:cpu = 2:1, 核心为2，则线程数 = 2 * （1+2） = 6 个线程

* 实践中怎么合理设置线程数？
    * 面向业务，不同类型的业务应做到线程池隔离
    * 性能测试，分析请求的时间消耗（io/cpu), 制定合理的线程数
    * 系统监控，分析请求性能（rt/qs)是否达标，动态调整线程数
    * 线程池监控，考察线程池是否繁忙或饱和，动态调整线程数


#### [11 | Java线程（下）：为什么局部变量是线程安全的？](https://time.geekbang.org/column/article/86695)
        
> 笔记

* 方法调用机制
    * 系统用栈管理方法调用
	* 方法调用 -> 入栈
	    * 递归调用可能导致栈溢出，栈的大小和深度是有限的
		* 如何配置每个线程的调用栈大小
		    * -Xss = 512k, 默认是215k
	* 方法返回 -> 出栈
    * 用栈桢管理方法相关的数据
	* 局部变量
	    * 基本类型
	    * 引用类型
		* 引动的对象在堆中分配
	* 参数
	* 方法返回地址
    * 栈是线程独享的
	* 方法结束，栈数据自动被回收，栈内存释放
	* 堆对象是线程共享的，由gc管理对象的回收与内存释放

* 如何保证数据的线程安全？
    * 实际上只要保证数据不是线程共享的，就一定不会出现并发安全问题
	* 即变量是线程封闭的
	    * ThreadLocal 
	    * 方法局部变量

#### [12 | 如何用面向对象思想写好并发程序？](https://time.geekbang.org/column/article/87365)

> 笔记

* 什么是面向对象思想
    * 封装/继承/多态
    * 为了统一管控共享资源的访问，需要统一入口
        * 对资源进行封装，暴露统一的访问方法
            * 方法内保证线程安全

例如用Counter封装对共享变量val的访问:

```
public class SafeCounter {

    private AtomicInteger count = new AtomicInteger();

    private Integer get(){
        return this.count.get();
    }

    private void addOne(){
        this.count.addAndGet(1);
    }

    public static void main(String[] args) throws Exception{

        SafeCounter counter = new SafeCounter();
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++){
                counter.addOne();
            }
        });
        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++){
                counter.addOne();
            }
        });

        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        log.info("result: {}", counter.get());
    }
}

```

对比cas无锁策略和synchronize互斥锁策略，测试后发现无锁策略的耗时更小。

#### [13 | 理论基础模块热点问题答疑](https://time.geekbang.org/column/article/87749)

> 如何理解JMM中线程的工作内存？

* 如何定义JMM
    * JMM描述了jvm线程是如何以及何时可以读取到其他线程更新的共享数据
    * 开发者基于JMM的规则进行开发就能避免因为可见性导致的线程安全问题
* JMM中将内存分解为主内存和线程工作内存，怎么理解工作内存
    * 在物理内存架构中，并没有线程工作内存的说法，物理内存架构只有：
        * 主内存
        * cpu缓存(L1/L2/L3)
        * cpu寄存器
    * jvm定义的堆栈都是在主内存中分配，映射为特定的虚拟内存区域
        * cpu运行线程指令的时候，会把堆栈的数据从主内存加载到缓存，再加载到寄存器
    	* 主内存(堆栈）-> cpu缓存 -> 寄存器
    * java线程工作内存可以理解为线程独享的内存：
        * 线程栈，默认大小512k，可通过jvm参数-Xss限制其大小 
        * cpu缓存和寄存器
    * 主内存可以认为线程共享的内存
        * 可以映射为堆

> 什么是逃逸分析？

* 逃逸分析是jvm的一种优化机制
    * 逃逸对象：可能被其他线程或对象引用的对象
    * 逃逸分析：找出那些不会逃逸的对象并进行一定的优化
    * 通过-X:+/-DoEscapseAnalysis开启或关闭（默认开启）
        * 如何查看虚拟机参数
    	* jinfo -flag DoEscapeAnalysis <pid>

* 逃逸分析包括哪些优化
    * 锁消除
        * 例如synchronize(new Object) 
    	* 锁对象不会被多线程持有，不会逃逸，在运行时会取消该锁，线程不会阻塞

    * 对象在栈上分配
        * 如果对象指针不会逃逸，则有可能在栈上分配
    	* 方法退出则回收对象，不需要gc管控

#### [14 | Lock和Condition（上）：隐藏在并发包中的管程](https://time.geekbang.org/column/article/87779)

> Lock相比Synchronize的优势

获取锁失败时，Lock可以实现有条件阻塞或不阻塞

1. 支持锁中断

lock.lockInterruptibly()

* lock.lock()不可响应中断
* synchronized 阻塞的线程不能响应中断

```
//等待的线程可以响应中断
try {
    lock.lockInterruptibly();
} catch (InterruptedException e) {
    e.printStackTrace();
    return;
}
try {
    this.count++;
    sleep(600000);
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    lock.unlock();
}

```

2. 支持锁超时

lock.tryLock(10, TimeUnit,SECOND)

* 超时返回false

```
//等待的时候可以设置超时, 且支持中断
boolean result = false;
try {
    result = lock.tryLock(10, TimeUnit.SECOND)
    if (!result){
        log.info("获取锁超时");
        return;
    }
} catch (InterruptedException e) {
    e.printStackTrace();
    return;
}
try {
    this.count++;
    sleep(600000);
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    lock.unlock();
}

```
3. 支持非阻塞的获取锁

lock.tryLock()

```

//支持非阻塞获取锁，获取失败直接返回false
boolean result = lock.tryLock();
if (!result){
    log.info("获取锁失败");
    return;
}
try {
    this.count++;
    sleep(600000);
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    lock.unlock();
}

```

> Lock如何保证共享变量的可见性

加锁时，cas修改锁的state, 具有和volatile一致的JMM语义

```
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

```

释放锁时, 最后会写state（volatile)

```
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

```

加锁时会读volatile，解锁时会写volatile, 根据JMM的happens-before原则，volatile的写对
volatile的读可见，意味着上一个线程解锁前对共享变量的更新对下一个获取锁的线程可见。

所以，Lock通过volatile的读写保证了和synchronize一致的语义，从而保证了共享变量的可见性。

#### [15 | Lock和Condition（下）：Dubbo如何用管程实现异步转同步？](https://time.geekbang.org/column/article/88487)

> 相比synchronize和Object#notify实现的管程，Lock/Condition实现的管程有哪些优势？

* synchronize的锁对象和条件变量必须为同一个对象，意味着获取锁的时候我们只能指定一个条件变量
    * Lock和Condition是分离的，一把锁可以同时锁定多个条件变量，例如ArrayBlockingQueue
        * Condition notFull
        * Condition notEmpty
    * 相比之下，Lock/Condition实现的管程更加灵活，可以实现多条件的等待通知。

> dubbo如何实现异步转同步
    
* 为什么要异步转同步？
    * 通信协议例如tcp本身就是异步的，发送和接收数据的线程不同
        * 如果发送请求的线程不等待对方的响应，那么它就是异步的，天然是不等待的，请求发送后线程不会阻塞
        * 如果发送请求的线程阻塞直到获取对方响应为止，那么它就是同步的 
    * 如何实现发送和接收线程之间的等待通知呢？
        * 发送请求的线程 -> 阻塞，并等待响应（条件为“已接收到响应”)
        * 接收请求的线程 -> 唤醒，基于等待条件去唤醒请求的线程
        * dubbo通过Lock/Condition, 这里的Condition是“已接收到响应”，来实现等待同时从而完成异步转同步
    * 接收响应的线程如何把响应和请求对应起来？
        * 发送请求时，生成requestId，并将请求id和条件关联：requestId : condition，缓存
            * 不同的发送线程(请求），requestId不同，对应的条件也不同，每个请求线程都会等待自己的条件
        * 接收响应时，从response取出requestId， 从缓存中取出条件并唤醒该requestId对应的请求线程
            * 相当于通过requestId将请求和响应关联起来，并通过等待通知机制实现（请求等待响应通知）实现异步转同步

#### [16 | Semaphore：如何快速实现一个限流器？](https://time.geekbang.org/column/article/88499)

* 信号量与管程模型的区别
    * 应用场景不同
        * 管程：当前线程阻塞直到某个条件满足为止 
            * 网络通信中的同步请求，例如dubbo的同步RPC/http同步请求
                * 等待响应返回
            * 阻塞队列
                * 入队时等待队列不满
                * 出队时等待队列非空
        * 信号量: 限制能够获取资源的线程数量   
            * 对象池/连接池等
                * 资源用完时阻塞获取资源的线程
                * 资源释放时唤醒等待的线程重新获取资源

    * 线程数限制不同
        * 管程或锁限制同时只有一个线程访问资源
        * 信号量限制一个或多个线程访问资源
            * 理论上信号量可以替代管程或锁

* 信号量实现对象池

> 要求

1. 实现一个包含10个对象的对象池
2. 一个线程获取一个对象
3. 线程获取对象后打印当前线程，并延时，模拟任务耗时
4. 同时创建100个线程执行任务，观察阻塞情况，阻塞时打印日志

```
public class ObjectPool<T> {

    /**
     * 信号量，控制获取资源的线程数
     */
    private final Semaphore counter;

    //多线程读写list，要求线程安全的list
    private final List<T> objects = new CopyOnWriteArrayList<>();

    public ObjectPool(Integer size, Class<T> clazz) throws Exception {
        this.counter = new Semaphore(size);
        for (int i = 0; i < size; i++) {
            objects.add((clazz.newInstance()));
        }
    }

    /**
     * 提交任务
     */
    public void submit(Consumer<T> consumer) {
        //获取信号，无可用信号则阻塞
        try {
            this.counter.acquire();
        } catch (InterruptedException ex) {
            ex.printStackTrace();
            return;
        }
        T object = null;
        try {
            //获取对象并执行
            object = objects.remove(0);
            consumer.accept(object);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            //归还对象并释放信号量
            this.objects.add(object);
            this.counter.release();
        }
    }


    public static void main(String[] args) throws Exception {
        ObjectPool<String> objectPool = new ObjectPool<>(10, String.class);
        for (int i = 0; i < 100; i++) {
            Thread thread = new Thread(() -> {
                objectPool.submit(s -> {
                    log.info("thread: {}", Thread.currentThread().getName());
                    try {
                        sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                });
            });
            thread.start();
        }
    }
}

```
                
#### [17 | ReadWriteLock：如何快速实现一个完备的缓存？](https://time.geekbang.org/column/article/88909)

> 如何排查可能因为读写锁导致的线程锁死问题（永久阻塞）

* 锁升级：先获取读锁，未释放的情况下继续获取写锁
* 锁降级: 先获取死锁，未释放的情况下继续获取读锁

ReentrantReadWriteLock 支持降级，不支持升级

实现一段锁升级代码

```
 /**
  *  模拟锁升级（会导致线程永久阻塞）
  */
 public void lockUpgrade(){
     //读共享变量
     readLock.lock();
     try {
         log.info("current count: {}", this.get());
         sleep(3000);
         writeLock.lock();
         try {
             this.addOne();
         }finally {
             writeLock.unlock();
         }
     }catch (Exception e){
         e.printStackTrace();
     }finally {
         readLock.unlock();
     }
 }
```
jstack 导出线程调用栈

```

"pool-1-thread-3" #11 prio=5 os_prio=31 tid=0x00007fc2b42a4000 nid=0x5103 waiting on condition [0x000070000f8ff000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x0000000795f41420> (a java.util.concurrent.locks.ReentrantReadWriteLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantReadWriteLock$WriteLock.lock(ReentrantReadWriteLock.java:943)
	at com.example.demo.concurrent.readWriteLock.ReadWriteLockDemo.lockUpgrade(ReadWriteLockDemo.java:47)
	at com.example.demo.concurrent.readWriteLock.ReadWriteLockDemo.lambda$main$0(ReadWriteLockDemo.java:65)
	at com.example.demo.concurrent.readWriteLock.ReadWriteLockDemo$$Lambda$1/1288141870.run(Unknown Source)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)o

```

线程处于waiting状态(阻塞），查看调用栈，阻塞在WriteLock.lock上，说明有可能因为无法获取写锁导致永久阻塞

> 如何查看进程包含的线程资源使用情况

* 查看进程包含的线程
    * pstree -p <pid>
    * ps -efT | grep <pid>

* 查看线程的资源使用情况
    * top -Hp <pid>

> 如何理解innodb中的读锁

默认读不加锁，可以并发读写（写的时候可以读），不可以并发写。有点类似于copyOnWrite机制，innodb是通过MVVC来实现的，读写的是数据的不同快照。它的问题是可能无法读到最新写入的值，有三种隔离级别来调整读的可靠性：

* read uncommited 读未提交
* read commited 读已提交
* repeated read 可重复读

如果想保证实时的读，就要在写的时候禁止读，即加入读锁

* innodb有2种读锁：
    * lock in share mode -> 类似于java中的ReentranLock读写锁
        * 可以并发读，读写互斥
    * lock for update 
        * 读写互斥
	* 可以快照读，但是不允许加s锁或x锁的读

接下来看下在可重复读级别下，两种锁模式的互斥情况：

1. lock in share mode, 观察是否可以同时读；
2. lock for update， 观察是否可以同时读
3. 读lock in share mode, 观察是否可以写入;
4. 读lock for update, 观察是否可以写入;
5. 写锁的情况下，是否可以读lock in share mode
6. 写锁的情况下，是否可以读lock for update

* 结论

1. lock in share mode 可以并发读，读写互斥
2. lock for update 支持快照读，读写互斥，阻塞其他加锁的读（s或x锁）
3. 写锁支持快照读，阻塞其他加锁的读


#### [18 | StampedLock：有没有比读写锁更快的锁？](https://time.geekbang.org/column/article/89456)

> 笔记

* 为什么说StampedLock比读写锁要快？
    * StampedLock增加了无锁策略 -> 乐观锁
	* 类似于数据库的乐观锁，通过version来保证更新安全，StampedLock通过stamp保证更新安全
	    * 乐观读：返回stamp -> tryOptimisticRead()
	    * 写时验证：validate(stamp)
		* 验证失败 -> 共享变量被其他线程更新
		    * 重新取最新的共享变量值
		    * 升级为悲观读锁 -> readLock()
			* 等价于读写锁，并发读，读写互斥
			* 读写锁模式下保证可读到最新的变量
    * 乐观锁在读多写少的场景性能更好
	* 无锁，并发读，减少线程因为阻塞导致的延时并提高吞吐量
	* 在读少写多的场景下，乐观锁可能频繁更新失败重试，造成cpu繁忙，占用cpu时间
	    * 可升级为读写锁或互斥锁，避免频繁重试

* 使用StampedLock需要注意什么问题？
    * 不支持重入
    * 中断阻塞的StampedLock会导致cpu使用率飙升
	* todo 为什么？

#### [19 | CountDownLatch和CyclicBarrier：如何让多线程步调一致？](https://time.geekbang.org/column/article/89461)

>  CountDownLatch是如何实现等待通知的？

场景：线程阻塞，等待其他线程完成任务后，由最后一个线程唤醒它再执行任务

实现：通过内部类Sync（实际上是AQS的子类）通过锁获取和释放，间接实现了等待通知机制

* 条件: 变量state 作为计数器和等待通知的条件
* 等待：await() -> sync#tryAcquireShared()
    * 对于AQS来说，获取锁失败（返回false)则阻塞当前线程，加入等待队列
	* sync应该在state计数器不为0时阻塞线程（返回false)
* 通知：countDown() -> sync#tryReleaseShared()
    * 对于AQS来说，释放锁成功（返回true)时幻想等待的线程
	* sync应该在state计数器更减为0时唤醒等待线程（返回true)

```

//计数器不为0时等待：
protected int tryAcquireShared(int acquires) {
    //state计数器为0时，获取锁成功，返回1，不等待；否则阻塞当前线程，加等待队列
    return (getState() == 0) ? 1 : -1;
}

//计数器为0时唤醒等待线程
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
	//避免重复释放锁（重复唤醒等待线程）
        if (c == 0)
            return false;
	//cas更新计数器
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
	    //计数器为0时（最后一个countDown的线程），唤醒等待线程
            return nextc == 0;
    }
}

```

> CycleBarrier如何实现等待通知？

场景：线程间相互等待，直到所有线程都完成任务触发同步回调再执行下一批任务

实现: 通过并发包管程（Lock+Condition) 实现等待通知机制

* 条件：变量count
* 等待/通知：await()
    * count--
    * count不为0时（还有线程未完成任务）时阻塞当前线程，加入该条件的等待队列
    * count为0时，执行回调（Runnable), 并开启下一批任务
	* 复原count
	* 唤醒所有基于count等待的线程


```

//计数器为0时：通知
if (index == 0) {  // tripped
    boolean ranAction = false;
    //同步执行回调函数
    try {
        final Runnable command = barrierCommand;
        if (command != null)
            command.run();
        ranAction = true;
	//开启下一轮
        nextGeneration();
        return 0;
    } finally {
        if (!ranAction)
            breakBarrier();
    }
}

//重置计数器并通知等待线程开启下一轮任务
private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();
    // set up next generation
    count = parties;
    generation = new Generation();
}




//等待或超时等待
for (;;) {
    try {
        if (!timed)
            trip.await();
        else if (nanos > 0L)
            nanos = trip.awaitNanos(nanos);
    } catch (InterruptedException ie) {
	...
    }
}

```


