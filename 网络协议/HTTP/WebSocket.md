# WebSocket

是一种基于HTTP上的通讯协议，是浏览器和服务器之间全双工通讯的标准。



##功能点



### 推送功能

支持服务器向客户端推送数据



### 减少通信量

只要不断开连接，就一直保持通讯状态，并且WebSocket的首部信息比较小。





## 建立WebSocket

为了实现WebSocket通信，在建立HTTP连接后，需要完成**一次“握手”**。



### 握手.请求

为了实现WebSocket通讯，需要使用到Upgrade字段，告诉服务器通讯的协议发生变更。

![image-20190928124022342](/Users/huangyuan/Library/Application Support/typora-user-images/image-20190928124022342.png)

Sec-WebSocket-key  记录着握手过程中必不可少的键值

Sec-WebSocket-Protocol  记录着通讯过程中客户端支持的子协议（服务器从这里面选一个）



###握手.响应

![image-20190928124328060](/Users/huangyuan/Library/Application Support/typora-user-images/image-20190928124328060.png)

Sec-WebSocket-Accept ：通过请求中的Sec-WebSocket-key值计算生成的



成功建立WebSocket连接之后，通讯时不再采用HTTP数据帧，而是采用WebSocket独特的数据帧。

![image-20190928124519635](/Users/huangyuan/Library/Application Support/typora-user-images/image-20190928124519635.png)





