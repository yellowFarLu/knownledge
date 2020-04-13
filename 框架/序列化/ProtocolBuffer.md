# Protocol Buffer



## 概述

什么是 Google Protocol Buffer？ 

Google Protocol Buffer( 简称 Protobuf) 是 Google 公司内部的混合语言数据标准，目前已经正在使用的有超过 48,162 种报文格式定义和超过 12,183 个 .proto 文件。他们用于 RPC 系统和持续数据存储系统。

Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。目前提供了 C++、Java、Python 三种语言的 API。





## 实战

```java
import io.protostuff.LinkedBuffer;
import io.protostuff.ProtobufIOUtil;
import io.protostuff.ProtostuffIOUtil;
import io.protostuff.Schema;
import io.protostuff.runtime.RuntimeSchema;

public class ProtoBufUtil {

    public static <T> byte[] serializer(T o) {
        Schema schema = RuntimeSchema.getSchema(o.getClass());
        return ProtobufIOUtil.toByteArray(o, schema, LinkedBuffer.allocate(256));
    }

    public static <T> T deserializer(byte[] bytes, Class<T> clazz) {

        T obj = null;
        try {
            obj = clazz.newInstance();
            Schema schema = RuntimeSchema.getSchema(obj.getClass());
            ProtostuffIOUtil.mergeFrom(bytes, obj, schema);
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

        return obj;
    }
}
```

```java
import io.protostuff.Tag;

public class Student {

    @Tag(1)
    private String name;
    @Tag(2)
    private String studentNo;
    @Tag(3)
    private int age;
    @Tag(4)
    private String schoolName;

    // 关于@Tag,要么所有属性都有@Tag注解,要么都没有,不能一个类中只有部分属性有@Tag注解

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getStudentNo() {
        return studentNo;
    }

    public void setStudentNo(String studentNo) {
        this.studentNo = studentNo;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getSchoolName() {
        return schoolName;
    }

    public void setSchoolName(String schoolName) {
        this.schoolName = schoolName;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", studentNo='" + studentNo + '\'' +
                ", age=" + age +
                ", schoolName='" + schoolName + '\'' +
                '}';
    }
}
```

```java
import java.util.Arrays;


public class ProtoBufUtilTest {

    public static void main(String[] args) {

        Student student = new Student();
        student.setName("lance");
        student.setAge(28);
        student.setStudentNo("2011070122");
        student.setSchoolName("BJUT");

        byte[] serializerResult = ProtoBufUtil.serializer(student);

        System.out.println("serializer result:" + Arrays.toString(serializerResult));

        Student deSerializerResult = ProtoBufUtil.deserializer(serializerResult,Student.class);

        System.out.println("deSerializerResult:" + deSerializerResult.toString());
    }

}
```

需要引入maven依赖

```java
    <dependency>
      <groupId>io.protostuff</groupId>
      <artifactId>protostuff-core</artifactId>
      <version>1.4.0</version>
    </dependency>

    <dependency>
      <groupId>io.protostuff</groupId>
      <artifactId>protostuff-runtime</artifactId>
      <version>1.4.0</version>
    </dependency>
```







## 优点

- **生成字节数比较小，性能高**
  - 因为字节流中不存储字段类型信息，并且存储内容所使用的字节数是根据值的大小来决定的。
- 支持跨语言





## 原理



### 编码

有两项技术保证了采用 Protobuf 的程序能获得相对于 XML 极大的性能提高。

- 第一点，我们可以考察 Protobuf 序列化后的信息内容。您可以看到 Protocol Buffer 信息的表示非常紧凑，这意味着消息的体积减少，自然需要更少的资源。比如网络上传输的字节数更少，需要的 IO 更少等，从而提高性能。
- ProtoBuf的编码非常巧妙



Protobuf 序列化后所生成的二进制消息非常紧凑，这得益于 Protobuf 采用的非常巧妙的 编码 方法。

考察消息结构之前，让我首先要介绍一个叫做 Varint 的术语。

Varint 是一种紧凑的表示数字的方法。它用一个或多个字节来表示一个数字，**值越小的数字使用越少的字节数。这能减少用来表示数字的字节数**。

比如对于 int32 类型的数字，一般需要 4 个 byte 来表示。但是采用 Varint，对于很小的 int32 类型的数字，则可以用 1 个 byte 来表示。当然凡事都有好的也有不好的一面，采用 Varint 表示法，大的数字则需要 5 个 byte 来表示。从统计的角度来说，一般不会所有的消息中的数字都是大数，因此大多数情况下，采用 Varint 后，可以用更少的字节数来表示数字信息。

**Varint 中的每个 byte 的最高位 bit 有特殊的含义**，如果该位为 1，表示后续的 byte 也是该数字的一部分，如果该位为 0，则结束。其他的 7 个 bit 都用来表示数字。因此小于 128 的数字都可以用一个 byte 表示。大于 128 的数字，比如 300，会用两个字节来表示：1010 1100 0000 0010。

下图演示了 Google Protocol Buffer 如何解析两个 bytes。注意到最终计算前将两个 byte 的位置相互交换过一次，这是因为 Google Protocol Buffer 字节序采用 little-endian 的方式。

<img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbx935mbrnj30hy0cidli.jpg" alt="image-20200215181419644" style="zoom:50%;" />

消息经过序列化后会成为一个二进制数据流，该流中的数据为一系列的 Key-Value 对。如下图所示：

<img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbx93ira7wj30s00au78v.jpg" alt="image-20200215181441012" style="zoom:50%;" />

**采用这种 Key-Pair 结构无需使用分隔符来分割不同的 Field（Field即key-value对）**。对于可选的 Field，如果消息中不存在该 field，那么在最终的 Message Buffer 中就没有该 field，这些特性都有助于节约消息本身的大小。

以代码清单 1 中的消息为例。假设我们生成如下的一个消息 Test1:

```java
Test1.id = 10; 
Test1.str = “hello”；
```

则最终的 Message Buffer 中有两个 Key-Value 对，一个对应消息中的 id；另一个对应 str。

Key 用来标识具体的 field，在解包的时候，Protocol Buffer 根据 Key 就可以知道相应的 Value 应该对应于消息中的哪一个 field。

Key 的定义如下：

```java
(field_number << 3) | wire_type
```

可以看到 Key 由两部分组成。第一部分是 field_number，比如消息 lm.helloworld 中 field id 的 field_number 为 1。第二部分为 wire_type。表示 Value 的传输类型。

Wire Type 可能的类型如下表所示：

<img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbx959z0xpj318w0mwmzp.jpg" alt="image-20200215181622297" style="zoom:80%;" />

在我们的例子当中，field id 所采用的数据类型为 int32，因此对应的 wire type 为 0。细心的读者或许会看到在 Type 0 所能表示的数据类型中有 int32 和 sint32 这两个非常类似的数据类型。Google Protocol Buffer 区别它们的主要意图也是为了减少 encoding 后的字节数。

在计算机内，一个负数一般会被表示为一个很大的整数，因为计算机定义负数的符号位为数字的最高位。如果采用 Varint 表示一个负数，那么一定需要 5 个 byte。为此 Google Protocol Buffer 定义了 sint32 这种类型，采用 zigzag 编码。

Zigzag 编码用无符号数来表示有符号数字，正数和负数交错，这就是 zigzag 这个词的含义了。

如图所示：

<img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbx95wg0dlj30k80e6afz.jpg" alt="image-20200215181658121" style="zoom:50%;" />

使用 zigzag 编码，绝对值小的数字，无论正负都可以采用较少的 byte 来表示，充分利用了 Varint 这种技术。

其他的数据类型，比如字符串等则采用类似数据库中的 varchar 的表示方法，即用一个 varint 表示长度，然后将其余部分紧跟在这个长度部分之后即可。

通过以上对 protobuf Encoding 方法的介绍，想必您也已经发现 protobuf 消息的内容小，适于网络传输。假如您对那些有关技术细节的描述缺乏耐心和兴趣，那么下面这个简单而直观的比较应该能给您更加深刻的印象。

对于代码清单 1 中的消息，用 Protobuf 序列化后的字节序列为：

```java
08 65 12 06 48 65 6C 6C 6F 77
```

而如果用 XML，则类似这样：

```java
31 30 31 3C 2F 69 64 3E 3C 6E 61 6D 65 3E 68 65 
 6C 6C 6F 3C 2F 6E 61 6D 65 3E 3C 2F 68 65 6C 6C 
 6F 77 6F 72 6C 64 3E 
 
一共 55 个字节，这些奇怪的数字需要稍微解释一下，其含义用 ASCII 表示如下：
 <helloworld> 
    <id>101</id> 
    <name>hello</name> 
 </helloworld>
```





### 解码

首先我们来了解一下 XML 的封解包过程。XML 需要从文件中读取出字符串，再转换为 XML 文档对象结构模型。之后，再从 XML 文档对象结构模型中读取指定节点的字符串，最后再将这个字符串转换成指定类型的变量。这个过程非常复杂，其中将 XML 文件转换为文档对象结构模型的过程通常需要完成词法文法分析等大量消耗 CPU 的复杂计算。

反观 Protobuf，它只需要简单地将一个**二进制序列，按照指定的格式读取到对应的结构类型中**就可以了。从上一节的描述可以看到消息的 decoding 过程也可以通过几个位移操作组成的表达式计算即可完成。速度非常快。

为了说明这并不是我拍脑袋随意想出来的说法，下面让我们简单分析一下 Protobuf 解包的代码流程吧。

以代码清单 3 中的 Reader 为例，该程序首先调用 msg1 的 ParseFromIstream 方法，这个方法解析从文件读入的二进制数据流，并将解析出来的数据赋予 helloworld 类的相应数据成员。

该过程可以用下图表示：

![image-20200215182020440](https://tva1.sinaimg.cn/large/0082zybpgy1gbx99eug4uj30nk0g0gvd.jpg)

整个解析过程需要 Protobuf 本身的框架代码和由 Protobuf 编译器生成的代码共同完成。Protobuf 提供了基类 Message 以及 Message_lite 作为通用的 Framework，，CodedInputStream 类，WireFormatLite 类等提供了对二进制数据的 decode 功能，从 5.1 节的分析来看，Protobuf 的解码可以通过几个简单的数学运算完成，无需复杂的词法语法分析，因此 ReadTag() 等方法都非常快。 在这个调用路径上的其他类和方法都非常简单，感兴趣的读者可以自行阅读。 相对于 XML 的解析过程，以上的流程图实在是非常简单吧？这也就是 Protobuf 效率高的第二个原因了。





## 问题



**为什么Protocol Buffer能够动态根据内容大小调整码流？**

因为Protocol Buffer编码规则，如果内容是varint类型，那么判断值是小于128则用1字节来表示，其他类型也一样。（这里不懂，期望后面能找到更好的答案。。。）







## 参考

[Protocol Buffer原理](https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/)

[ProtoBuf实战](https://blog.csdn.net/zhglance/article/details/56017926)