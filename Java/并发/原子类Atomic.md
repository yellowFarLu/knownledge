# 原子类

原子类都是基于CAS实现的，使用CAS实现乐观锁，保证内存可见性及并发更新问题。



## 原子更新基本类型

### AtomicInteger

AtomicInteger的常用方法如下：

- int addAndGet(int delta) ：以原子方式将输入的数值与实例中的值（AtomicInteger里的value）相加，并返回结果
- boolean compareAndSet(int expect, int update) ：如果输入的数值等于预期值，则以原子方式将该值设置为输入的值。
- int getAndIncrement()：以原子方式将当前值加1，注意：这里返回的是自增前的值。
- void lazySet(int newValue)：最终会设置成newValue，使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。关于该方法的更多信息可以参考并发网翻译的一篇文章《[AtomicLong.lazySet是如何工作的？](http://ifeve.com/how-does-atomiclong-lazyset-work/)》
- int getAndSet(int newValue)：以原子方式设置为newValue的值，并返回旧值。



Atomic包提供了三种基本类型的原子更新，但是Java的基本类型里还有char，float和double等。那么问题来了，如何原子的更新其他的基本类型呢？Atomic包里的类基本都是使用Unsafe实现的，让我们一起看下[Unsafe的源码](http://www.docjar.com/html/api/sun/misc/Unsafe.java.html)，发现Unsafe只提供了三种CAS方法，compareAndSwapObject，compareAndSwapInt和compareAndSwapLong，再看AtomicBoolean源码，发现其是先把Boolean转换成整型，再使用compareAndSwapInt进行CAS，所以原子更新double也可以用类似的思路来实现。





## 原子更新数组

通过原子的方法更新数组里的某个元素。数组作为参数传入构造函数时，会被拷贝一份。所以在原子类中修改元素，不会影响原来的数组。

通过原子的方式更新数组里的某个元素，Atomic包提供了以下三个类：

- AtomicIntegerArray：原子更新整型数组里的元素。
- AtomicLongArray：原子更新长整型数组里的元素。
- AtomicReferenceArray：原子更新引用类型数组里的元素。

AtomicIntegerArray类主要是提供原子的方式更新数组里的整型，其常用方法如下

- int addAndGet(int i, int delta)：以原子方式将输入值与数组中索引i的元素相加。
- boolean compareAndSet(int i, int expect, int update)：如果当前值等于预期值，则以原子方式将数组位置i的元素设置成update值。





## 原子更新引用类型

原子更新基本类型的AtomicInteger，只能更新一个变量，如果要原子的更新多个变量，就需要使用这个原子更新引用类型提供的类。Atomic包提供了以下三个类：

- AtomicReference：原子更新引用类型。
- AtomicReferenceFieldUpdater：
  - 原子更新引用类型里的字段。
  - 字段必须是volatile修饰的，并且是public
- AtomicMarkableReference：原子更新带有标记位的引用类型。可以原子的更新一个布尔类型的标记位和引用类型。构造方法是AtomicMarkableReference(V initialRef, boolean initialMark)



### AtomicMarkableReference

### AtomicReferenceFieldUpdater

```java
public class AtomicDemo {

    private static void atomicReferenceFieldUpdaterDemo() {
        Person person = new Person(1001,"whisper");

        AtomicReferenceFieldUpdater<Person,String> updater =
                AtomicReferenceFieldUpdater.newUpdater(Person.class, String.class,"tem2");

        boolean isSuccess = updater.compareAndSet(person,"whisper","godyan");

        System.out.println("更新的结果=" + isSuccess);
        System.out.println("修改后的name为:"+person.getTem2());
    }

    private static final Integer INIT_NUM = 10;

    private static final Integer TEM_NUM = 20;

    private static final Integer UPDATE_NUM = 100;

    private static final Boolean INITIAL_MARK = Boolean.FALSE;

    private static AtomicMarkableReference<Integer> atomicMarkableReference =
            new AtomicMarkableReference<>(INIT_NUM, INITIAL_MARK);

    /**
     * 由于线程B修改了对象，标记由false改为true，所以当上下文切换为线程A的时候，如果标记不一致CAS方法就会返回false
     * 可以用于解决ABA问题
     */
    private static void atomicMarkableReferenceDemo() {
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " ： 初始值为：" + INIT_NUM +
                    " , 标记为： " + INITIAL_MARK);

            SleepUtils.second(1);

            if (atomicMarkableReference.compareAndSet(INIT_NUM, UPDATE_NUM,
                    atomicMarkableReference.isMarked(), Boolean.TRUE)) {
                System.out.println(Thread.currentThread().getName() + " ： 修改后的值为：" +
                        atomicMarkableReference.getReference() + " , 标记为： " +
                        atomicMarkableReference.isMarked());
            }else{
                System.out.println(Thread.currentThread().getName() +  " CAS返回false");
            }
        }, "线程A").start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " ： 初始值为：" +
                    atomicMarkableReference.getReference() + " , 标记为： " + INITIAL_MARK);
            atomicMarkableReference.compareAndSet(atomicMarkableReference.getReference(),
                    TEM_NUM, atomicMarkableReference.isMarked(), Boolean.TRUE);
            System.out.println(Thread.currentThread().getName() + " ： 修改后的值为：" +
                    atomicMarkableReference.getReference() + " , 标记为： " + atomicMarkableReference.isMarked());
        }, "线程B").start();

    }

    public static void main(String[] args) {
        atomicMarkableReferenceDemo();
    }

}
```





### AtomicReference

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











## 原子更新字段类

如果我们只需要某个类里的某个字段，那么就需要使用原子更新字段类，Atomic包提供了以下三个类：

- AtomicIntegerFieldUpdater：原子更新整型的字段的更新器。
- AtomicLongFieldUpdater：原子更新长整型字段的更新器。
- AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于原子的更数据和数据的版本号，可以解决使用CAS进行原子更新时，可能出现的ABA问题。

原子更新字段类都是抽象类，每次使用都时候必须使用静态方法newUpdater创建一个更新器。

限制：

- **原子更新类的字段的必须使用volatile修饰符**。
- 字段只能是基础数据类型
- 只能是实例变量

AtomicIntegerFieldUpdater的例子代码如下：

```java
public class AtomicIntegerFieldUpdaterTest {

	private static AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater
			.newUpdater(User.class, "old");

	public static void main(String[] args) {
		User conan = new User("conan", 10);
		System.out.println(a.getAndIncrement(conan));
		System.out.println(a.get(conan));
	}

	public static class User {
		private String name;
		public volatile int old;

		public User(String name, int old) {
			this.name = name;
			this.old = old;
		}

		public String getName() {
			return name;
		}

		public int getOld() {
			return old;
		}
	}
}
```

```java
public class AtomicIntegerFieldUpdaterTest {

    // 如果使用private修饰符，必须在本类中使用AtomicIntegerFieldUpdater原子更新字段
    private volatile int old;

    public AtomicIntegerFieldUpdaterTest(int old) {
        this.old = old;
    }

    private static AtomicIntegerFieldUpdater<AtomicIntegerFieldUpdaterTest> a = AtomicIntegerFieldUpdater
            .newUpdater(AtomicIntegerFieldUpdaterTest.class, "old");

    public static void main(String[] args) {
        AtomicIntegerFieldUpdaterTest conan = new AtomicIntegerFieldUpdaterTest( 10);
        System.out.println(a.getAndIncrement(conan));
        System.out.println(a.get(conan));
    }
}
```









### AtomicStampedRefernce

**AtomicStampedReference通过增加版本戳的形式**，避免了CAS中的ABA问题

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

[各种原子类的使用](https://www.jianshu.com/p/eb1a97b0a1b0)