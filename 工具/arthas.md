# arthas



## 概述

arthas是阿里巴巴开源的一款Java诊断工具，可以帮忙开发人员快速定位问题。



## 问题

arthas可以定位的问题例如：

1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
5. 是否有一个全局视角来查看系统的运行状况？
6. 有什么办法可以监控到JVM的实时运行状态？
7. 怎么快速定位应用的热点，生成火焰图？



`Arthas`支持JDK 6+，支持Linux/Mac/Windows，采用命令行交互模式，同时提供丰富的 `Tab` 自动补全功能，进一步方便进行问题的定位和诊断。





## 常用命令



### logger

参考：https://arthas.aliyun.com/doc/logger.html



#### 查看logger信息

使用logger命令可以查看所有logger信息。

```java
[arthas@2062]$ logger
 name                                   ROOT
 class                                  ch.qos.logback.classic.Logger
 classLoader                            sun.misc.Launcher$AppClassLoader@2a139a55
 classLoaderHash                        2a139a55
 level                                  INFO
 effectiveLevel                         INFO
 additivity                             true
 codeSource                             file:/Users/hengyunabc/.m2/repository/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar
 appenders                              name            CONSOLE
                                        class           ch.qos.logback.core.ConsoleAppender
                                        classLoader     sun.misc.Launcher$AppClassLoader@2a139a55
                                        classLoaderHash 2a139a55
                                        target          System.out
                                        name            APPLICATION
                                        class           ch.qos.logback.core.rolling.RollingFileAppender
                                        classLoader     sun.misc.Launcher$AppClassLoader@2a139a55
                                        classLoaderHash 2a139a55
                                        file            app.log
                                        name            ASYNC
                                        class           ch.qos.logback.classic.AsyncAppender
                                        classLoader     sun.misc.Launcher$AppClassLoader@2a139a55
                                        classLoaderHash 2a139a55
                                        appenderRef     [APPLICATION]
```

对应的logback.xml为：

```java
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="APPLICATION" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>mylog-%d{yyyy-MM-dd}.%i.txt</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>60</maxHistory>
            <totalSizeCap>2GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%logger{35} - %msg%n</pattern>
        </encoder>
    </appender>
 
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="APPLICATION" />
    </appender>
 
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg %n
            </pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>
 
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="ASYNC" />
    </root>
</configuration>
```



#### 更新日志级别

```java
[arthas@2062]$ logger --name ROOT --level debug
update logger level success.
```

更新ROOT的日志级别为debug。



#### 指定classloader更新 logger level

默认情况下，logger命令会在SystemClassloader下执行，如果应用是传统的`war`应用，或者spring boot fat jar启动的应用，那么需要指定classloader。

可以先用 `sc -d yourClassName` 来查看具体的 classloader hashcode，然后在更新level时指定classloader：

```java
[arthas@2062]$ logger -c 2a139a55 --name ROOT --level debug
```





### heapdump

生成快照，参考：https://arthas.aliyun.com/doc/heapdump.html





### watch

观察方法执行情况，比如说入参、返回值



#### 相关参数

| *class-pattern*     | 类名表达式匹配                             |
| ------------------- | ------------------------------------------ |
| *method-pattern*    | 方法名表达式匹配                           |
| *express*           | 观察表达式                                 |
| *condition-express* | 条件表达式                                 |
| [b]                 | 在**方法调用之前**观察                     |
| [e]                 | 在**方法异常之后**观察                     |
| [s]                 | 在**方法返回之后**观察                     |
| [f]                 | 在**方法结束之后**(正常返回和异常返回)观察 |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配       |
| [x:]                | 指定输出结果的属性遍历深度，默认为 1       |



#### 观察方法调用前的参数，调用后的参数

示例：`watch demo.MathGame primeFactors "{params,target,returnObj}" -x 2 -b -s -n 2`

格式：`watch 类的全限定名 方法名 表达式`

![image-20201027163945909](https://tva1.sinaimg.cn/large/0081Kckwgy1gk3zde9rwxj30ui0h90vy.jpg)

![image-20201027164223920](https://tva1.sinaimg.cn/large/0081Kckwgy1gk3zg37k44j31aj0gfdiq.jpg)



#### 观察对象的属性

访问整个对象示例，target表示对象： `watch demo.MathGame primeFactors 'target'`

访问对象的某个属性示例： `watch demo.MathGame primeFactors 'target.illegalArgumentCount'`

- 格式： watch 类全限定名 方法名 'target.属性名称'
- watch只可以访问成员变量
- 静态变量 无论是public，还是private都不能用watch访问。

参考：https://arthas.aliyun.com/doc/watch.html





### trace

方法内部调用路径，并输出方法路径上的每个节点上耗时。

参考：https://arthas.aliyun.com/doc/trace.html





### stack

输出当前方法被调用的调用路径。

参考：https://arthas.aliyun.com/doc/stack.html





### tt

记录下指定方法每次调用的入参和返回信息，并能对不同时间下的调用进行观测

- 首先查看方法的每次调用情况  `tt -t demo.MathGame primeFactors`
- 查看单次调用的入参、出参  `tt -i 1003`   
  - 1003是Index，即时间片段记录编号

 参考：https://arthas.aliyun.com/doc/tt.html





### profiler

`profiler` 命令支持生成应用热点的火焰图。本质上是通过不断的采样，然后把收集到的采样结果生成火焰图。

参考：https://arthas.aliyun.com/doc/profiler.html









## 后台异步任务

参考：https://arthas.aliyun.com/doc/async.html





## 扩展



### 火焰图

火焰图是展示CPU调用栈的图片。

![image-20201026170335941](https://tva1.sinaimg.cn/large/0081Kckwgy1gk2ufuj4rnj30j60aygsh.jpg)

y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。

x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。注意，x 轴不代表时间，而是所有的调用栈合并后，按字母顺序排列的。

**火焰图就是看顶层的哪个函数占据的宽度最大。只要有"平顶"（plateaus），就表示该函数可能存在性能问题。**

颜色没有特殊含义，因为火焰图表示的是 CPU 的繁忙程度，所以一般选择暖色调。



火焰的每一层都会标注函数名，鼠标悬浮时会显示完整的函数名、抽样抽中的次数、占据总抽样次数的百分比。下面是一个例子。

```java
mysqld'JOIN::exec (272,959 samples, 78.34 percent)
```



在某一层点击，火焰图会水平放大，该层会占据所有宽度，显示详细信息。

![image-20201026170801978](https://tva1.sinaimg.cn/large/0081Kckwgy1gk2ukgnfvnj30hp0acwio.jpg)

左上角会同时显示"Reset Zoom"，点击该链接，图片就会恢复原样。



按下 Ctrl + F 会显示一个搜索框，用户可以输入关键词或正则表达式，所有符合条件的函数名会高亮显示。









## 参考

[arthas官方](https://arthas.aliyun.com/doc/)

[学习-火焰图](http://www.ruanyifeng.com/blog/2017/09/flame-graph.html)