#概要

logback是log4j作者推出的新日志系统，原生支持slf4j通用日志api，允许平滑切换日志系统，并且对简化应用部署中日志处理的工作做了有益的封装。



Logback日志需要依赖一下jar包：

slf4j-api-1.6.0.jar

logback-core-0.9.21.jar

logback-classic-0.9.21.jar

logback-access-0.9.21.jar



#配置

- Logback默认配置的步骤

1. 尝试在 classpath下查找文件logback-test.xml；
2. 如果文件不存在，则查找文件logback.xml；
3. 如果两个文件都不存在，logback用BasicConfigurator自动对自己进行配置，这会导致记录输出到控制台。



主配置文件为logback.xml，放在src目录下或是WEB-INF/classes下，logback会自动加载

logback.xml的基本结构如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="DEBUG"><appender-ref ref="STDOUT" /></root>
</configuration>
```

- logback.xml的基本配置信息都包含在configuration标签中，

- 需要含有至少一个appender标签用于指定日志输出方式和输出格式，

- root标签为系统默认日志进程，通过level指定日志级别，

- 通过appender-ref关联前面指定的日志输出方式。
- 例子中的appender使用的是ch.qos.logback.core.ConsoleAppender类，用于对控制台进行日志输出。其中encoder标签指定日志输出格式为“时间 线程 级别 类路径 信息”



## 日志分包

```xml
<appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>E:/logs/mylog.txt</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <!-- rollover daily -->
      <fileNamePattern>E:/logs/mylog-%d{yyyy-MM-dd_HH-mm}.%i.log</fileNamePattern>
      <maxHistory>5</maxHistory> 
      <timeBasedFileNamingAndTriggeringPolicy
            class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
        <!-- or whenever the file size reaches 100MB -->
        <maxFileSize>100MB</maxFileSize>
      </timeBasedFileNamingAndTriggeringPolicy>
    </rollingPolicy>
    <encoder>
      <pattern>%date %level [%thread] %logger{36} [%file : %line] %msg%n</pattern>
    </encoder>
  </appender
```

文件日志输出采用的ch.qos.logback.core.rolling.RollingFileAppender类，它的基本属性包括：

- file指定输入文件路径

- rollingPolicy标签指定的是日志分包策略。
  - ch.qos.logback.core.rolling.TimeBasedRollingPolicy类实现的是基于时间的分包策略，分包间隔是根据fileNamePattern中指定的事件最小单位，比如例子中的%d{yyyy-MM-dd_HH-mm}的最小事件单位为分，它的触发方式就是1种，**策略在每次向日志中添加新内容时触发**，如果满足条件，就将mylog.txt复制到E:/logs/目录并更名为mylog-2010-06-22_13-13.1.log，并删除原mylog.txt。
    - maxHistory：可选节点，控制保留的归档文件的最大数量，超出数量就删除旧文件，假设设置每个月滚动，且maxHistory 是 6，则只保存最近6个月的文件，删除之前的旧文件，注意：删除旧文件时，那些为了归档而创建的目录也会被删除。
  - 此外，策略还可以互相嵌套，比如本例中在时间策略中又嵌套了文件大小策略，
  - ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP实现对单文件大小的判断，当超过maxFileSize中指定大大小时，文件名中的变量%i会加一，即在不满足时间触发且满足大小触发时，会生成mylog-2010-06-22_13-13.1.log和mylog-2010-06-22_13-13.2.log两个文件。



参考:

<https://www.cnblogs.com/warking/p/5710303.html>

https://www.cnblogs.com/DeepLearing/p/5663178.html

https://blog.csdn.net/jiaincs/article/details/5686287