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

- 公平锁，判断当前锁的占用状态 == 0，如果等于0，则会继续判断当前队列中是否有排队的情况。如果没有，才会使用CAS操作尝试获取锁
- 这样可以保证遵循FIFO的原则，每一个先来的线程都可以最先获取到锁，但是增加了上下文切换与等待线程的状态变换时间。所以效率相较于非公平锁较慢。
