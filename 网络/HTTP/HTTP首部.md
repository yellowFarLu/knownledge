

## HTTP首部

HTTP首部：用于在请求和响应过程中传递一些重要信息。



### HTTP首部字段的结构

HTTP首部字段是用 字段名和字段值组成，中间使用 : 分隔，如下

![image-20190914104425327](http://ww4.sinaimg.cn/large/006y8mN6gy1g6yunm6pavj31900ngjw3.jpg)

1个字段名可以有多个值，每个值使用 , 分隔



**如果HTTP首部字段名重复了，会怎样？**

这种情况在规范内尚未明确，根据客户端的不同处理方式不同，有些客户端可能优先处理第一次出现的首部字段，有些客户端可能优先处理最后一次出现的首部字段。



### 首部字段的类型



#### 通用首部字段

请求报文和响应报文两方都会使用的字段



##### Cache-Control

Cache-Control能够控制缓存的行为。

**请求指令**：

![image-20190915120446423](http://ww1.sinaimg.cn/large/006y8mN6gy1g702livhoaj30wc0ie43n.jpg)





**响应指令**

![image-20190915120510378](http://ww3.sinaimg.cn/large/006y8mN6gy1g702lw44loj317g0nen4t.jpg)



###### public

public指令表明缓存是公开的，其他用户也可以使用该缓存。

![image-20190915120604851](http://ww4.sinaimg.cn/large/006y8mN6gy1g702muak5mj317i032glp.jpg)



###### private

缓存只能给特定用户使用。

缓存服务器对该用户返回缓存的响应，对于其他用户的请求，缓存服务器则不会返回缓存，而是请求源服务器。

![image-20190915120659552](http://ww3.sinaimg.cn/large/006y8mN6gy1g702nsinilj31820ju4co.jpg)



###### no-cache

该指令客户端、服务器都可以使用，含义不用。目的是防止从缓存中返回过期的资源。

no-cache并不代表不使用缓存，**no-cache代表不缓存过期的资源**，而是缓存服务器会向源服务器确认资源的有效性，如果资源还有效还是会返回缓存的数据，如果资源失效了，就从把请求转发到源服务器，然后更新缓存。



**客户端的含义**

表示客户端不会接收缓存过的响应

**服务器的含义**

服务通返回的响应包含no-cache，那么缓存服务器不能对响应进行缓存。



![image-20190915121533373](http://ww4.sinaimg.cn/large/006y8mN6gy1g702wpepuhj31200i8qf2.jpg)



**利用no-cache控制客户端使用缓存**

如果在响应中把no-cache字段指定值，那么客户端将无法使用缓存，如下：

![image-20190915121430142](http://ww1.sinaimg.cn/large/006y8mN6gy1g702vliewij318w03uzke.jpg)

如果no-cache不指定值，则客户端端可以使用缓存：

![image-20190915121559075](http://ww1.sinaimg.cn/large/006y8mN6gy1g702x54wf1j319403iglp.jpg)

只能在响应字段中指定该参数值





###### no-store

no-store指不进行缓存。（no-cache是不缓存过期的资源）



###### max-age

代表缓存的过期时间。

**客户端角度**

如果判断缓存资源的缓存时间比指定时间小，客户端就接收响应。

**服务器角度**

当响应包含max-age指令，缓存服务器在max-age指定的时间内将不会对资源进行有效性确认。max-age是保存缓存的最长时间。

如果服务器使用了HTTP1.1，并且同时存在Expires和max-age字段，则Expires字段会被忽略表。在HTTP1.0则max-age会被忽略掉。





###### min-fresh

要求缓存服务器返回未超过指定时间的资源。

这就和max-age类似，但是min-fresh只能用在客户端。

![image-20190915144626626](http://ww2.sinaimg.cn/large/006y8mN6gy1g7079p92pfj31880isjz1.jpg)





###### min-stale

即时资源过期了，只要还在指令指定的时间内，客户端都会接收响应。

如果指令中未指定参数值，那么无论过期多久，客户端都会接收响应。

![image-20190915144638865](http://ww3.sinaimg.cn/large/006y8mN6gy1g7079wp3nvj317y034jrr.jpg)



###### on-if-cache

要求缓存服务器使用缓存，客户端才会响应。该指令要求缓存服务器不重新加载缓存，也不确认缓存的有效性。

![image-20190915145032943](http://ww2.sinaimg.cn/large/006y8mN6gy1g707dyqacyj317o03amxb.jpg)





###### must-revalidate

使用该指令，代理服务器接收到客户端的请求时，会向源服务器确认资源的有效性。

![image-20190915145148930](http://ww4.sinaimg.cn/large/006y8mN6gy1g707fa4dy6j318w03kweo.jpg)

使用must-revalidate指令，服务器会忽略掉max-stale。





###### proxy-revalidate

如果请求带有该指令，那么所有缓存服务器在返回响应之前，都必须检查缓存的有效性。

![image-20190915145513357](http://ww1.sinaimg.cn/large/006y8mN6gy1g707itssnfj317u036aa9.jpg)





###### no-transform

![image-20190915145545151](http://ww1.sinaimg.cn/large/006y8mN6gy1g707jdlykvj3182040q33.jpg)

无论在请求还是响应中，缓存服务器都不能改变实体主体的媒体类型。作用是**防止缓存服务器压缩图片的操作**





###### Cache-Control扩展

自定义指令，服务器对该指令进行处理。如果代理服务器不理解该指令，会直接忽略。





##### Connection

具有如下作用：

- 控制不再转发给代理的首部字段、

  ![image-20190915145933659](http://ww4.sinaimg.cn/large/006y8mN6gy1g707ndrbeoj316o0ry1bu.jpg)

- 管理持久连接

![image-20190915150020080](http://ww2.sinaimg.cn/large/006y8mN6gy1g707o5cburj31660igqad.jpg)

HTTP1.1版本默认是持久连接。当客户端想要断开连接时，使用close指令。

![image-20190915150131983](http://ww1.sinaimg.cn/large/006y8mN6gy1g707peb0gbj315g0lyqiw.jpg)

HTTP1.1之前的版本默认是非持久连接，需要使用Keep-Alive指令来使用持久连接。





##### Date

Date表明创建HTTP报文的时间

![image-20190915150320640](http://ww3.sinaimg.cn/large/006y8mN6gy1g707ra7b1wj318003iweq.jpg)





##### Pragma

Pragma是历史遗留字段，仅为了向后兼容。

![image-20190915150425466](http://ww3.sinaimg.cn/large/006y8mN6gy1g707seh166j317w03iq31.jpg)

是通用首部字段。但只用在客户端发送的请求中。客户端要求所有代理服务器不返回缓存的资源。

![image-20190915150526002](http://ww1.sinaimg.cn/large/006y8mN6gy1g707tgg0x9j31280ew47x.jpg)

因为无法掌握所有代理服务器的HTTP版本，所有一般都会同时带有Cache-Control和Pragma首部字段

![image-20190915150633431](http://ww2.sinaimg.cn/large/006y8mN6gy1g707umnlymj317u04sq3a.jpg)





##### Trailer

该字段说明在报文主体后记录了哪些字段。可以用于分块传输编码时。

![image-20190915150806261](http://ww3.sinaimg.cn/large/006y8mN6gy1g707w88p8hj318q0iigp8.jpg)





##### Transfer-Encoding

规定了传输报文主体的编码方式

![image-20190915150851818](http://ww2.sinaimg.cn/large/006y8mN6gy1g707x1715mj319i0k4wuz.jpg)

HTTP1.1仅支持分块传输编码

![image-20190915151019145](http://ww3.sinaimg.cn/large/006y8mN6gy1g707yjlpnuj31500u0tf5.jpg)





##### Upgrade

客户端用于询问是否可以使用更高版本的协议来进行通讯。

![image-20190915151155668](http://ww4.sinaimg.cn/large/006y8mN6gy1g70807x7ppj319i0j2kcg.jpg)

![image-20190915151214623](http://ww4.sinaimg.cn/large/006y8mN6gy1g7080jmhslj318i0eg0yu.jpg)





##### Via

Via是为了追踪请求和响应报文的传输路径。

报文在经过代理服务器时，会在Via字段上附加该服务器的信息，然后再进行转发。

![image-20190915151541103](http://ww3.sinaimg.cn/large/006y8mN6gy1g70844i11bj31ai0rkha4.jpg)

![image-20190915151621867](http://ww3.sinaimg.cn/large/006y8mN6gy1g7084u4h8zj319809edke.jpg)







##### Warning

告知用户一些缓存相关的问题和告警。

![image-20190915151730654](http://ww3.sinaimg.cn/large/006y8mN6gy1g70860t0b4j318y03qmxl.jpg)



![image-20190915151744925](http://ww2.sinaimg.cn/large/006y8mN6gy1g70869sogbj318k0660ua.jpg)











#### 请求首部字段

请求报文中使用的首部字段



##### Accept

告诉服务器客户端能够处理的媒体类型及媒体类型的优先级。

可使用type/subType形式，一次告诉多种媒体类型。

![image-20190915151952078](http://ww3.sinaimg.cn/large/006y8mN6gy1g7088h8wdsj319y0kw4ey.jpg)

若想给媒体类型增加优先级，可以使用q来表示权重。权重的范围是0~1，不指定权重时，默认是1.0。当服务器支持多种类型时，服务器会优先返回权重高的类型。





##### Accept-Charset

![image-20190915152317512](http://ww2.sinaimg.cn/large/006y8mN6gy1g708c1k3vwj318o0og4fd.jpg)

该字段告诉服务器 客户端支持的字符集以及字符集的优先顺序。同样可以使用q来表示权重。





##### Accept-Encoding

![image-20190915152518770](http://ww4.sinaimg.cn/large/006y8mN6gy1g708e50rdbj317m0n87il.jpg)

告诉服务器 客户端支持的编码及优先级。客户端可以在接收到文件后，进行反编码获取到源文件。







##### Accept-Language

![image-20190915152615386](http://ww3.sinaimg.cn/large/006y8mN6gy1g708f4nexwj31440u07ue.jpg)

![image-20190915152626210](http://ww3.sinaimg.cn/large/006y8mN6gy1g708fb1g6lj317w03imxg.jpg)

告诉服务器 客户端支持的自然语言集及优先级。





##### Authorization

![image-20190915152752933](http://ww2.sinaimg.cn/large/006y8mN6gy1g708gtm8z6j313e0u0nox.jpg)

该字段用来告诉服务器 客户端的认证信息。







##### Expect

![image-20190915152917815](http://ww4.sinaimg.cn/large/006y8mN6gy1g708iad4sxj318e0jgank.jpg)

![image-20190915152933688](http://ww3.sinaimg.cn/large/006y8mN6gy1g708ik4413j318q0f2grl.jpg)





##### Form

告诉服务器 客户端用户的邮件地址

![image-20190915153053074](http://ww3.sinaimg.cn/large/006y8mN6gy1g708jxqmquj31140igk1r.jpg)







##### Host

告诉服务器 **请求资源的主机名和端口号**

![image-20190915153219988](http://ww4.sinaimg.cn/large/006y8mN6gy1g708lg5twej319p0u01dp.jpg)

![image-20190915153307361](http://ww2.sinaimg.cn/large/006y8mN6gy1g708m9iy3kj317y0h4gsv.jpg)







##### if-match

![image-20190915153412672](http://ww1.sinaimg.cn/large/006y8mN6gy1g708necmtoj31780m215b.jpg)

像if-xxxx这种都是**附带条件的请求**。服务器接收到请求后，只有判断条件为真，才会执行请求。

![image-20190915153554296](http://ww3.sinaimg.cn/large/006y8mN6gy1g708p60d0oj31ai0jstpq.jpg)

![image-20190915153608945](http://ww1.sinaimg.cn/large/006y8mN6gy1g708pevo2sj318y0gq7j8.jpg)

![image-20190915153620703](http://ww2.sinaimg.cn/large/006y8mN6gy1g708pmclvtj317o04675e.jpg)

首部字段if-match是附带条件之一。它的值是ETag，服务器判断该值和资源的ETag值是否一致，一致则返回资源。（ETag是与特定资源相关的值，资源变化，ETag也会跟着变化）





##### if-modify-since

![image-20190915153951236](http://ww2.sinaimg.cn/large/006y8mN6gy1g708ta1l74j318i0jm7lw.jpg)

![image-20190915154005440](http://ww3.sinaimg.cn/large/006y8mN6gy1g708tiui99j316o0hmap8.jpg)

如果指定的资源在指定时间后发生过更新，则服务器处理请求。![image-20190915154040328](http://ww2.sinaimg.cn/large/006y8mN6gy1g708u4tkk3j317m0i4wli.jpg)









##### if-none-match

只有在if-none-match的值与资源的ETag值不一致时，服务器才可以执行请求。

与if-match首部字段作用相反。

![image-20190915154338408](http://ww4.sinaimg.cn/large/006y8mN6gy1g708x7u1o7j318y0gm7j1.jpg)







##### if-range

如果if-range的值与资源的ETag值（或者时间）一致，则服务器作为范围请求处理，反之返回全部资源。

![image-20190915154550062](http://ww3.sinaimg.cn/large/006y8mN6gy1g708zhve11j31840o4awf.jpg)

![image-20190915154605856](http://ww4.sinaimg.cn/large/006y8mN6gy1g708zs3ikwj318o0jcaro.jpg)

![image-20190915154659699](http://ww1.sinaimg.cn/large/006y8mN6gy1g7090pj19jj31590u0qv1.jpg)





##### if-unmodified-since

![image-20190916162600759](http://ww2.sinaimg.cn/large/006y8mN6gy1g71frnivd7j30lz01l0sq.jpg)

告诉服务器只有在指定时间后资源没有更新过，才处理请求。



##### max-forwards

![image-20190916170742808](http://ww1.sinaimg.cn/large/006y8mN6gy1g71gz00y83j30nr0clq9h.jpg)

发送Trace或者Options方法的请求时，使用max-forwards表示最多可以经过的服务器数目，请求转发一次则减1。当服务器接收到max-forwards为0的请求，不会转发请求，而是直接返回响应。





##### Proxy-Authorization

在发起某些请求之前，服务器要求客户端进行认证，客户端通过该字段携带认证信息给服务器。

![image-20190916172111291](http://ww4.sinaimg.cn/large/006y8mN6gy1g71hd0p03ej30mi01w3yj.jpg)





##### Range

告诉服务器需要获取资源的哪个范围的数据。

![image-20190916172259195](http://ww2.sinaimg.cn/large/006y8mN6gy1g71hew2va6j30lq01l0sn.jpg)

当服务器无法处理该范围请求时，会返回完整资源。





##### Refer

告诉服务器请求的来源。

![image-20190916172641238](http://ww1.sinaimg.cn/large/006y8mN6gy1g71hiqofxjj30mf0agjvt.jpg)

利用该字段可以实现拦截访问：判断refer是某个域名，才允许继续访问，否则拦截。参考：https://blog.csdn.net/shenqueying/article/details/79426884



##### TE

告诉服务器 客户端能够处理的**传输编码**方式。

![image-20190916180344698](http://ww3.sinaimg.cn/large/006y8mN6gy1g71ilaihpdj30ma020jrb.jpg)



##### User-Agent

用于给服务传递浏览器的名称等信息。

![image-20190916174706170](http://ww2.sinaimg.cn/large/006y8mN6gy1g71i3zca5ej30m401m0st.jpg)







#### 响应首部字段

响应报文中使用的首部字段



##### Accept-Range

服务器告诉客户端 其是否支持范围请求。

![image-20190916175647384](http://ww1.sinaimg.cn/large/006y8mN6gy1g71ie2740xj30mb095goo.jpg)

指令bytes表示支持范围请求，none表示不支持范围请求。





##### Age

![image-20190916184825185](http://ww3.sinaimg.cn/large/006y8mN6gy1g71jvrzvacj30m70amjvv.jpg)

当代理服务器使用缓存的实体去响应请求时，Age字段表示实体被创建到现在经过了多少时间，单位为秒。





##### ETag

![image-20190916190439123](http://ww2.sinaimg.cn/large/006y8mN6gy1g71kcoeyh1j30ly0cj791.jpg)

服务器会为每份资源分配对应的ETag值，唯一标识该资源。并且在响应中通过ETag字段返回给客户端。

当资源更新后，ETag值也会随之改变。

![image-20190916190821920](http://ww3.sinaimg.cn/large/006y8mN6gy1g71kgjencfj30mp09r0wy.jpg)

当资源被缓存时，就会被分配ETag值。



ETag值中有强ETag值和弱ETag值之分



###### 强ETag值

无论资源什么改变，都会改变该值

![image-20190916191155230](http://ww1.sinaimg.cn/large/006y8mN6gy1g71kk88wm9j30m301pa9y.jpg)



###### 弱ETag值

只有资源发生根本性改变，才会改变该值。会在值的最开始出添加W/

比如文件只是更新时间改了，但是文件内容并未变化，因此弱ETag值并未变化。

![image-20190916191224116](http://ww1.sinaimg.cn/large/006y8mN6gy1g71kkqlv2ej30m601rwee.jpg)





##### Location

![image-20190916191730074](http://ww3.sinaimg.cn/large/006y8mN6gy1g71kq1qd6xj30jg0h1wm8.jpg)

可以将客户端引导至Location指定的URI。通常该字段用在重定向中。





##### Proxy-Authenticate

![image-20190916191922548](http://ww2.sinaimg.cn/large/006y8mN6gy1g71krzwpvij30m9020weh.jpg)

Proxy-Authenticate会把代理服务器所要求的认证信息发送给客户端。







##### Retry-After

![image-20190916192106408](http://ww1.sinaimg.cn/large/006y8mN6gy1g71ktselscj30mf08njuv.jpg)

告诉客户端多久之后再次发送请求。





##### Server

![image-20190916192229879](http://ww3.sinaimg.cn/large/006y8mN6gy1g71kv936i4j30mw0dbaf6.jpg)

告诉客户端 当前服务器的信息





##### Vary

![image-20190916192324298](http://ww1.sinaimg.cn/large/006y8mN6gy1g71kw6p2urj30md0bx7b5.jpg)

首部字段Vary可以实现缓存的控制。源服务器通过该字段控制缓存服务器上缓存的使用。

Vary字段用于列出一个请求字段列表，告诉缓存服务器遇到同一个URL存在不同版本的情况下，如何筛选适合的版本（使用请求字段的值和响应字段的值进行对比，比如Accept-Language和Content-Language）。

参考：https://www.jianshu.com/p/5c601087c18e







##### WWW-Authenticate

![image-20190916193015357](http://ww1.sinaimg.cn/large/006y8mN6gy1g71l3b9pfnj30m701oweg.jpg)

告诉客户端对于请求URI应该使用的认证方案。













#### 实体首部字段

请求报文和响应报文实体部分使用的首部字段。



##### Allow

![image-20190917102021536](http://ww1.sinaimg.cn/large/006y8mN6gy1g72ath13k7j30m208qdju.jpg)

告诉客户端对于URI指定的资源，服务器支持哪些方法。

情景：若服务器返回状态码 [`405`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/405) `Method Not Allowed，则该首部字段亦需要同时返回给客户端。`



##### Content-Encoding

![image-20190917102827264](http://ww4.sinaimg.cn/large/006y8mN6gy1g72b1vypvnj30lo01hmx2.jpg)

![image-20190917102833945](http://ww2.sinaimg.cn/large/006y8mN6gy1g72b1znrpdj30dl06ijti.jpg)

告诉客户端，服务器按照何种**内容编码**方式对实体主体进行编码。

**内容编码**：不丢失实体信息的前提下进行的压缩。





##### Content-Language

![image-20190917103047520](http://ww1.sinaimg.cn/large/006y8mN6gy1g72b4bflsuj30m608s0ui.jpg)

告诉客户端，实体主体所使用的自然语言。





##### Content-Length

![image-20190917103148897](http://ww2.sinaimg.cn/large/006y8mN6gy1g72b5df8hlj30ls0800up.jpg)

告诉客户端 实体主体部分的大小（传输长度），单位是字节。

在HTTP协议中，消息实体长度和消息实体的传输长度是有区别，比如说gzip压缩下，消息实体长度是压缩前的长度，消息实体的传输长度是gzip压缩后的长度。

**客户端怎么知道实体主体的大小呢？（怎么知道下载完了）**

- 服务器已经知道实体主体的大小了，就直接通过Content-Length告诉客户端

  ```http
  // body的大小是1076B，你读取1076B就可以完成任务了
  Content-Length:1076
  // 注意Transfer-Encoding是null，两者不能共存
  Transfer-Encoding: null
  ```

- 服务器没法提前知道资源的大小，就会使用分块传输编码，最后一块大小是0，就传输完了

  ```http
  // 接下来的body我要一块一块的传，每一块开始是这一块的大小，等我传到大小为0的块时，就没了
  Content-Length:null
  Transfer-Encoding:chunked
  ```

- 服务器不知道资源的大小，同时也不支持chunked的传输模式，那么就既没有content-length头，也没有transfer-encoding头，这种情况下必须使用短连接，以连接结束来标示数据传输结束，传输结束就能知道大小了。这时候服务器返回的header里Connection一定是close。

  ```http
  // 我不知道大小，我也用不了chunked，啥时候我关了tcp连接，就说明传输结束了
  Content-Length:null
  Transfer-Encoding:null
  Connection:close
  ```





##### Content-Location

Content-Location表示 返回资源对应的URI。

![image-20190917111410548](http://ww4.sinaimg.cn/large/006y8mN6gy1g72cdg7utbj30lw01ydfu.jpg)

返回的资源内容与请求的不符时，会返回该字段。（比如请求英文，返回的是中文文档）





##### Content-MD5

![image-20190917111839978](http://ww2.sinaimg.cn/large/006y8mN6gy1g72ci4yuqzj30md0g0grc.jpg)

目的是校验报文传输过程中是否保持完整性。
不过这种方式是存在漏洞的，因为Content-MD5字段的值无法知道是否被串改。

![image-20190917112527823](http://ww3.sinaimg.cn/large/006y8mN6gy1g72cp74vfej30mf04labt.jpg)



##### Content-Range

![image-20190917112613332](http://ww1.sinaimg.cn/large/006y8mN6gy1g72cpzu8trj30m40gzak7.jpg)

范围请求时，返回实体主体的范围，单位字节。





##### Content-Type

![image-20190917113144096](http://ww2.sinaimg.cn/large/006y8mN6gy1g72cvq8bwrj30lz0bemy3.jpg)





##### Expires

![image-20190917113426681](http://ww3.sinaimg.cn/large/006y8mN6gy1g72cyjinj2j30lt0frtes.jpg)





##### Last-Modified

![image-20190917113514798](http://ww4.sinaimg.cn/large/006y8mN6gy1g72czdrwyaj30ly07cjv4.jpg)

![image-20190917113530089](http://ww3.sinaimg.cn/large/006y8mN6gy1g72czn31fjj30lo04qmyl.jpg)





#### 为Cookie服务的首部字段

##### Set-Cookie

![image-20190917113740118](http://ww3.sinaimg.cn/large/006y8mN6gy1g72d1wold4j30hu09uae2.jpg)

set-cookie字段的属性：

![image-20190917114046577](http://ww2.sinaimg.cn/large/006y8mN6gy1g72d54s3jqj30lu082go4.jpg)



###### expires

浏览器可发送的Cookie的有效期。

![image-20190917114657677](http://ww1.sinaimg.cn/large/006y8mN6gy1g72dbkeyjqj30l802iwfd.jpg)



###### path

可以访问该Cookie的目录。



###### domain

可以访问该Cookie的域名。如果设置为“.google.com”，则所有以“google.com”结尾的域名都可以访问该Cookie。注意第一个字符必须为“.”。



###### secure

该Cookie是否仅被使用安全协议传输。安全协议。安全协议有HTTPS，SSL等，在网络上传输数据之前先将数据加密。默认为false。

![image-20190917115417469](http://ww4.sinaimg.cn/large/006y8mN6gy1g72dj6toesj30kf07m760.jpg)

回收就是指Cookie失效。





###### HttpOnly

如果在Cookie中设置了"HttpOnly"属性，那么通过程序(JS脚本、Applet等)将无法读取到Cookie信息，这样能有效的防止XSS攻击。

![image-20190917115845779](http://ww2.sinaimg.cn/large/006y8mN6gy1g72dnudidnj30kp01lq2v.jpg)



##### Cookie

![image-20190917115958407](http://ww3.sinaimg.cn/large/006y8mN6gy1g72dp3y259j30kq04r3zi.jpg)





#### 其他首部字段

##### X-Frame-Options

属于HTTP响应字段，用于控制当前内容在其他web网站的Frame标签是怎么展示的。

主要目的是防止点击劫持攻击。

有2个可选的字段值：

- deny：拒绝
- SAMZEORIGIN：仅同域名下的页面匹配时许可。比如比如请求URI为www.baidu.com，那么只有www.baidu.com上的所有页面的frame允许加载页面内容，其他域名的页面不能加载内容。



**点击劫持**

点击劫持是指在一个Web页面下隐藏了一个透明的iframe（opacity：0），用外层假页面诱导用户点击，实际上是在隐藏的frame上触发了点击事件进行一些用户不知情的操作。

![image-20190919194837972](http://ww2.sinaimg.cn/large/006y8mN6gy1g752hdcpfyj30xe0i449g.jpg)

（BEST GAME这页面是用户看到的页面）



##### X-XSS-Protection

![image-20190919195059400](http://ww4.sinaimg.cn/large/006y8mN6gy1g752jt0vr6j30m201kdfq.jpg)

HTTP响应首部，用于控制浏览器防御XSS机制的开关。0关闭，1开启。



**XSS**

XSS攻击是Web攻击中最常见的攻击方法之一，它是通过对网页注入可执行代码且成功地被浏览器 执行，达到攻击的目的，形成了一次有效XSS攻击，一旦攻击成功，它可以获取用户的联系人列表，然后向联系人发送虚假诈骗信息，可以删除用户的日志等等，有时候还和其他攻击方式同时实 施比如SQL注入攻击服务器和数据库、Click劫持、相对链接劫持等实施钓鱼，它带来的危害是巨 大的，是web安全的头号大敌。



##### DNT

![image-20190919195327430](http://ww4.sinaimg.cn/large/006y8mN6gy1g752mdty5tj30mb07zadi.jpg)

属于HTTP请求首部。表示是否愿意被追踪。0同意被追踪，1不同意被追踪。

