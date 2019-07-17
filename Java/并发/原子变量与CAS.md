### CAS

概念：比较并交换（Compare-and-Set），实际上是一个机器指令。CAS包含了3个操作数，分别是 需要读取的内存位置V、进行比较的值A、准备写入的新值B。CAS的语义：我认为V的值因为为A，如果是，则V的值更新为，否则不修改，并且告诉V的值实际上为多少。是乐观锁的一种实现。

- 当且仅当位置V的值等于A，才会用新值B更新V的值，否则不会执行任何操作。
- 无论位置V的值 是否 等于A，都将返回V原有的值。



#### CAS的模拟代码实现

![image-20190207145609682](https://ws4.sinaimg.cn/large/006tNc79gy1fzxv8095jsj30o20daac8.jpg)

当多个线程尝试使用CAS更新同一个变量时，只有其中一个线程能更新变量的值，而其他线程都将失败。失败的线程不会被挂起，并可以再次尝试。



#### 典型使用

首先从V中读取值A，并根据A计算新值B，然后在通过CAS已原子方式将V中的值由A更新为B（只要在这期间没有其他线程将V的值修改为其他值）。由于CAS能检测来自其他线程的干扰，因此即使不使用锁，也能实现原子的 **读-改-写** 操作。



#### 缺点

调用者要处理竞争问题（通过重试、回退、放弃）





### 原子变量类

原子变量类相当于泛化的volatile变量，能支持原子的和有条件的**读-改-写**操作。基于CAS实现。



### 非阻塞算法

实现非阻塞算法的要诀在于：将原子修改的范围缩小到单个变量上，同时还要维护数据的一致性。

#### 原子的域更新器

能够在已有的volatile域上使用CAS。更新器类提供的原子性比普通原子类更弱一些，因为无法保证底层的域不被直接修改——compareAndSet以及其他算术方法只能确保其他使用原子域更新器方法的原子性。

```java
public class FieldUpdate {

    static class Node {
        public volatile int tem;
    }

    public static void main(String[] args) {
        Node node = new Node();

        AtomicIntegerFieldUpdater<Node> updater =
                AtomicIntegerFieldUpdater.newUpdater(Node.class, "tem");
        int rt = updater.addAndGet(node, 5);
        System.out.println(rt);
    }

}
```



#### ABA问题

值先变成A，然后变成B，最后又变成A。使用CAS算法会认为没有发生变化。

解决方法：加入版本号。变成A1-B2-A3。AtomicStampedReference以及AtomicMarkableReference支持在两个变量上执行原子的条件更新。