# LinkedTransferQueue

LinkedTransferQueue是一个由链表结构组成的无界阻塞TransferQueue队列。相对于其他阻塞队列，LinkedTransferQueue多了tryTransfer和transfer方法。



LinkedTransferQueue采用一种预占模式。意思就是消费者线程取元素时，如果队列不为空，则直接取走数据，若队列为空，那就生成一个节点（节点元素为null）入队，然后消费者线程被等待在这个节点上，后面生产者线程入队时发现有一个元素为null的节点，生产者线程就不入队了，直接就将元素填充到该节点，并唤醒该节点等待的线程，被唤醒的消费者线程取走元素，从调用的方法返回。我们称这种节点操作为“匹配”方式。



## 示例

```java
public class LinkedTransferQueueDemo {

    public static void main(String[] args) {

        LinkedTransferQueue<Integer> queue = new LinkedTransferQueue<>();

        Thread consumer = new Thread(()->{
           try {
               System.out.println("消费者尝试拿数据");
               Integer result = queue.take();

               System.out.println("消费者拿到数据啦，result=" + result);
           } catch (Exception e) {
               e.printStackTrace();
           }
        });
        consumer.start();

        SleepUtils.second(2);

        queue.put(1);
    }

}

输出
消费者尝试拿数据
消费者拿到数据啦，result=1  
```





## 源码



### transfer()

transfer方法用于将生产者的元素传递给消费者。

如果有消费者正在等待接收元素，transfer方法可以立即把生产者的元素传递给消费者。

如果没有消费者等待接收元素，transfer会将元素存在队列的尾节点，并等待该元素被消费者消费了才返回。



### tryTransfer()

tryTransfer方法是试探生产者传入的元素能否直接传递给消费者，如果没有消费者等待接收元素，则直接返回false。transfer方法在没有消费者接收元素的情况下，会阻塞生产者线程，而tryTransfer方法不会阻塞生产者线程。





## 参考

[LinkedTransferQueue阻塞队列详解](https://blog.csdn.net/qq_38293564/article/details/80593821)