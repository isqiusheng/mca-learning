JUC （java util concurrent）

**超级面试重灾区**

synchrnized wait notify (经典同步机制)

# ReentrantLock

部分场合替代synchronized

- 手工释放锁

  - 标准写法：

    ```java
    lock.lock();
    try {
        xxxxx
    } finally {
        lock.unlock();
    }
    ```

- 可以是公平锁

  - 公平锁：线程抢锁先排队
  - 非公平锁：线程到了就插队抢

- 可被打断的上锁过程

  - tryLock()
  - lockInterruptibly()

- 锁上面的队列可以指定任意数量

  - 区分了不同条件下的等待队列（Condition）
  - ABC ABC 问题

## 可重入验证

```java
import java.util.concurrent.TimeUnit;

/**
 * reentrantlock用于替代synchronized
 * 本例中由于m1锁定this,只有m1执行完毕的时候,m2才能执行
 * 这里是复习synchronized最原始的语义
 */
public class T01_ReentrantLock1 {
    synchronized void m1() {
        for (int i = 0; i < 5; i++) {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(i);
            if (i == 2) {
                m2();
            }
        }
    }

    synchronized void m2() {
        System.out.println("m2 ...");
    }

    public static void main(String[] args) {
        T01_ReentrantLock1 reentrantLock1 = new T01_ReentrantLock1();

        new Thread(reentrantLock1::m1).start();
    }
}
```

运行结果：

```shell
0
1
2
m2 ...
3
4
```

如果这里的锁 synchronized  不可重入。那么只有等m1执行我完毕之后，才能执行m2

如果锁不可重入，子类重写父类的方法，在子类方法中可以使用super() 调用父类的方法。如果这个方法是 synchronized  的。如果不可重入，那么super() 这种写法就不支持了。

## ReentrantLock用法

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 使用reentrantlock可以完成同样的功能
 * 需要注意的是，必须要必须要必须要手动释放锁（重要的事情说三遍）
 * 使用syn锁定的话如果遇到异常，jvm会自动释放锁，但是lock必须手动释放锁，因此经常在finally中进行锁的释放
 */
public class T01_ReentrantLock2 {

    Lock lock = new ReentrantLock();

    void m1() {
        try {
            lock.lock();
            for (int i = 0; i < 5; i++) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(i);
                if (i == 2) {
                    m2();
                }
            }
        } finally {
            lock.unlock();
        }

    }

    synchronized void m2() {
        try {
            lock.lock();
            System.out.println("m2 ...");
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        T01_ReentrantLock2 reentrantLock1 = new T01_ReentrantLock2();

        new Thread(reentrantLock1::m1).start();
    }
}
```

## 尝试获取锁tryLock

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 使用reentrantlock可以完成同样的功能
 * 需要注意的是，必须要必须要必须要手动释放锁（重要的事情说三遍）
 * 使用syn锁定的话如果遇到异常，jvm会自动释放锁，但是lock必须手动释放锁，因此经常在finally中进行锁的释放
 */
public class T01_ReentrantLock3 {

    Lock lock = new ReentrantLock();

    void m1() {
        try {
            lock.lock();
            for (int i = 0; i < 3; i++) {
                TimeUnit.SECONDS.sleep(1);
                System.out.println(i);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }

    /**
     * 使用tryLock进行尝试锁定，不管锁定与否，方法都将继续执行
     * 可以根据tryLock的返回值来判定是否锁定
     * 也可以指定tryLock的时间，由于tryLock(time)抛出异常，所以要注意unclock的处理，必须放到finally中
     */
    synchronized void m2() {
        boolean locked = false;
        try {
            locked = lock.tryLock(1, TimeUnit.SECONDS);
            System.out.println("m2 ... locked = " + locked);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            if (locked) {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) {
        T01_ReentrantLock3 reentrantLock1 = new T01_ReentrantLock3();

        new Thread(reentrantLock1::m1).start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(reentrantLock1::m2).start();
    }

}
```

使用tryLock进行尝试锁定，不管锁定与否，方法都将继续执行

根据tryLock的返回值来判定是否锁定

可以指定tryLock的时间，由于tryLock(time)抛出异常，所以要注意unclock的处理，必须放到finally中

## lockInterruptibly方法

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 使用ReentrantLock还可以调用lockInterruptibly方法，可以对线程interrupt方法做出响应，
 * 在一个线程等待锁的过程中，可以被打断
 */
public class T01_ReentrantLock4 {


    public static void main(String[] args) {

        Lock lock = new ReentrantLock();

         Thread t1 = new Thread(() -> {
             try {
                 lock.lock();
                 System.out.println("t1 start");
                 Thread.sleep(10000);
                 System.out.println("t1 end");
             } catch (InterruptedException e) {
                 e.printStackTrace();
             } finally {
                 lock.unlock();
             }
         }, "t1");

         t1.start();

        Thread t2 = new Thread(() -> {
            try {
                lock.lockInterruptibly();  //可以对interrupt()方法做出响应
                System.out.println("t2 start");
                Thread.sleep(5000);
                System.out.println("t2 end");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }, "t2");


        t2.start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t2.interrupt(); //打断线程2的等待
    }

}
```

使用 lock.lockInterruptibly() 可以对 interrupt() 方法作出响应

## 指定锁的类型（公平锁 or 非公平锁）

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * ReentrantLock还可以指定为公平锁
 */
public class T01_ReentrantLock5 extends Thread {

    static Lock lock = new ReentrantLock(true);

    public void run() {
        for (int i = 0; i < 100; i++) {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + "获得锁");
            } finally {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) {
        T01_ReentrantLock5 rl = new T01_ReentrantLock5();
        Thread th1 = new Thread(rl);
        Thread th2 = new Thread(rl);
        th1.start();
        th2.start();
    }

}
```

> - 默认是非公平锁的。主要是为了效率
> - 公平锁不是绝对的公平的

公平锁遵循FIFO（先到先出）的原则，先到的线程优先获取到资源，后到的会进入队列中进行等待。而非公平锁不支持这个原则

**源码分析**

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

```java
 /**
    * Performs non-fair tryLock.  tryAcquire is implemented in
    * subclasses, but both need nonfair try for trylock method.
    */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

- 非公平锁，判断当前锁的占用状态 == 0，如果等于0，则会直接使用CAS 尝试获取锁
- 在非公平锁里，因为可以直接compareAndSetState来获取锁，不需要加入队列，然后等待队列头线程唤醒再获取锁这一步骤，所以效率较快

```java
/**
    * Fair version of tryAcquire.  Don't grant access unless
    * recursive call or no waiters or is first.
    */
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

- 公平锁，判断当前锁的占用状态 == 0，如果等于0，则会继续判断队列是否为空或者当前线程是否在队列头部。如果没有，才会使用CAS操作尝试获取锁
- 这样可以保证遵循FIFO的原则，每一个先来的线程都可以最先获取到锁，但是增加了上下文切换与等待线程的状态变换时间。所以效率相较于非公平锁较慢。

## isLock判断锁的状态

```java
// 底层源码
final boolean isLocked() {
	return getState() != 0;
}
```

## ReentrantLock VS Synchronized

1. Synchronized 是JVM层次的锁，ReentrantLock是JDK层次的锁实现；
2. Synchronized的锁状态是无法在代码中直接判断的，但是ReentrantLock可以通过isLock() 方法来判断；
3. Synchronized是非公平锁，而ReentrantLock可以是公平锁，也可以是非公平锁；
4. SYnchronized是不可以被中断的，而ReentrantLock#lockInterruptibly方法是可以被中断的；
5. 在发生异常的时，Synchronized会自动释放锁。而ReentrantLock 需要开发者在finally块中显示释放锁;
6. ReentrantLock可以尝试获取锁，以及指定等待时长获取，相对比Synchronized更灵活些。

# CountDownLatch

一个或者多个线程，等待其他多个线程完成某件事情之后才能执行；

门闩

等待线程结束

# CyclicBarrier

多个线程互相等待，直到到达同一个同步点，再继续一起执行。

循环栅栏

满人发车

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class T03_TestCyclicBarrier {

    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(20, () -> System.out.println("满人, 发车"));

        for (int i = 0; i < 100; i++) {
            new Thread(() -> {
                try {
                    barrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

运行结果

```shell
满人, 发车
满人, 发车
满人, 发车
满人, 发车
满人, 发车
```

> 打印5次满人，发车。如果循环的此时是90，那么就打印4次，而且此时线程是阻塞状态，没有结束

## CountDownLatch 和 CyclicBarrier 的不同点：

- CountDownLatch的计数器只能使用一次。而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次。
- CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得CyclicBarrier阻塞的线程数量。isBroken方法用来知道阻塞的线程是否被中断。比如以下代码执行完之后会返回true。
- CountDownLatch会阻塞主线程，CyclicBarrier不会阻塞主线程，只会阻塞子线程。
- 某线程中断CyclicBarrier会抛出异常，避免了所有线程无限等待。

# Phaser

Phase是阶段的意思

一般用不到。涉及到遗传算法会用到

每个阶段要等人到齐，才能进行下一个阶段。

```java
import java.util.Random;
import java.util.concurrent.Phaser;
import java.util.concurrent.TimeUnit;

public class T04_TestPhaser {

    static Random r = new Random();
    static MarriagePhaser phaser = new MarriagePhaser();


    static void milliSleep(int milli) {
        try {
            TimeUnit.MILLISECONDS.sleep(milli);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {

        phaser.bulkRegister(7);

        for (int i = 0; i < 5; i++) {

            new Thread(new Person("p" + i)).start();
        }

        new Thread(new Person("新郎")).start();
        new Thread(new Person("新娘")).start();

    }


    static class MarriagePhaser extends Phaser {
        @Override
        protected boolean onAdvance(int phase, int registeredParties) {

            switch (phase) {
                case 0:
                    System.out.println("所有人到齐了！" + registeredParties);
                    System.out.println();
                    return false;
                case 1:
                    System.out.println("所有人吃完了！" + registeredParties);
                    System.out.println();
                    return false;
                case 2:
                    System.out.println("所有人离开了！" + registeredParties);
                    System.out.println();
                    return false;
                case 3:
                    System.out.println("婚礼结束！新郎新娘抱抱！" + registeredParties);
                    return true;
                default:
                    return true;
            }
        }
    }

    static class Person implements Runnable {
        String name;

        public Person(String name) {
            this.name = name;
        }

        public void arrive() {

            milliSleep(r.nextInt(1000));
            System.out.printf("%s 到达现场！\n", name);
            phaser.arriveAndAwaitAdvance();
        }

        public void eat() {
            milliSleep(r.nextInt(1000));
            System.out.printf("%s 吃完!\n", name);
            phaser.arriveAndAwaitAdvance();
        }

        public void leave() {
            milliSleep(r.nextInt(1000));
            System.out.printf("%s 离开！\n", name);
            phaser.arriveAndAwaitAdvance();
        }

        private void hug() {
            if (name.equals("新郎") || name.equals("新娘")) {
                milliSleep(r.nextInt(1000));
                System.out.printf("%s 洞房！\n", name);
                phaser.arriveAndAwaitAdvance();
            } else {
                phaser.arriveAndDeregister();
                //phaser.register()
            }
        }

        @Override
        public void run() {
            arrive();

            eat();

            leave();

            hug();

        }
    }
}
```

# ReadWriteLock

读写锁。其实就是

- 共享锁（读）
- 排它锁（写）

```java
import java.util.Random;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * @author liufei
 * @version 1.0.0
 * @description
 * @date 2022/10/26
 */
public class T05_TestReadWriteLock {

    static Lock lock = new ReentrantLock();
    private static int value;

    static ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    static Lock readLock = readWriteLock.readLock();
    static Lock writeLock = readWriteLock.writeLock();

    public static void read(Lock lock) {
        try {
            lock.lock();
            Thread.sleep(1000);
            System.out.println("read over!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void write(Lock lock, int v) {
        try {
            lock.lock();
            Thread.sleep(1000);
            value = v;
            System.out.println("write over!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }


    public static void main(String[] args) {

        for (int i = 0; i < 18; i++) {
            new Thread(() -> {
                //read(lock);
                read(readLock);
            }).start();
        }

        for (int i = 0; i < 2; i++) {
            new Thread(() -> {
                //write(lock, new Random().nextInt());
                write(writeLock, new Random().nextInt());
            }).start();
        }
    }

}
```

分析下：

如果读锁和写锁，共用同一锁，效率会很低。因为你读的时候，其他线程线程也是可以读的，读线程是互相不影响的。但是使用同一把锁之后，读的时候，其它在线程不管是读锁还是写锁都要等待。

所以使用读写锁

写锁是排它的。写的时候，别的线程不能写也不能读

读锁是共享的。读的时候，别的线程也是可以读的。

在读的上面可以大大提高效率

# Semaphore（信号量）

可以控制同时有几个线程在运行

```java
import java.util.concurrent.Semaphore;

public class T06_TestSemaphore {

    public static void main(String[] args) {

        Semaphore s = new Semaphore(1);

        new Thread(() -> {
            try {
                s.acquire();

                System.out.println("T1 running...");
                Thread.sleep(200);
                System.out.println("T1 running end ...");

            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                s.release();
            }
        }).start();

        new Thread(() -> {
            try {
                s.acquire();
                System.out.println("T2 running...");
                Thread.sleep(200);
                System.out.println("T2 running end ...");

            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                s.release();
            }
        }).start();
    }
}
```

这里有两个线程

```shell
# 如果 Semaphore 的数量是1，那么就表示只能允许一个线程运行。输出结果
T1 running...
T1 running end ...
T2 running...
T2 running end ...

# 如果 Semaphore 的数量是2，那么就表示能允许两个线程同时运行。输出结果
T1 running...
T2 running...
T2 running end ...
T1 running end ...
```

# Exchanger（交换器）

**两个**线程之前交换数据

```java
import java.util.concurrent.Exchanger;

public class T07_TestExchanger {

    public static void main(String[] args) {

        Exchanger<String> exchanger = new Exchanger<>();

        new Thread(() -> {
            String s = "T1";
            try {
                s = exchanger.exchange(s);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " -> " + s);
        }, "t1").start();

        new Thread(() -> {
            String s = "T2";
            try {
                s = exchanger.exchange(s);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println(Thread.currentThread().getName() + " -> " + s);
        }, "t2").start();
    }
}
```

运行结果：

```java
t2 -> T1
t1 -> T2
```

**exchange() 方法是阻塞方法**

# LockSupport

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.LockSupport;

public class T08_TestLockSupport {

    public static void main(String[] args) {
        Thread t = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                System.out.println(i);
                if (i == 5) {
                    LockSupport.park();
                }
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        t.start();

        // 如果提前unpark，那么后面再park，也不在有用
        //LockSupport.unpark(t);

        try {
            TimeUnit.SECONDS.sleep(8);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("after 8 senconds!");
        LockSupport.unpark(t);
    }
}
```

注意：

- 如果提前unpark，那么后面再park，也不在有用
- LockSupport是可以指定线程的