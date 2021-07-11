# IO



## 整体概念



Java 的 I/O 大概可以分成以下几类：

- 磁盘操作：File
- 字节操作：InputStream 和 OutputStream
- 字符操作：Reader 和 Writer
- 对象操作：Serializable
- 网络操作：Socket
- 新的输入/输出：NIO



Java IO 体系看起来类很多，感觉很复杂，但其实是 IO 涉及的因素太多了。在设计 IO 相关的类时，编写者也不是从同一个方面考虑的，所以会给人一种很乱的感觉，并且还有设计模式的使用，更加难以使用这些 IO 类，所以特地对 Java 的 IO 做一个总结。

IO 类设计出来，肯定是为了解决 IO 相关的操作的，想一想哪里会有 IO 操作？网络、磁盘。网络操作相关的类是在 java.net 包下，不在本文的总结范围内。提到磁盘，你可能会想到文件，文件操作在 IO 中是比较典型的操作。在 Java 中引入了 “流” 的概念，它表示任何有能力产生数据源或有能力接收数据源的对象。数据源可以想象成水源，海水、河水、湖水、一杯水等等。数据传输可以想象为水的运输，古代有用桶运水，用竹管运水的，现在有钢管运水，不同的运输方式对应不同的运输特性。

**从数据传输方式看，可以将 IO 类分为：**

- 1、字节流
- 2、字符流

字节流是以一个字节单位来运输的，比如一杯一杯的取水。而字符流是以多个字节来运输的，比如一桶一桶的取水，一桶水又可以分为几杯水。



**字节流和字符流的区别：**

- 字节流读取单个字节，字符流读取单个字符
- 字节流用来处理二进制文件（图片、MP3、视频文件），字符流用来处理文本文件（可以看做是特殊的二进制文件，使用了某种编码，人可以阅读）
- 简而言之，字节是个计算机看的，字符才是给人看的。







## IO类

IO 类虽然很多，但最基本的是 4 个抽象类：InputStream、OutputStream、Reader、Writer。

最基本的方法也就是一个读 read() 方法、一个写 write() 方法。方法具体的实现还是要看继承这 4 个抽象类的子类，毕竟我们平时使用的也是子类对象。这些类中的一些方法都是（Native）本地方法、所以并没有 Java 源代码，这里给出笔者觉得不错的 Java IO 源码分析 [传送门](https://blog.csdn.net/panweiwei1994/article/details/78046000)，按照上面这个思路看，先看子类基本方法，然后在看看子类中还新增了那些方法，相信你也可以看懂的，我这里就只对上后面说的常用的类进行总结。

先来看 InputStream 和 OutStream 中的方法简介，因为都是抽象类、大都是抽象方法、所以就不贴源码喽！**注意这里的读取和写入，其实就是获取（输入）数据和输出数据。**



## 磁盘操作

File 类可以用于表示文件和目录的信息，但是它不表示文件的内容。

递归地列出一个目录下所有文件：

```java
public static void listAllFiles(File dir) {
    if (dir == null || !dir.exists()) {
        return;
    }
    if (dir.isFile()) {
        System.out.println(dir.getName());
        return;
    }
    for (File file : dir.listFiles()) {
        listAllFiles(file);
    }
}
```

从 Java7 开始，可以使用 Paths 和 Files 代替 File。





## 字节操作

通过IO操作字节。

比如说实现文件复制

```java
public static void copyFile(String src, String dist) throws IOException {
    FileInputStream in = new FileInputStream(src);
    FileOutputStream out = new FileOutputStream(dist);

    byte[] buffer = new byte[20 * 1024];
    int cnt;

    // read() 最多读取 buffer.length 个字节
    // 返回的是实际读取的个数
    // 返回 -1 的时候表示读到 eof，即文件尾
    while ((cnt = in.read(buffer, 0, buffer.length)) != -1) {
        out.write(buffer, 0, cnt);
    }

    in.close();
    out.close();
}
```





## 字符操作

### 编码与解码

编码就是把字符转换为字节，而解码是把字节重新组合成字符。

如果编码和解码过程使用不同的编码方式那么就出现了乱码。

- GBK 编码中，中文字符占 2 个字节，英文字符占 1 个字节；
- UTF-8 编码中，中文字符占 3 个字节，英文字符占 1 个字节；
- UTF-16be 编码中，中文字符和英文字符都占 2 个字节。

UTF-16be 中的 be 指的是 Big Endian，也就是大端。相应地也有 UTF-16le，le 指的是 Little Endian，也就是小端。

Java 的内存编码使用双字节编码 UTF-16be，这不是指 Java 只支持这一种编码方式，而是说 char 这种类型使用 UTF-16be 进行编码。char 类型占 16 位，也就是两个字节，Java 使用这种双字节编码是为了让一个中文或者一个英文都能使用一个 char 来存储。



### String 的编码方式

String 可以看成一个字符序列，可以指定一个编码方式将它编码为字节序列，也可以指定一个编码方式将一个字节序列解码为 String。

```java
String str1 = "中文";
byte[] bytes = str1.getBytes("UTF-8");
String str2 = new String(bytes, "UTF-8");
System.out.println(str2);
```

在调用无参数 getBytes() 方法时，默认的编码方式不是 UTF-16be。双字节编码的好处是可以使用一个 char 存储中文和英文，而将 String 转为 bytes[] 字节数组就不再需要这个好处，因此也就不再需要双字节编码。getBytes() 的默认编码方式与平台有关，一般为 UTF-8。



### Reader 与 Writer

不管是磁盘还是网络传输，最小的存储单元都是字节，而不是字符。但是在程序中操作的通常是字符形式的数据，因此需要提供对字符进行操作的方法。

- InputStreamReader 实现从字节流解码成字符流；
- OutputStreamWriter 实现字符流编码成为字节流。

```java
// 实现逐行输出文本文件的内容
public static void readFileContent(String filePath) throws IOException {

    FileReader fileReader = new FileReader(filePath);
    BufferedReader bufferedReader = new BufferedReader(fileReader);

    String line;
    while ((line = bufferedReader.readLine()) != null) {
        System.out.println(line);
    }

    // 装饰者模式使得 BufferedReader 组合了一个 Reader 对象
    // 在调用 BufferedReader 的 close() 方法时会去调用 Reader 的 close() 方法
    // 因此只要一个 close() 调用即可
    bufferedReader.close();
}
```





## 对象操作

### 序列化

序列化就是将一个对象转换成字节序列，方便存储和传输。

- 序列化：ObjectOutputStream.writeObject()
- 反序列化：ObjectInputStream.readObject()

不会对静态变量进行序列化，因为序列化只是保存对象的状态，静态变量属于类的状态。



### Serializable

序列化的类需要实现 Serializable 接口，它只是一个标准，没有任何方法需要实现，但是如果不去实现它的话而进行序列化，会抛出异常。



### transient

transient 关键字可以使一些属性不会被序列化。

ArrayList 中存储数据的数组 elementData 是用 transient 修饰的，因为这个数组是动态扩展的，并不是所有的空间都被使用，因此就不需要所有的内容都被序列化。通过重写序列化和反序列化方法，使得可以只序列化数组中有内容的那部分数据。





## 网络操作

Java 中的网络支持：

- InetAddress：用于表示网络上的硬件资源，即 IP 地址；
- URL：统一资源定位符；
- Sockets：使用 TCP 协议实现网络通信；
- Datagram：使用 UDP 协议实现网络通信。



### InetAddress

没有公有的构造函数，只能通过静态方法来创建实例。

```java
InetAddress.getByName(String host);
InetAddress.getByAddress(byte[] address);
```



### URL

可以直接从 URL 中读取字节流数据。

```java
public static void main(String[] args) throws IOException {

    URL url = new URL("http://www.baidu.com");

    /* 字节流 */
    InputStream is = url.openStream();

    /* 字符流 */
    InputStreamReader isr = new InputStreamReader(is, "utf-8");

    /* 提供缓存功能 */
    BufferedReader br = new BufferedReader(isr);

    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }

    br.close();
}
```



### Sockets

- ServerSocket：服务器端类
- Socket：客户端类
- 服务器和客户端通过 InputStream 和 OutputStream 进行输入输出。

![image-20191130120801738](https://tva1.sinaimg.cn/large/006tNbRwgy1g9fxsappq8j30u80i0q5r.jpg)



### Datagram

- DatagramSocket：通信类
- DatagramPacket：数据包类







## 设计模式

Java I/O 使用了装饰者模式来实现。以 InputStream 为例，

- InputStream 是抽象组件；
- FileInputStream 是 InputStream 的子类，属于具体组件，提供了字节流的输入操作；
- FilterInputStream 属于抽象装饰者，装饰者用于装饰组件，为组件提供额外的功能。例如 BufferedInputStream 为 FileInputStream 提供缓存的功能。

实例化一个具有缓存功能的字节流对象时，只需要在 FileInputStream 对象上再套一层 BufferedInputStream 对象即可。





### 装饰器公共接口

FilterInputStream和FilterOutputStream是用来提供装饰器类接口以控制特定输入流（InputStream）和输出流（OutputStream）的两个类。所有装饰器都必须实现FilterInputStream或者FilterOutputStream。



#### FilterInputStream

![image-20190421120348826](https://ws2.sinaimg.cn/large/006tNc79gy1g2a4h6aeu9j31ca0s2qgq.jpg)

比如我们常用的BufferedInputStream就实现了FilterInputStream接口。



#### FilterOutputStream

![image-20190421120553005](https://ws3.sinaimg.cn/large/006tNc79gy1g2a4jc22hdj31aw0nak3h.jpg)



## 读取jar包中的文件

```java
JarFile jarFile = new JarFile(file);

// XXX 是相对路径，相对jar包下面resource路径，一般是前面没有/
JarEntry jarEntry = jarFile.getJarEntry("XXXX");

if (jarEntry == null) {
  log.info("jarFile getJarEntry is null, path={}", file);
  return new ArrayList<>();
}

// 根据实体创建输入流
inputStream = jarFile.getInputStream(jarEntry);
```

参考：https://www.javatt.com/p/81226

​            https://blog.csdn.net/u013467442/article/details/88807557

​            https://www.iteye.com/blog/hxraid-483115





## 参考

[IO体系](https://www.jianshu.com/p/715659e4775f)

