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



### cpu load高的排查思路

和排查CPU打满了排查思路是一样的。

一般的原因可能为：

- Full GC次数过多
- 代码中存在死循环



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





## 参考
https://www.cnblogs.com/gbyukg/p/3326825.html
https://www.cnblogs.com/zongfa/p/7967935.html
https://www.cnblogs.com/intval/p/5763929.html

[cpu load过高问题排查](https://www.cnblogs.com/lddbupt/p/5779655.html)

[grep用法参考](https://www.cnblogs.com/leo-li-3046/p/5690613.html)