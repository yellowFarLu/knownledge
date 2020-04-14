# Linux命令





ssh
---
登录远程服务器，shh 用户名@IP地址，例如 `ssh huangy@10.111.32.21`。登录之后，如果想退出，可以使用`logout`退出。
常用参数：
（1）-p : 指定远程服务器的端口





tail
---

从末尾查看文件，常用`tail -f XXXX`
常用参数：
（1）-f : 查看文件的新添加的内容
（2）-n : n可以是任意数字，查看从末尾开始的n行





head
---

从头查看文件，常用`head -100 XXXX`
常用参数：
（1）-n : n可以是任意数字，查看从头开始的n行





ps
--- 

查看名称对应的进程，常用`ps aux | grep XXX`，ps aux按照指定格式打印进程信息。

ps aux输出格式：

`USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND`

格式说明：
USER: 行程拥有者
PID: pid
%CPU: 占用的 CPU 使用率
%MEM: 占用的记忆体使用率
VSZ: 占用的虚拟记忆体大小
RSS: 占用的记忆体大小
TTY: 终端的次要装置号码 (minor device number of tty)

STAT: 该行程的状态，linux的进程有5种状态：
    D 不可中断 uninterruptible sleep (usually IO)
    R 运行 runnable (on run queue)
    S 中断 sleeping
    T 停止 traced or stopped
    Z 僵死 a defunct (”zombie”) process
        注: 其它状态还包括W(无驻留页), <(高优先级进程), N(低优先级进程), L(内存锁页).

START: 行程开始时间 
TIME: 执行的时间 
COMMAND:所执行的指令





free
---

查看**机器**内存使用情况，常用`free -m`
真正未用到的内存数（可用内存）：free+buffers+cached  的值，也就是+ buffers/cache。如果这个值太小，说明内存不足了。可以考虑把其他较小的项目内存弄小
老版本的linux，没有+ buffers/cache字段，可以使用available字段观察





## top



### 概念

查看**进程**内存和CPU的使用情况，
%CPU 上次更新到现在的CPU时间占用百分比
%MEM 进程使用的物理内存百分比



top命令中load average显示的是最近1分钟、5分钟和15分钟的系统平均负载。

系统平均负载被定义为在**特定时间间隔内**运行队列中(**在CPU上运行或者等待运行的进程数**)的平均进程数。

**系统的load是指正在运行和准备好运行的进程的总数。**比如现在系统有2个正在运行的进程，3个可运行进程，那么系统的load就是5。load average就是一段时间内的load数量。

Load Average 就是一段时间(1min,5min,15min)内平均Load。**平均负载的最佳值是1**，这意味着每个进程都可以在一个完整的CPU周期内完成。



### 判别和处理load高问题

一般根据cpu数量去判断,也就是Load平均要小于CPU的数量。

负载的正常值在不同的系统中有着很大的差别。在单核处理器的工作站中，1或2都是可以接受的。多核处理器的服务器(比如24核)上，load 会到达20 ，甚至更高。













lsof
---

查看文件的打开情况





scp
---

下载文件到本地，常用 `scp 登录名@IP:路径 本地路径`





zcat
---

查看压缩包内容，常和grep一起使用，`zcat 文件名 | grep '查找的内容' --color`





cat
---

查看文件内容，常和grep一起使用，`cat 文件名 | grep '查找的内容' --color`





grep
---

1、匹配文本内容，常用`grep -E '查找的内容'` 文件名。
    参数：
    --color 把匹配的内容显示为红色
    -E 使用正则匹配
    -A10 显示匹配行后面10行
    -B10 显示匹配行前面10行
    -C10 显示匹配行前后10行 
    -c  显示匹配行的计数
2、grep实现**and**语义：`grep 'pattern1' filename | grep 'pattern2' `，不过一般情况下，搜索日志需要搜索整个文件，因此使用cat和grep搭配使用：`cat filename | grep 'pattern1' | grep 'pattern2'`
3、假如一页无法显示完，需要grep、cat、more结合使用，例如 `cat install.log | grep “i686”| more`。
（1）在`more 文件名`下，`空格`向后一页，`ctrl + B`往前一页。在`cat install.log | grep “i686”| more`情况下，无法使用`ctrl + B`往前一页
（2）在这种情况下，推荐使用`cat test.text | grep -C100 '2' | less`，可以达到more一样的效果，`d`往后翻页，`b`往前翻页





curl
---

1、默认模拟get请求：curl -u username https://api.github.com/user?access_token=XXXXXXXXXX

使用这种形式，可以拼接多个参数 
curl 'http://eservice.nsvc.foneshare.cn/eservice/canon/customized/sendShortMsg?code=Y2Fubm9ua2V5&fsEa=fktest085&mobile=13527206719&content=%E4%BD%B3%E8%83%BD%E6%B5%8B%E8%AF%95'

2、模拟post请求：curl -u username --data "param1=value1&param2=value" https://api.github.com





iptables
--- 

1、使用`iptables -nvL`查看防火墙开放的端口

![1085580279-5b5f2bbc8b767_articlex](https://tva1.sinaimg.cn/large/006tNbRwgy1gaimmzsq9aj30m809tjwj.jpg)

如图: dpt:9001表示9001端口开放； dpts:31000:38000表示31000到38000之间的端口开放

2、开放端口：

```linux
// 开放22端口
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

// 开放范围的端口
iptables -A INPUT -p tcp --dport 4800:4900 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 4800:4900 -j ACCEPT

// 保存配置：
/etc/rc.d/init.d/iptables save
               
// 重启服务：
/etc/init.d/iptables restart
```





netstat
--- 
1、使用`netstat  -anp  |grep   端口号`，如果对应端口显示情况如下：
![3042069176-5b5dcaa900cdf_articlex](https://tva1.sinaimg.cn/large/006tNbRwgy1gaimnmmiqfj30m801aaaa.jpg)
如图，表示3306端已经被占用





telnet
---

1、登录远程：`telnet ip port`，比如说`telnet localhost 8080`





su
---

使用su命令，可以切换到其他账号  `su XXXX`





crontab
---

`crontab -l` 查看当前用户的定时任务
`crontab -e` 创建并编辑一个定时任务
参考：https://www.cnblogs.com/intval/p/5763929.html





df
---

`df -H` 查看每个目录下磁盘的使用情况
![3432986146-5c9f4cf8a0257_articlex](https://tva1.sinaimg.cn/large/006tNbRwgy1gaimo8s6zsj30m808hgn9.jpg)







## sort

sort将文件的每一行作为一个单位，相互比较，比较原则是从首字符向后，依次按ASCII码值进行比较，最后将他们按升序输出。



### sort的-u选项

```
[rocrocket@rocrocket programming]$ cat seq.txt
banana
apple
pear
orange
pear

[rocrocket@rocrocket programming]$ sort seq.txt
apple
banana
orange
pear
pear

[rocrocket@rocrocket programming]$ sort -u seq.txt
apple
banana
orange
pear
```

pear由于重复被-u选项无情的删除了



### sort的-r选项

sort默认的排序方式是升序，如果想改成降序，就加个-r就搞定了。

```
[rocrocket@rocrocket programming]$ cat number.txt
1
3
5
2
4

[rocrocket@rocrocket programming]$ sort number.txt
1
2
3
4
5

[rocrocket@rocrocket programming]$ sort -r number.txt
5
4
3
2
1
```





### sort的-o选项

由于sort默认是把结果输出到标准输出，所以需要用重定向才能将结果写入文件，形如sort filename > newfile。

但是，如果你想把排序结果输出到原文件中，用重定向可就不行了。

```java
[rocrocket@rocrocket programming]$ sort -r number.txt > number.txt
[rocrocket@rocrocket programming]$ cat number.txt
[rocrocket@rocrocket programming]$
```

看，竟然将number清空了。

就在这个时候，-o选项出现了，它成功的解决了这个问题，让你放心的将结果写入原文件。这或许也是-o比重定向的唯一优势所在。

```
[rocrocket@rocrocket programming]$ cat number.txt
1
3
5
2
4
  
[rocrocket@rocrocket programming]$ sort -r number.txt -o number.txt

[rocrocket@rocrocket programming]$ cat number.txt
5
4
3
2
1
```



### sort的-n选项

你有没有遇到过10比2小的情况。我反正遇到过。出现这种情况是由于排序程序将这些数字按字符来排序了，排序程序会先比较1和2，显然1小，所以就将10放在2前面喽。这也是sort的一贯作风。

我们如果想改变这种现状，就要使用-n选项，来告诉sort，“要以数值来排序”！

```
[rocrocket@rocrocket programming]$ cat number.txt
1
10
19
11
2
5

[rocrocket@rocrocket programming]$ sort number.txt
1
10
11
19
2
5

[rocrocket@rocrocket programming]$ sort -n number.txt
1
2
5
10
11
19
```



### sort的-t选项和-k选项

如果有一个文件的内容是这样：

```
[rocrocket@rocrocket programming]$ cat facebook.txt
banana:30:5.5
apple:10:2.5
pear:90:2.3
orange:20:3.4
```

这个文件有三列，列与列之间用冒号隔开了，第一列表示水果类型，第二列表示水果数量，第三列表示水果价格。

那么我想以水果数量来排序，也就是以第二列来排序，如何利用sort实现？

幸好，sort提供了-t选项，后面可以设定间隔符。（是不是想起了cut和paste的-d选项，共鸣～～）

指定了间隔符之后，就可以用-k来指定列数了。

```
[rocrocket@rocrocket programming]$ sort -n -k 2 -t : facebook.txt
apple:10:2.5
orange:20:3.4
banana:30:5.5
pear:90:2.3
```

我们使用冒号作为间隔符，并针对第二列来进行数值升序排序，结果很令人满意。



### 其他的sort常用选项

-f会将小写字母都转换为大写字母来进行比较，亦即忽略大小写

-c会检查文件是否已排好序，如果乱序，则输出第一个乱序的行的相关信息，最后返回1

-C会检查文件是否已排好序，如果乱序，不输出内容，仅返回1

-M会以月份来排序，比如JAN小于FEB等等

-b会忽略每一行前面的所有空白部分，从第一个可见字符开始比较。





## uniq

uniq用于删除文件中重复的行。

uniq 命令读取由 InFile 参数指定的标准输入或文件。该命令首先比较相邻的行，然后除去第二行和该行的后续副本。重复的行一定相邻。（在发出 uniq 命令之前，请使用 sort 命令使所有重复行相邻。）最后，uniq 命令将最终单独的行写入标准输出或由 OutFile 参数指定的文件。InFile 和 OutFile 参数必须指定不同的文件。如果输入文件用“- ”表示，则从标准输入读取；输入文件必须是文本文件。文本文件是包含组织在一行或多行中的字符的文件。这些行的长度不能超出 2048 个字节（包含所有换行字符），并且其中不能包含空字符。

缺省情况下，uniq 命令比较所有行。如果指定了-f Fields 或 -Fields 标志, uniq 命令忽略由 Fields 变量指定的字段数目。 field 是一个字符串，用一个或多个 <空格 > 字符将它与其它字符串分隔开。

如果指定了 -s Characters 或 -Characters 标志, uniq 命令忽略由 Characters 变量指定的字段数目。为 Fields 和 Characters 变量指定的值必须是正的十进制整数。

当前本地语言环境决定了 -f 标志使用的 <空白> 字符以及 -s 标志如何将字节解释成字符。

如果执行成功，uniq 命令退出，返回值 0。否则，命令退出返回值大于 0。



### uniq参数

-c 在输出行前面加上每行在输入文件中出现的次数。

-d 仅显示重复行。

-u 仅显示不重复的行。

-f Fields 忽略由 Fields 变量指定的字段数目。如果 Fields 变量的值超过输入行中的字段数目, uniq 命令用空字符串进行比较。这个标志和 -Fields 标志是等价的。

-s Characters 忽略由 Characters 变量指定的字符的数目。如果 Characters 变量的值超过输入行中的字符的数目, uniq 用空字符串进行比较。如果同时指定 -f 和 -s 标志, uniq 命令忽略由 -s Characters 标志指定的字符的数目，而从由 -f Fields 标志指定的字段后开始。 这个标志和 +Characters 标志是等价的。

-Fields 忽略由 Fields 变量指定的字段数目。这个标志和 -f Fields 标志是等价的。

+Characters 忽略由 Characters 变量指定的字符的数目。如果同时指定 - Fields 和 +Characters 标志, uniq 命令忽略由 +Characters 标志指定的字符数目，并从由 -Fields 标志指定的字段后开始。 这个标志和 -s Characters 标志是等价的。

\- c 显示输出中，在每行行首加上本行在文件中出现的次数。它可取代- u和- d选项。

\- d 只显示重复行 。

\- u 只显示文件中不重复的各行 。

\- n 前n个字段与每个字段前的空白一起被忽略。一个字段是一个非空格、非制表符的字符串，彼此由制表符和空格隔开（字段从0开始编号）。

\+ n 前n个字符被忽略，之前的字符被跳过（字符从0开始编号）。

\- f n 与- n相同，这里n是字段数。

\- s n 与＋n相同，这里n是字符数。





## awk

awk可以将文本格式化成我们想要的样子。

awk的基本语法如下：

`awk [options] 'Pattern{Action}' file`

从字面上理解 ，action指的就是动作，awk擅长文本格式化，并且将格式化以后的文本输出，所以awk最常用的动作就是print和printf，因为awk要把格式化完成后的文本输出啊，所以，这两个动作最常用。

我们先从最简单用法开始了解awk，我们先不使用[options] ,也不指定pattern，直接使用最简单的action，从而开始认识awk，示例如下：

![image-20200414221911816](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdtnq5z5cbj30ic04g3z6.jpg)

上图中，我们只是使用awk执行了一个打印的动作，将testd文件中的内容打印了出来。



好了，现在我们来操作一下另一个类似的场景。

![image-20200414222039576](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdtnrn613ej30rw0fawjg.jpg)

上图中的示例没有使用到options和pattern，上图中的awk '{print \$5}'，表示输出df的信息的第5列，$5表示将当前行按照分隔符分割后的第5列，不指定分隔符时，默认使用空格作为分隔符。

细心的你一定发现了，上述信息用的空格不止有一个，而是有连续多个空格，awk自动将连续的空格理解为一个分割符了，是不是比cut命令要简单很多，这样比较简单的例子，有利于我们开始了解awk。

awk是逐行处理的，逐行处理的意思就是说，当awk处理一个文本时，会一行一行进行处理，处理完当前行，再处理下一行，awk默认以"换行符"为标记，识别每一行，也就是说，awk跟我们人类一样，每次遇到"回车换行"，就认为是当前行的结束，新的一行的开始，awk会按照用户指定的分割符去分割当前行，如果没有指定分割符，默认使用空格作为分隔符。

![image-20200414222817161](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdtnzkmzp1j312w0b0jvq.jpg)

0 表示显示整行 ，\$NF表示当前行分割后的最后一列（​\$0和\$NF均为内置变量）

注意，\$NF 和 NF 要表达的意思是不一样的，对于awk来说，$NF表示最后一个字段，NF表示当前行被分隔符切开以后，一共有几个字段。

也就是说，假如一行文本被空格分成了7段，那么NF的值就是7，\$NF的值就是\$7,  而\$7表示当前行的第7个字段，也就是最后一列，那么每行的倒数第二列可以写为$(NF-1)。

我们也可以一次输出多列，使用逗号隔开要输出的多个列，如下，一次性输出第一列和第二列

![image-20200414222833058](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdtnzv1xz5j30lo074dhu.jpg)
同理，也可以一次性输出多个指定的列，如下图

![image-20200414222844355](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto05oe36j30na07gmz8.jpg)
我们发现，第一行并没有第5列，所以并没有输出任何文本，而第二行有第五列，所以输出了。



除了输出文本中的列，我们还能够添加自己的字段，将自己的字段与文件中的列结合起来，如下做法，都是可以的。

![image-20200414222928774](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto0toxzoj30x20hw7hc.jpg)

从上述实验中可以看出，awk可以灵活的将我们指定的字符与每一列进行拼接，或者把指定的字符当做一个新列插入到原来的列中，也就是awk格式化文本能力的体现。

但是要注意，\$1这种内置变量的外侧不能加入双引号，否则$1会被当做文本输出，示例如下

![image-20200414223016262](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto1n8kr3j30qu0eo42f.jpg)

我们也可以输出整行，比如，如下两种写法都表示输出整行。

![image-20200414223038874](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto2162plj30lc09wgof.jpg)



我们说过，awk的语法如下

awk [options] 'Pattern{Action}' file

而且我们说过awk是逐行处理的， 刚才已经说过了最常用的Action：print

现在，我们来认识下一Pattern，也就是我们所说的模式

不过，我们准备先把awk中最特殊的模式展示给大家，以后再介绍普通的模式，因为普通模式需要的篇幅比较长，所以我们先来总结特殊模式。

AWK 包含两种特殊的模式：BEGIN 和 END。

BEGIN 模式指定了处理文本之前需要执行的操作：

END 模式指定了处理完所有行之后所需要执行的操作：

什么意思呢？光说不练不容易理解，我们来看一些小例子，先从BEGIN模式开始，示例如下

![image-20200414223149839](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto39qs4cj30q8066mz3.jpg)

上述写法表示，在开始处理test文件中的文本之前，先执行打印动作，输出的内容为"aaa","bbb".

也就是说，上述示例中，虽然指定了test文件作为输入源，但是在开始处理test文本之前，需要先执行BEGIN模式指定的"打印"操作

既然还没有开始逐行处理test文件中的文本，那么是不是根本就不需要指定test文件呢，我们来试试。

![image-20200414223219301](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto3s3ycqj30nc038t97.jpg)

经过实验发现，还真是，我们并没有给定任何输入来源，awk就直接输出信息了，因为，BEGIN模式表示，在处理指定的文本之前，需要先执行BEGIN模式中指定的动作，而上述示例没有给定任何输入源，但是awk还是会先执行BEGIN模式指定的"打印"动作，打印完成后，发现并没有文本可以处理，于是就只完成了"打印 aaa bbb"的操作。

这个时候，如果我们想要awk先执行BEGIN模式指定的动作，再根据执我们自定义的动作去操作文本，该怎么办呢？示例如下

![image-20200414223252579](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto4cno4kj30vw08cmzl.jpg)

上图中，蓝色标注的部分表示BEGIN模式指定的动作，这部分动作需要在处理指定的文本之前执行，所以，上图中先打印出了"aaa bbb"，当BEGIN模式对应的动作完成后，在使用后面的动作处理对应的文本，即打印test文件中的第一列与第二列，这样解释应该比较清楚了吧。

看完上述示例，似乎更加容易理解BEGIN模式是什么意思了，BEGIN模式的作用就是，在开始逐行处理文本之前，先执行BEGIN模式所指定的动作。以此类推，END模式的作用就一目了然了，举例如下。

![image-20200414223327932](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto4yn9q7j30ww090wgw.jpg)

聪明如你一定明白了，END模式就是在处理完所有的指定的文本之后，需要指定的动作。

那么，我们可以结合BEGIN模式和END模式一起使用。示例如下

![image-20200414223401650](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdto5jqkmxj310u07qq77.jpg)

上述示例中返回的结果有没有很像一张"报表"，有"表头"  、"表内容"、  "表尾"，awk对文本的格式化能力你体会到了吗？









## 参考

[awk基础](http://www.zsythink.net/archives/1336)

[uniq命令详解](https://blog.csdn.net/jesseen/article/details/8005056)

[sort命令详解](https://www.cnblogs.com/51linux/archive/2012/05/23/2515299.html)

https://www.cnblogs.com/gbyukg/p/3326825.html
https://www.cnblogs.com/zongfa/p/7967935.html
https://www.cnblogs.com/intval/p/5763929.html

[cpu load过高问题排查](https://www.cnblogs.com/lddbupt/p/5779655.html)

[grep用法参考](https://www.cnblogs.com/leo-li-3046/p/5690613.html)

