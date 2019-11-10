# 原子类Atomic

原子类都是基于CAS实现的，使用CAS实现乐观锁，保证内存可见性及并发更新问题。



## AtomicInteger







## AtomicReference

AtomicReference可以保证你在修改对象引用时的线程安全性。也就是读-改-写对象引用是原子的。

```java
public class AtomicReferenceDemo2 {

    public static void main(String[] args) {

        // 创建两个Person对象，它们的id分别是101和102。
        Person2 p1 = new Person2(101);
        Person2 p2 = new Person2(102);

        // 新建AtomicReference对象，初始化它的值为p1对象
        AtomicReference ar = new AtomicReference(p1);

        //更改p1的id.
        p1.setId(106);
        // 通过CAS设置ar。如果ar的值为p1的话，则将其设置为p2。
        boolean tag = ar.compareAndSet(p1, p2);
        System.out.println("设置p2的结果=" + tag);

        System.out.println("看下还是不是p1， 结果=" + ar.get().equals(p1));
    }

}

class Person2 {
    volatile long id;

    public Person2(long id) {
        this.id = id;
    }

    public String toString() {
        return "id:" + id;
    }

    public void setId(long id) {
        this.id = id;
    }
}
```





**对AtomicReference中的属性，不保证内存可见性**，如：

```java
public class AtomicReferenceDemo {

    static AtomicReference<Node> atomicReference = new AtomicReference<>();

    public static void main(String[] args) throws Exception {

        atomicReference.set(new Node());

        Thread t1 = new Thread() {
            @Override
            public void run() {
                Node node = atomicReference.get();
                for (int i = 0; i < 100000; i++) {
                    node.a++;
                }
            }
        };

        Thread t2 = new Thread() {
            @Override
            public void run() {
                Node node = atomicReference.get();
                for (int i = 0; i < 100000; i++) {
                    node.a++;
                }
            }
        };

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println(atomicReference.get().a);
    }


    static class Node {
        public int a;
    }
}

输出
153207
```



### 多个变量的更新保持原子性

```java
/**
 * 演示2个变量的自增操作 如何保持原子性
 */
public class AtomicReferenceDemo3 {

    private AtomicReference<Reference> atomicReference;

    /**
     * 构建器中初始化AtomicReference
     */
    public AtomicReferenceDemo3() {
        Reference reference = new Reference(0, 0);
        this.atomicReference = new AtomicReference<>(reference);
    }

    /**
     * 原子性保持自增
     */
    public void atomicIncr() {
        Reference referenceOld;
        Reference referenceNew;

        long sequence;
        long sequence2;

        while (true) {
            referenceOld = this.atomicReference.get();

            sequence = referenceOld.getSequence();
            sequence++;

            sequence2 = referenceOld.getSequence2();
            sequence2++;

            referenceNew = new Reference(sequence, sequence2);

            /*
             * 比较交换
             */
            if (this.atomicReference.compareAndSet(referenceOld, referenceNew)) {
                break;
            }
        }
    }

    public static void main(String[] args) throws Exception {

        AtomicReferenceDemo3 demo3 = new AtomicReferenceDemo3();

        // 模拟多个线程同时更新
        Thread t1 = new Thread(() -> {
           func(demo3);
        });
        t1.start();

        Thread t2 = new Thread(() -> {
            func(demo3);
        });
        t2.start();

        Thread t3 = new Thread(() -> {
            func(demo3);
        });
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        System.out.println(demo3.getAtomicReference().get());
    }

    private static void func(AtomicReferenceDemo3 demo3) {

        for (int i = 0; i < 100000; i++) {
            demo3.atomicIncr();
        }
    }

    public AtomicReference<Reference> getAtomicReference() {
        return atomicReference;
    }
}

/**
 * 业务场景模拟
 */
class Reference {

    public Reference(long sequence, long sequence2) {
        this.sequence = sequence;
        this.sequence2 = sequence2;
    }

    /**
     * 序列
     */
    private long sequence;

    /**
     * 序列2
     */
    private long sequence2;

    public long getSequence() {
        return sequence;
    }

    public void setSequence(long sequence) {
        this.sequence = sequence;
    }

    public long getSequence2() {
        return sequence2;
    }

    public void setSequence2(long sequence2) {
        this.sequence2 = sequence2;
    }

    @Override
    public String toString() {
        return "Reference{" +
                "sequence=" + sequence +
                ", sequence2=" + sequence2 +
                '}';
    }
}
```











## AtomicStampedRefernce

AtomicStampedReference通过增加版本戳的形式，避免了CAS中的ABA问题

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABA {

    private static AtomicInteger atomicInt = new AtomicInteger(100);

    private static AtomicStampedReference atomicStampedRef =
            new AtomicStampedReference(100, 0);

    public static void main(String[] args) throws InterruptedException {

        Thread intT1 = new Thread(new Runnable() {
            @Override
            public void run() {
                atomicInt.compareAndSet(100, 101);
                atomicInt.compareAndSet(101, 100);
            }
        });

        Thread intT2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                }
                boolean c3 = atomicInt.compareAndSet(100, 101);
                System.out.println(c3); // true
            }
        });

        intT1.start();
        intT2.start();
        intT1.join();
        intT2.join();

        Thread refT1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                }
                atomicStampedRef.compareAndSet(100, 101, atomicStampedRef.getStamp(), atomicStampedRef.getStamp() + 1);
                atomicStampedRef.compareAndSet(101, 100, atomicStampedRef.getStamp(), atomicStampedRef.getStamp() + 1);
            }
        });

        Thread refT2 = new Thread(new Runnable() {
            @Override
            public void run() {
                int stamp = atomicStampedRef.getStamp();
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                }
                // 跟预期的标志位不一样，就证明有别的线程改了，从而避免ABA问题
                boolean c3 =
                        atomicStampedRef.compareAndSet(100, 101, stamp, stamp + 1);
                System.out.println(c3); // false
            }
        });

        refT1.start();
        refT2.start();
    }
}
```













## 参考

https://www.cnblogs.com/java20130722/p/3206742.html