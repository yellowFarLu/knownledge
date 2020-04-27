# HTTP

HTTP是请求-响应模型，在TCP/IP四层协议中的应用层。

HTTP协议用于客户端和服务器之间进行通讯。

请求资源的是客户端；响应资源的是服务端。



##请求报文

![image-20190822102247324](http://ww1.sinaimg.cn/large/006y8mN6gy1g688rz57q0j30n50b60wz.jpg)

请求报文由**请求方法、请求的URI、协议版本、首部字段、内容实体**组成，其中首部字段、内容实体是可选的。







## 响应报文

![image-20190822111629163](http://ww4.sinaimg.cn/large/006y8mN6gy1g6azk1lb53j30j90bujus.jpg)

响应报文由**协议版本、状态码、解释短语、响应首部字段、响应体**组成

响应头和响应体之间使用空行分隔

**HTTP是不会保存状态，所以它是一种无状态协议。**



**HTTP协议本身并不保留请求响应报文的信息，这是为什么？**
把HTTP设计的简单、轻量，不用去管理报文信息的持久化，从而可以快速处理大量的请求



**指定请求URI的方式有很多种：**

- 完整的请求URI

  ![image-20190822111017637](http://ww4.sinaimg.cn/large/006y8mN6gy1g6azk6w1r8j30g805ugmk.jpg)

- 在首部字段host上写域名或者IP地址

  ![image-20190822111030925](http://ww2.sinaimg.cn/large/006y8mN6gy1g6azkijlkhj30fq03gaay.jpg)

- 如果不是访问某个资源而是访问服务器本身，可以使用*代替请求URI

![image-20190822111041683](http://ww1.sinaimg.cn/large/006y8mN6gy1g6azka8fb0j30m101sq2t.jpg)





## 请求方法



### GET方法

请求已被URI标识的资源。服务端返回的可能是某一个文本或者经过计算以后的结果



**示例**

![image-20190822112841278](http://ww4.sinaimg.cn/large/006y8mN6gy1g6azkq1ftoj309k03pweq.jpg)



### POST方法

POST用来传递消息的主体，也就是数据。

把数据放到请求体中，而不是像GET请求一样暴露在url中，一定程度上提高了安全性。



**示例**

![image-20190824202144056](http://ww4.sinaimg.cn/large/006y8mN6gy1g6b1bs46wdj311y0a840w.jpg)



### PUT方法

PUT方法用来传输文件。就像FTP协议的文件上传一样，要求在请求报文的主体中包含文件内容，然后保存到请求URI指定的位置。

由于HTTP1.1版本的PUT不带有验证机制，因此一般不采用这个请求方法。若要采用，可以结合网站自身的验证机制来使用。

**示例**

![image-20190824202726742](http://ww3.sinaimg.cn/large/006y8mN6gy1g6b1hq3jbfj31ac0t0qeu.jpg)



### HEAD方法

HEAD方法用于获取报文首部。HEAD方法其实和GET方法一样，只是它不返回报文主体部分。

主要用于在于确定资源的有效性、资源更新时间。

![image-20190824203221918](http://ww1.sinaimg.cn/large/006y8mN6gy1g6b1muk84vj30y809e0ue.jpg)



### DELTE方法
DELTE方法用于删除请求URI指定的文件。
不带有验证机制，所以一般不会使用DELETE方法。结合服务端的验证机制，保证安全性之后，还是有可能使用这个方法的。

![image-20190828194355496](http://ww2.sinaimg.cn/large/006y8mN6gy1g6fmpq86q2j30m50axn0o.jpg)





### OPTIONS方法

OPTIONS方法查询 请求URI指定的资源，服务器支持哪些方法。

![image-20190828221341204](http://ww1.sinaimg.cn/large/006y8mN6ly1g6fr1izzb6j317w0fswo9.jpg)

![image-20190828221353827](http://ww1.sinaimg.cn/large/006y8mN6ly1g6fr1pxes9j30yw0c4mzi.jpg)

 

### TRACE方法

trance方法用于**让服务器端将之前的请求回环给客户端**。

发送请求时，在MAx-Forward字段中填入数字，每经过一个服务器就减1，当该值减到0的时候，最后一个接收请求的服务器返回200 ok的响应。

会引发XST跨站式追踪

参考：https://blog.51cto.com/zhpfbk/2310807





### CONNECT方法

浏览器和服务器的通信使用HTTPS协议保证信息安全性，然而当浏览器通过代理服务器与目标服务器进行通讯时，代理服务器怎么知道目标服务器的地址和端口呢？由于目标服务器的地址和端口都是在HTTP报文中的，如果代理服务器去解析报文，会破坏安全性。不解析的话，代理服务器无法去获得目标服务器的地址和端口，这时就用到了CONNECT方法。

浏览器发起CONNECT请求，请求报文中含有目标服务器的地址和端口，代理服务器接收到请求报文后，向目标服务器的端口建立TCP连接，建立完成后代理服务器返回200 ok给浏览器，告诉浏览器与目标站点的加密通道已经建立完成了，可以进行通讯。接下来代理服务器仅仅是传递加密的报文，并不会去解析报文，以保证安全性。



**什么时候使用CONNECT方法**

当浏览器使用代理服务器的时候



参考：https://www.joji.me/zh-cn/blog/the-http-connect-tunnel/











## HTTP报文



###概念

用于HTTP协议交互的信息称为**HTTP报文**。是HTTP通讯中的基本单位。由8字节组成。

客户端的HTTP报文称为请求报文。

服务器的HTTP报文称为响应报文。



报文大致可以分为报文首部、报文主体。

报文主体是非必要的。

![image-20190902185906116](http://ww3.sinaimg.cn/large/006y8mN6gy1g6ldimt1pzj30lt0bqtdw.jpg)



**请求行**

包含请求方法、请求URI、HTTP版本



**状态行**

包含响应码、原因短语、HTTP版本



**首部字段**

用于配置请求或者响应的首部字段。

一般分为4种：请求首部、响应首部、通用首部、实体首部



**实体**

作为请求或响应的数据被传送，其内容由实体首部和实体主体组成。

HTTP报文的主体就是实体主体。但是当传输中发生编码操作时，实体主体的内容发生变化，导致它和报文主体产生差异。（难道报文主体就是内容最初版本的一个概念？）







###内容编码

HTTP发送邮件的时候，会先将内容按照一定的编码进行压缩，然后进行网络传输，客户端接收到文件后，再解码。**内容编码**指的就是把文件进行压缩编码。

![image-20190908134612899](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s26x4todj31a40sck5e.jpg)



**内容编码的格式**

gzip、compress、deflate、identity





#### 分块传输编码

如果HTTP传输的内容相当大，那么在传输完毕之前，浏览器就无法显示内容，因此在传输大量数据时，把数据分为多块，就能够让浏览器逐步显示内容。这种把实体主体分块的功能称为**分块传输编码**

![image-20190908134929883](http://ww1.sinaimg.cn/large/006y8mN6gy1g6s2aao27hj319n0u0atk.jpg)

分块传输编码会将实体主体分为多块，然后每一块的开头使用十六进制标明该块大小，而实体主体的最后一块的大小则为0。

使用分块传输编码的实体主体会由客户端负责解码，恢复到原来的实体主体。

HTTP1.1中存在一种**传输编码**的机制，指实体主体以某种编码方式进行传输。这种机制只能用于分块传输编码中。





### 多部分对象集合

**多部分对象集合**用于容纳多种不同类型的数据。 HTTP采用了多部分对象集合，在HTTP报文的主体中可能包含了多种不同类型的数据。

在HTTP中使用多部分对象集合时，需要在首部字段里加上Content-Type。



**多部分对象集合**包含的对象如下：

- multipart/form-data

  在表单文件上传时使用

  ![image-20190908135958117](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s2l6odeij315s0getay.jpg)

- multipart/byteranges

  响应报文包含了多个范围的内容时使用

  ![image-20190908140102655](http://ww4.sinaimg.cn/large/006y8mN6gy1g6s2matkn4j315a0f4taz.jpg)

![image-20190908140117729](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s2mk52r0j315q0auabs.jpg)





### 获取部分内容的范围请求

客户端向服务器请求实体的某个范围的数据。

执行范围请求时，会使用首部字段Range指定byte的范围。byte范围的指定格式如下：

![image-20190908151242721](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s4ovn2q4j31ak0js0vk.jpg)



示例：

![image-20190908151000031](http://ww4.sinaimg.cn/large/006y8mN6gy1g6s4m3sauoj31440u07wh.jpg)

响应吗为206。对于多重范围请求，服务器会在首部字段Content-Type标明multipart/byteranges后返回报文。

如果服务端无法响应范围请求，则会返回200 ok和完整的实体。





## 状态码

### 状态码的分类

![image-20190908151724473](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s4trce83j31280cgjvv.jpg)





### 2XX 成功



#### 200 ok

请求已经正常处理

![image-20190908151831710](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s4uxlfq7j315q0e4wpl.jpg)

在响应报文内，返回的信息会因请求方法的不同而发生改变，比如使用GET方法的时候，响应会返回实体；HEAD方法的时候，响应会返回首部字段。





#### 204 not content

请求已经成功处理，但是响应没有返回实体主体。

一般用于只需要客户端发出请求，而服务器对客户端不需要发送新信息内容的情况下使用。

![image-20190908152347379](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s50eiljpj30yq0dsgvo.jpg)





#### 206 partial content

该状态码表示客户端使用了范围请求，而服务器成功返回请求的实体主体。响应中使用Conten-Range指定实体主体的范围。





### 3XX 重定向



#### 301 moved permanently

![image-20190908152949570](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s56opspuj311e0dg12r.jpg)

永久性重定向。该状态码表明请求的资源已经分配了新的URI，以后应该使用新的URI进行访问。

客户端假如把旧的URI保存了书签，应该使用Location字段的URI重新保存书签。





#### 302 not found

![image-20190908152931071](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s56d08dyj30zo0gedq7.jpg)

临时性重定向。该状态码表示请求的资源被分配了新的URI，希望本次请求使用新的URI。

301是永久性移动，302是临时的。





#### 303 see other

![image-20190908153226906](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s59eloouj312i0f07eu.jpg)

和302一样，用于重定向，但是只能使用GET方法。





#### 304 not modified

![image-20190908153911129](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s5gf2ivij30yg0gewoe.jpg)

该状态码表示客户端发起附带条件的请求，但是资源不满足条件的情况。

自从上次请求过后，资源并没有修改，则会返回304，并且不返回文件内容。从而节省带宽和开销。

304虽然被划为3XX中，但是和重定向没有关系。



#### 307 temporary redirect

 临时重定向，和302有着相同的含义。





### 4XX

4XX的响应，表示是客户端的请求有错误。



#### 400

![image-20190908154209243](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s5jihjfej30zm0f847p.jpg)

该状态码表明客户端发送的请求存在语法错误，服务器无法处理



#### 401 unauthorized

![image-20190908154302667](http://ww4.sinaimg.cn/large/006y8mN6gy1g6s5kfpmzkj30z90u07qt.jpg)

该状态码表示客户端发送的请求必须要有HTTP认证的信息。

若之前已经发出过一次请求，返回401，现在第二次请求后，响应还返回401，则表示认证失败。

浏览器第一次接受到401时，会弹出认证用的对话窗口。





#### 403 forbidden

![image-20190908154707723](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s5ooqyk7j30zw0ewaj6.jpg)

该状态码表示客户端请求该资源，但是服务器拒绝了。





#### 404 not found

![image-20190908154810506](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s5prnkamj318c0fmwn8.jpg)

服务器上没有找到请求的资源。也可以在服务器拒绝请求但是不想说明原因的时候使用。





### 5XX

5xx表示服务器本身发生了错误。



#### 500 server error

![image-20190908154935861](http://ww1.sinaimg.cn/large/006y8mN6gy1g6s5r90splj30y00cw7d2.jpg)

服务器处理请求的时候发生了错误，可能是自身存在bug。



#### 503 service unavailable

![image-20190908155032962](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s5s8kd2aj30yg0fkqay.jpg)

表示服务器超过负载，无法处理新的请求。也有可能是服务器停机维护了。





## 通讯数据转发程序



### 代理

是一种有转发功能的应用程序，将客户端的请求转发给服务器，将服务器的响应转发给客户端。

![image-20190908155456594](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s5wtf4zlj31ak0hmtty.jpg)

持有资源实体的服务器称为源服务器。从源服务器返回的响应经过代理服务器，然后返回给客户端。

![image-20190908155618972](http://ww4.sinaimg.cn/large/006y8mN6gy1g6s5y8wbu6j318m0ekkfv.jpg)

每次经过代理服务器，都会写入Via首部，以标记处经过的主机信息。

**为什么要使用代理服务器？**

- 利用代理服务器实现缓存，以减少网络带宽的流量
- 对特定请求URI的访问控制



#### 缓存代理

代理转发响应时，先把响应资源的副本缓存在代理服务器上面。当代理再次接受到相同资源的请求时，直接返回缓存的副本。缓存资源副本的代理就叫做缓存代理。



#### 透明代理

指转发报文时，不对报文做任何加工的代理类型称为透明代理。





### 网关

网关是转发其他服务器通讯数据的服务器，当接受到客户端请求时，它就像源服务器一样处理请求。

![image-20190908160324611](http://ww4.sinaimg.cn/large/006y8mN6gy1g6s65mh5vzj315q0bcqdv.jpg)

利用网关可以将HTTP请求转化为其他协议进行通讯。

网关和代理的工作机制十分相似，但是网关可以让服务器提供非HTTP协议的服务。

利用网关能提高通讯的安全性，因为可以在客户端和网关的通讯线路上加密以确保连接的安全性。





### 隧道

![image-20190908160919814](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s6bsdtejj318609qn7m.jpg)

隧道指 在客户端和服务器之间进行中转，并且保持双方通讯链接的应用程序。

隧道使用SSL等加密手段进行通讯，确保安全。

隧道不会去解析报文。

隧道会在通讯双方断开连接时结束。





##资源缓存

资源缓存可能是服务器的缓存，也可能是客户端的缓存。用于减少服务器压力，减少带宽流量。



#### 缓存的有效期限

即使存在缓存，也会因为客户端请求、缓存有效期等因素，缓存服务器向源服务器确认资源的有效性。若缓存失效了，则从源服务器重新获取资源。

![image-20190908161418108](http://ww3.sinaimg.cn/large/006y8mN6gy1g6s6gyrse6j31ac0oiwyu.jpg)





#### 客户端缓存

客户端可以把响应的结果缓存到本地磁盘中。如果缓存有效，则直接使用本地缓存。如果缓存过期了，就会向源服务器确认资源的有效性，如果资源真的失效了，则向服务器重新请求资源。

![image-20190908161633110](http://ww2.sinaimg.cn/large/006y8mN6gy1g6s6jazyjoj30um0ok13s.jpg)









###拓展



#### 端到端首部

这类首部会转发给请求最终接受的目标。如果使用了缓存服务器，必须保存在缓存的响应中。该类首部必须被转发



####逐跳首部

此类首部此类首部只对单次转发有效。如果请求/响应通过代理则不在转发。

![image-20190915121344081](http://ww4.sinaimg.cn/large/006y8mN6gy1g702usx8gnj317v0u0787.jpg)





####内容协商机制



#####客户端筛选

服务端把资源的可用列表发给客户端，让客户端自己选。

弊端：

- 多一次网络往返
- 客户端不了解资源的情况，可能选择错误的版本



#####服务端根据请求首部字段筛选返回

通常使用这种机制。

服务端根据客户端请求首部字段，筛选返回给客户端最适合的版本。

可以用于这个机制的请求头字段又分两种：内容协商专用字段（Accept 字段）、其他字段。

首先来看 Accept 字段，详见下表：

| 请求头字段      | 说明                       | 响应头字段       |
| --------------- | -------------------------- | ---------------- |
| Accept          | 告知服务器发送何种媒体类型 | Content-Type     |
| Accept-Language | 告知服务器发送何种语言     | Content-Language |
| Accept-Charset  | 告知服务器发送何种字符集   | Content-Type     |
| Accept-Encoding | 告知服务器采用何种压缩方式 | Content-Encoding |