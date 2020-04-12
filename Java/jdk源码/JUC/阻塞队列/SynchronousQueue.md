# SynchronousQueue



## 概述

它不是一个真正的队列，不会为队列中的元素维护存储空间，它维护一组线程，这些线程在等待着把元素移除或者加入队列。

- **不用通过串行方式完成入队或者出队的操作，从而降低数据从生产者到消费者的时延**
- put和take方法会一直阻塞，直到有另一个线程准备好参与到交付过程中。
- 仅当有足够多的消费者，并且总有一个消费者准备好获取交付的工作时，才适合使用同步队列
- ThreadPoolExecutor的缓存队列，默认使用SynchronousQueue

SynchronousQueue像一个传球手，负责把生成者的数据直接传递给消费者，列表本身不存储任何元素，非常适合传递性场景。



## 原理

SynchronousQueue不是一个真正的队列，其主要功能不是存储元素，而且维护一个排队的线程。这些线程等待把元素加入或者移除队列。每一个线程的入队（出队）操作必须等待另一个线程的出队（入队）操作。

当许多线程进行入队时，并且没有线程来取数据，那么这些入队线程都会阻塞，SynchronousQueue 也会形成一个队列，队列中的每个节点有存储的数据，同时也有阻塞的线程（方便后续唤醒），因此SynchronousQueue 中既存储了数据，也存储了线程。把SynchronousQueue 当做普通的队列来用肯定是不行的，其SynchronousQueue 的主要意义在于适合传递性的工作，充当一个传球手的角色，只有两边同时准备好了的情况下才了交接“数据”。

SynchronousQueue有个比较特别的地方，**它没有使用锁来实现同步，而是通过CAS保证线程安全。**



SynchronousQueue有两种模式：

- 公平模式
  所谓公平就是遵循先来先服务的原则，因此其内部使用了一个FIFO队列 来实现其功能。
- 非公平模式
  SynchronousQueue 中的非公平模式是默认的模式，**其内部使用栈来实现其功能，也就是 后来的先服务，这个似乎也太不公平了**。





## 队列中存储的内容

```java
static final class SNode {

            volatile SNode next;        // next node in stack

            volatile SNode match;       // the node matched to this

            /**
             * 阻塞等待的线程
             */
            volatile Thread waiter;     // to control park/unpark

            /**
             * data; or null for REQUESTs
             * 数据，对应消费者来说，该字段为空
             */
            Object item;

            /**
             * 模式，可以是 消费者、生产者、匹配模式
             */
            int mode;
  
 .....
```







## 公平模式



### take()、put()

底层调用的是同一个方法。

SynchronousQueue 初始化是一个空队列，队列头尾指针都指向的是一个”空节点”，放进队列的不再是单独的数据，而是操作，这个操作可以是出队（take）操作，也可以是入队（put）操作。

其中take和put 是互相匹配的操作，也就是说，如果本次是take(put)，上次也是take(put)，那么这是相同的操作，需要放入队列，等待与其相应的匹配操作，如果本次是take(put)，上次是put(take)，那么则是互相匹配的操作，那么就相当于出队（put的数据传递给了take，take需要的数据由put来提供）。



假设现在队列为空，来了一个take操作，因为存在线程的并发操作，因此需要检查数据的有效性，现在需要将take 操作入队，如果在入队期间，没有线程修改队列，那么成功将操作入队，同时更新队列尾指针，如果有线程修改了队列，那么在重新开始进行操作。

入队后，当前线程经过一定时间的自旋后就阻塞了，等到匹配的操作到来，现在又来了一个take 操作，本次的操作和上次（**队尾指向的操作**）都是相同的（模式相同），那么进行重复上次的入队操作，线程自旋后阻塞，现在队列中有两个节点，同时有两个线程也阻塞了。

现在又来一个put 操作，本次操作和上次是不一样的，因此这两个操作就是匹配的（队列里面放的肯定都是相同操作的“数据”），**那么现在就开始从队列头开始进行匹配**，如果再匹配过程中，其它现在也进行了匹配，修改了队列，那么尝试更新队列头指针，然后重新再开始匹配，如果队头指针的操作和这次操作匹配，那么就出队（推进头指针），然后唤醒该匹配操作阻塞的线程，两个操作顺利匹配完成，现在队列中还有一个操作等待匹配（take），继续重复上面的过程。





## 非公平模式

非公平的SynchronousQueue是用栈来实现的，我们知道传统的队列，队尾放数据，队头取数据，但是栈总是在栈顶操作数据。

TransferStack中定义了三个状态：REQUEST表示消费者，DATA表示生产者，FULFILLING，表示操作匹配状态。任何线程对TransferStack的操作都属于上述3种状态中的一种（对应着SNode节点的mode）。同时还包含一个head域，表示栈顶



### take()、put()

将一个操作和栈顶进行匹配，如果和栈顶是相同的操作，那么就直接入栈。

如果和栈顶不是相同的操作（也就是匹配的操作，take匹配put，put匹配take），那么现在先不急出栈，因为此时可能有线程真正入栈，为了避免出现操作错误，这里加了一个环节，如果操作是匹配的(即需要出栈)，那么入栈一个节点，并标记是真正匹配状态，表示的是栈顶操作节点真正匹配，如果其他线程发现这个过程，那么就会帮助其匹配（使其顺序完成出栈工作），完成匹配过后，再进行自身的操作。







## 代码实战

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.SynchronousQueue;

/**
 * @author huangy on 2019-09-29
 */
public class SynchronousQueueDemo {

    public static void main(String[] args) {
        SynchronousQueue synchronousQueue = new SynchronousQueue();

        Thread t = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    Thread.sleep(1000);

                    /**
                     * 当拿出一个元素之后，如果队列没有被其他线程放入元素，当前线程会被阻塞
                     */
                    Object obj = synchronousQueue.take();

                    System.out.println("take obj=" + obj);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        t.start();

        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(1000);
                System.out.println("add item, i=" + i);

                /**
                 * 当放入一个元素之后，如果队列没有其他线程拿出元素，当前线程会被阻塞
                 */
                synchronousQueue.put(i);

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

}
```











## 参考

https://blog.csdn.net/u014634338/article/details/78419445

https://blog.csdn.net/sinat_34976604/article/details/88317045