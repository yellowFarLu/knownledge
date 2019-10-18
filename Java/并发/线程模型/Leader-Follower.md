# Leader-Follower

Leader-Follower线程模型。

在Leader-follower线程模型中每个线程有三种模式，leader,follower, processing。

在Leader-follower线程模型一开始会创建一个线程池，并且会选取一个线程作为leader线程，leader线程负责监听网络请求，其它线程为follower处于waiting状态，当leader线程接受到一个请求后，会释放自己作为leader的权利，然后从follower线程中选择一个线程进行激活，然后激活的线程被选择为新的leader线程作为服务监听，然后老的leader则负责处理自己接受到的请求（现在老的leader线程状态变为了processing），处理完成后，状态从processing转换为follower。

可知这种模式下接受请求和进行处理使用的是同一个线程，这避免了线程上下文切换和线程通讯数据拷贝。



















## 参考

https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651959914&idx=1&sn=3600fc162970b39afe25d65bee10cd2e&chksm=bd2d07b68a5a8ea03c2b1372345991b74aa412b7fd96cc82e9ed0eb38e8dd481bf5d8b7b631c&mpshare=1&scene=23&srcid=1017C51dI9CWZxjNXmMAoMZn&sharer_sharetime=1571293407928&sharer_shareid=1c062d5c810b024acf7d4936fe834135%23rd



https://blog.csdn.net/zhailuxu/article/details/80037942