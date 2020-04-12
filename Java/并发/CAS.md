# CAS



## 原子操作

指的是不可分割的一系列操作，要不都完成，要不都失败。



### 处理器实现原子操作

- 处理器保证从内存中读/写一个字节是原子的
- 总线锁保证原子性：当一个处理器在总线锁发出LOCK信号，那么其他处理器的请求将被阻塞。从而可以独占内存
- 缓存锁定保证原子性：所谓 缓存锁定 是指内存区域如果被缓存在处理器的缓存行中，并且在Lock 操作期间被锁定，那么当他执行锁操作写回到内存时，处理器不在总线上声言 LOCK# 信号，而是修改内部的内存地址，并允许他的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改两个以上处理器缓存的内存区域数据（这里和 volatile 的可见性原理相同），当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。

注意：有两种情况下处理器不会使用缓存锁定。

1. 当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行时，则处理器会调用**总线锁定**。
2. 有些处理器不支持缓存锁定，对于 Intel 486 和 Pentium 处理器，就是锁定的内存区域在处理器的缓存行也会调用总线锁定。



**缓存一致性**

缓存一致性机制整体来说，是当某块CPU对缓存中的数据进行操作了之后，就通知其他CPU放弃储存在它们内部的缓存，或者从主内存中重新读取。



### java实现原子操作

通过锁或者CAS。

二者还有另一个叫法：悲观锁与乐观锁：

悲观锁: 假定会发生并发冲突，即共享资源会被某个线程更改。所以当某个线程获取共享资源时，会阻止别的线程获取共享资源。也称独占锁或者互斥锁，例如java中的synchronized同步锁。

乐观锁: 假设不会发生并发冲突,只有在最后更新共享资源的时候会判断一下在此期间有没有别的线程修改了这个共享资源。如果发生冲突就重试，直到没有冲突，更新成功。CAS就是一种乐观锁实现方式。
悲观锁会阻塞其他线程。乐观锁不会阻塞其他线程，如果发生冲突，采用死循环的方式一直重试，直到更新成功。



## CAS

### 概念

CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。非阻塞。

CAS （compareAndSwap），中文叫比较交换，一种无锁原子算法。过程是这样：它包含 3 个参数 CAS（V，E，N），V表示要更新**变量的值**，E表示预期值，N表示新值。仅当 V值等于E值时，才会将V的值设为N，如果V值和E值不同，则说明已经有其他线程做两个更新，则当前线程则什么都不做。最后，CAS 返回当前V的真实值。CAS 操作时抱着乐观的态度进行的，它总是认为自己可以成功完成操作。

当多个线程同时使用CAS 操作一个变量时，只有一个会胜出，并成功更新，其余均会失败。失败的线程不会挂起，仅是被告知失败，并且允许再次尝试，当然也允许实现的线程放弃操作。基于这样的原理，CAS 操作即使没有锁，也可以发现其他线程对当前线程的干扰。

与锁相比，使用CAS会使程序看起来更加复杂一些，但由于其非阻塞的，它对死锁问题天生免疫，并且，线程间的相互影响也非常小。更为重要的是，使用无锁的方式完全没有锁竞争带来的系统开销，也没有线程间频繁调度带来的开销，因此，他要比基于锁的方式拥有更优越的性能。

简单的说，CAS 需要你额外给出一个期望值，也就是你认为这个变量现在应该是什么样子的。如果变量不是你想象的那样，哪说明它已经被别人修改过了。你就需要重新读取，再次尝试修改就好了。



###底层原理

这样归功于硬件指令集的发展，实际上，我们可以使用同步将这两个操作变成原子的，但是这么做就没有意义了。所以我们只能靠硬件来完成，硬件保证一个从语义上看起来需要多次操作的行为只通过一条处理器指令就能完成。这类指令常用的有：

1. 测试并设置（Tetst-and-Set）
2. 获取并增加（Fetch-and-Increment）
3. 交换（Swap）
4. 比较并交换（Compare-and-Swap）
5. 加载链接/条件存储（Load-Linked/Store-Conditional）

其中，前面的3条是20世纪时，大部分处理器已经有了，后面的2条是现代处理器新增的。而且这两条指令的目的和功能是类似的，在IA64，x86 指令集中有 cmpxchg 指令完成 CAS 功能，在 sparc-TSO 也有 casa 指令实现，而在 ARM 和 PowerPC 架构下，则需要使用一对 ldrex/strex 指令来完成 LL/SC 的功能。



### 优点

- 不会阻塞线程。将线程挂起的话，将涉及操作系统由用户态到的内核态的切换，从而提高性能。





### 缺点

- ABA问题

比如说一个线程one从内存位置V中取出A，这时候另一个线程two也从内存中取出A，并且two进行了一些操作变成了B，然后two又将V位置的数据变成A，这时候线程one进行CAS操作发现内存中仍然是A，然后one操作成功。尽管线程one的CAS操作成功，但是不代表这个过程就是没有问题的。如果链表的头在变化了两次后恢复了原值，但是不代表链表就没有变化。

**AtomicStampedReference来解决ABA问题**

这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。详细见**原子类Atomic**的笔记。



-  长时间不成功导致开销大

自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。



- 只能保证一个共享变量的原子操作

可以使用AtomicReference保证多个变量更新的原子性。





### 适用场景

适用于竞争较少的场景。





### CAS实现乐观锁

```java
import sun.misc.Unsafe;

import java.lang.reflect.Field;

/**
 *  CAS实现乐观锁
 *  AtomicInteger 实际上就是实现了一个乐观锁
 * @author huangy on 2020-04-12
 */
public class CasLock {

    // 通过反射方法获取unsafe对象
    private static final Unsafe unsafe = getUnsafe();

    private static final long valueOffset;

    private volatile int value;

    static {
        try {
            // 获取内存中value字段的偏移值
            valueOffset = unsafe.objectFieldOffset
                    (CasLock.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    /**
     * 使用乐观锁更新value字段的值
     */
    public void incrValue() {

        int old;

        do {
            // 读取旧值（即预期的值）
            old = value;
        } while (!compareAndSet(old, old + 1));
    }

    public int getValue() {
        return value;
    }

    public static Unsafe getUnsafe() {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            return (Unsafe)field.get(null);

        } catch (Exception e) {
        }
        return null;
    }
}
```

测试代码如下：

```java
/**
 * @author huangy on 2020-04-12
 */
public class CasDemo {

    public static void main(String[] args) throws Exception {

        CasLock casLock = new CasLock();

        Thread  t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    casLock.incrValue();
                }
            }
        });

        Thread  t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    casLock.incrValue();
                }
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(casLock.getValue());
    }

}
```









## 问题

**CAS如何同时具有volatile读和写的内存语义？**

JVM会根据当前处理器的类型来决定是否为cmpxchg指令增加lock前缀。如果程序是在多处理器上运行，就增加lock前缀。如果程序是单处理器上运行，就不增加lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

这个lock前缀的用处：

- 确保对内存的读-改-写操作原子执行
  - 在一些比较旧的处理器中，lock前缀的指令会锁住总线
  - 一些比较新的处理器会使用缓存锁定，来保证指令执行的原子性。缓存锁定将大大降低lock前缀指令的开销
- 禁止该指令与前后的指令进行重排序
- 把写缓冲区中的所有数据刷到主内存中

上述几条规则，足以让CAS实现volatile读和volatile写的内存语义。



<br/>









## 参考

https://www.cnblogs.com/stateis0/p/9062006.html

https://blog.csdn.net/sinat_34976604/article/details/80970939

https://blog.csdn.net/qq_21125183/article/details/80848941