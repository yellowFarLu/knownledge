# File类

既能代表1个类的名称，也能代表一个目录下一组文件的名称。

可以用File对象来创建新的目录 或者 尚不存在的目录。

```java
// 文件过滤示例
public class DirList {

    public static void main(String[] args) {
        // 获取某个目录的下的一组文件
        File path = new File(
                "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/hyreference");

        // 利用文件夹过滤器，过滤出符合正则的文件
        // 这里是过滤出java后缀的文件
        DirFilter dirFilter = new DirFilter(Pattern.compile(".*\\.java"));
        String[] list = path.list(dirFilter);

        System.out.println(Arrays.toString(list));
    }

}

class DirFilter implements FilenameFilter {

    private Pattern pattern;

    public DirFilter(Pattern pattern) {
        this.pattern = pattern;
    }

    // 每一个文件都会回调这个方法进行过滤
    @Override
    public boolean accept(File dir, String name) {
        return pattern.matcher(name).matches();
    }
}
```



# IO及NIO



## BIO及NIO的区别

- IO是阻塞的，NIO是非阻塞的
- IO面向流，NIO面向块（ByteBuffer）传输
- NIO使用多路复用实现





## 获取路径的方法

```java
getPath()  // 获取相对路径，getPath()返回的是File构造方法里的路径，是什么就是什么，不增不减
  
getAbsolutePath() // 获取绝对路径，返回的其实是user.dir+getPath()的内容
  
getCanonicalPath() // 返回的就是标准的将符号完全解析的路径  
```



```java
public static void main(String[] args) throws IOException {
    File file = new File(".././ProcessFiles.java");
    // 相对路径
    System.out.println("getPath=" + file.getPath());
    // 绝对路径
    System.out.println("getAbsolutePath=" + file.getAbsolutePath());
    // 解析../ 或者 ./ 符号之后的绝对路径
    System.out.println("getCanonicalPath=" + file.getCanonicalPath());
}

输出
getPath=.././ProcessFiles.java
getAbsolutePath=/Users/huangyuan/Desktop/Study/code/java-demo/.././ProcessFiles.java
getCanonicalPath=/Users/huangyuan/Desktop/Study/code/ProcessFiles.java
```



# 字节流



##InputStream

InputStream的作用是 表示那些从不同数据源产生输入的类。包括字节数组，String对象，文件，管道，其他数据源(如Internet链接)。每一种子类都有相应的InputStream子类，如下图:

![Snip20190421_1](https://ws3.sinaimg.cn/large/006tNc79gy1g2a44ntyh7j314i0u0dwy.jpg)



所有输入有关的类都继承自InputStream



### DataInputStream

使用readByte()读取的字节的任何值都是合法的，因此该返回值不能用来检测输入是否结束。可以利用available()方法查看还有多少可供读取的字符。

```java
available()  没有阻塞的情况下，可以读取的字节数
```

```java
public static void main(String[] args) throws Exception {
        DataInputStream in = new DataInputStream(
                new BufferedInputStream(
                        new FileInputStream(
                                "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/hyreference/MyDate.java")));
				
  	    // 当文件读取完，available()的值会自动减少
        while (in.available() != 0) {
            // readByte()会使得available()的值减少
            System.out.print((char) in.readByte());
        }
    }
```

readUTF()  <https://blog.csdn.net/xkwong/article/details/6450040>







##OutputStream

OutputStream决定了输出所要去的目标：字节数组、文件或管道。

![Snip20190421_2](https://ws2.sinaimg.cn/large/006tNc79gy1g2a47owf7aj31c40s4gye.jpg)

所有输出有关的类都继承自OutputStream





#字符流

Reader和Writer提供兼容Unicode与面向字符的IO功能。

设计Reader和Writer继承层次结构主要是为了国际化。老的IO流继承层次结构仅支持8位字节流，并不能很好的处理16位的Unicode字符。由于Unicode用于国际化，所以添加Reader和Writer继承层次结构就是为了在所有IO啊操作中都支持Unioncode。

![image-20190421121732647](https://ws3.sinaimg.cn/large/006tNc79gy1g2a4vgypnrj31ay0mcaio.jpg)



## API

```java
java.io.BufferedReader#readLine()   读取一行内容，会把\n换行符删掉
```







##RandomAccessFile

RandomAccessFile是Java输入/输出流体系中功能最丰富的文件内容访问类，既可以读取文件内容，也可以向文件输出数据。与普通的输入/输出流不同的是，RandomAccessFile支持跳到文件任意位置读写数据，RandomAccessFile对象包含一个记录指针，用以标识当前读写处的位置，当程序创建一个新的RandomAccessFile对象时，该对象的文件记录指针对于文件头（也就是0处），当读写n个字节后，文件记录指针将会向后移动n个字节。除此之外，RandomAccessFile可以自由移动该记录指针。

是一个完全独立的类，它和2大继承结构没有任何关系。



#### 源码解析

```java
long getFilePointer()：返回文件记录指针的当前位置

void seek(long pos)：将文件记录指针定位到pos位置
```



```java
RandomAccessFile类在创建对象时，除了指定文件本身，还需要指定一个mode参数，该参数指定RandomAccessFile的访问模式，该参数有如下四个值：

r：以只读方式打开指定文件。如果试图对该RandomAccessFile指定的文件执行写入方法则会抛出IOException
rw：以读取、写入方式打开指定文件。如果该文件不存在，则尝试创建文件
rws：以读取、写入方式打开指定文件。相对于rw模式，还要求对文件的内容或元数据的每个更新都同步写入到底层存储设备，默认情形下(rw模式下)，是使用buffer的，只有cache满的或者使用RandomAccessFile.close()关闭流的时候儿才真正的写到文件
rwd：与rws类似，只是仅对文件的内容同步更新到磁盘，而不修改文件的元数据
```

```java
public static void main(String[] args) {
    args = new String[1];
    args[0] = "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/hyreference/MyDate.java";

    RandomAccessFile raf = null;

    try {

      raf = new RandomAccessFile(args[0], "r");
      //            System.out.println("RandomAccessFile的文件指针初始位置:" + raf.getFilePointer());
      // 我理解是pos按字节为单位
      raf.seek(10);

      byte[] bbuf = new byte[1024];
      int hasRead = 0;

      while ((hasRead = raf.read(bbuf)) > 0) {
        System.out.print(new String(bbuf, 0, hasRead));
      }

    } catch (IOException e) {
      e.printStackTrace();
    } finally {
      try {
        if (raf != null) {
          raf.close();
        }
      } catch (IOException e) {
        e.printStackTrace();
      }
    }
}
```



## 缓冲

缓冲能显著的改善IO性能。比如BufferWriter。







#标准IO

为我们提供了标准IO，比如System.out、System.in、System.err 



##标准IO重定向

**输入本来默认是键盘，我们改成其他输入，就是输入重定向** ：例如从文本文件里输入。
**本来输出的位置是显示器，我们改成其他输出，就是输出重定向**：例如输出到文件。

System类提供了3个重定向输入输出方法

```java
static void setErr(PrintStream err)：重定向“标准“错误输出流。

static void setIn(InputStream in)：重定向”标准“输入流。

static void setOut(PrintStream out)：重定向”标准“输出流。
```



###重新向输入流

```java
/*
 * 重定向输入
 * 1.有一个已经初始化的InputStream输入流
 * 2.调用System.setIn()方法，将标淮输入流重定向到目的输入流
 * 3.读取System.in中的内容
 */
public class ReIn {

    public static void main(String[] args) throws UnsupportedEncodingException {


        try {
            //1.声明一个输入流
            FileInputStream fis = new FileInputStream(
                    "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/hyreference/test.txt");

            //2.重定向，把System.in重定向到fis输入流
            System.setIn(fis);

            //3.读取System.in标准输入流中的内容
            BufferedReader br =
                    new BufferedReader(new InputStreamReader(System.in, "UTF-8")); //设置字符编码

            //4.输出System.in中的内容
            String line;

            while((line = br.readLine()) != null){
                System.out.println(line);
            }

            //5.关闭流
            br.close();
            fis.close();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```



###重定向输出流

```java
public static void main(String[] args) {

    try {
      //1.声明一个输出流PrintStream对象
      PrintStream ps = new PrintStream(
        new FileOutputStream(
          "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/hyreference/test.txt",
          true));   //追加内容

      //2.重定向标淮输出流  把System.out重定向到ps
      System.setOut(ps);

      //3.使用PrintStream对象向流中写信息
      System.out.println(new ReOut());

      ps.close();

    } catch (FileNotFoundException e) {
      e.printStackTrace();
    }

}
```







# NIO

上述描述的都是IO，接下来描述的是NIO。NIO速度的提升来自于所使用的结构更加接近于操作系统执行IO的方法：通道和缓冲器。我们可以把它想象成煤矿，通道是一个包含煤炭(数据)的矿藏，而缓冲器则是矿藏的卡车。客车载满煤炭而归，我们再从卡车上获得煤炭。也就是说，我们没有直接和通道交互，我们只是和缓冲器交互，把缓冲器派送到通道，通道要么从缓冲器中获取数据，要么向缓冲器发送数据。

唯一直接与通道交互的缓冲器是ByteBuffer。



###IO多路复用

java里面的NIO使用了**IO多路复用模型**。

在IO多路复用模型中，会有一条线程（java中的selector）不断去轮询多个socket的状态（这些socket可能是读写网络、磁盘等），只有当socket真正有读写事件时，才真正调用IO读写操作。

因此，在IO多路复用模型中，只需要一条线程就可以管理多个socket，系统不需要建立新的线程，也不必维护这些线程，并且只有真正有socket读写事件的时候，才会使用IO资源，所以它大大减少了

参考: https://www.cnblogs.com/zwt1990/p/8821185.html



###Java NIO优点

- 在IO中，IO线程可能一直在等待网络，不能处理其他事；而在NIO中，只有在真正有读写请求的时候，才调用IO线程进行处理（只有selector线程一直在轮询）。
- Java NIO使用IO多路复用，避免每一个请求都有一条线程进行处理，减少了内存消耗，线程上下文切换。







## API

首先理解几个个概念。 mark标记、position位置、limit界限、capacity容量

![image-20190501132549101](https://ws1.sinaimg.cn/large/006tNc79gy1g2lr1mor6mj31b80ekah6.jpg)

### clear()

- 清空缓冲区，将position设置为0，limit设置为容量。我们可以调用此方法覆写缓冲区



### flip()

- 将limit设置为position，position设置为0。此方法用于准备从缓冲区读取已经写入的数据



###mark()

将mark设置为position



###rewind()

rewind()在读写模式下都可用，它单纯的将当前位置置0，同时取消mark标记，仅此而已；

也就是说写模式下limit仍保持与Buffer容量相同，只是重头写而已；

读模式下limit仍然与rewind()调用之前相同，也就是为flip()调用之前写模式下的position的最后位置，flip()调用后此position位置变为了读模式的limit位置，即越界位置，代码如下：

```java
public final Buffer rewind() { 
  position = 0; 
  mark = -1; 
  return this; 
}
```



### reset()

将position=mark；  也就是回到上一次标记的位置。



##视图缓冲器

视图缓冲器可以让我们通过某个特定的基本数据类型的视窗查看其底层的ByteBuffer。底层存储仍然使用的是ByteBuffer。对视图的修改都会映射成为ByteBuffer中数据的修改。以IntBuffer为例：

```java
// 在当前位置插入整数值，并且position的值递增1
java.nio.IntBuffer#put(int) 
  
// 获取position位置的值
java.nio.IntBuffer#get(int)
```



## 字节存放次序

不同的机器可能会使用不同的字节排序方法来存储数据。**高位优先**将最重要的的字节放在地址最低的存储单元。而**低位优先**将最重要的字节放到地址最高的存储单元。

ByteBuffer是以高位优先的形式存储数据的。我们可以使用ByteOrder.BIG_ENDIAN、ByteOrder.LITTLE_ENDIAN的order()方法改变ByteBuffer的排序方式。

![Snip20190425_1](/Users/huangyuan/Downloads/Snip20190425_1.png)



```java
public class ByteOrderDemo {

    public static void main(String[] args) {
        // ByteBuffer里面2个字节，存1个字符
        ByteBuffer bb = ByteBuffer.wrap(new byte[12]);

        bb.asCharBuffer().put("abcdef");
        System.out.println(Arrays.toString(bb.array()));

        bb.rewind();
        bb.order(ByteOrder.BIG_ENDIAN);
        bb.asCharBuffer().put("abcdef");
        System.out.println(Arrays.toString(bb.array()));

        bb.rewind();
        bb.order(ByteOrder.LITTLE_ENDIAN);
        bb.asCharBuffer().put("abcdef");
        System.out.println(Arrays.toString(bb.array()));
    }

}
```



## MappedByteBuffer

MappedByteBuffer是一种特殊的直接缓冲器，继承自ByteBuffer。

```java
public class MappedByteBufferDemo {

    // 128MB的大小
    static int length = 0x8FFFFFF;

    public static void main(String[] args) throws Exception {
        /**
         * 该程序创建的文件为128MB，这可能比操作系统一次允许载入的内存的空间大，
         * 但我们可以一次访问到整个文件，因为一次只有一部分文件放入了内存，
         * 文件的其他部分被交换出去，需要访问的时候再交换进来。
         * 用这种方式，很大的文件（最大2G）也可以很容易修改。
         * 这样子可以极大提高性能。
         */
        MappedByteBuffer mappedByteBuffer = new RandomAccessFile(
                "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/nio/text.txt",
                "rw")
                .getChannel()
                .map(FileChannel.MapMode.READ_WRITE, 0, length);

        for (int i = 0; i < length; i++) {
            mappedByteBuffer.put((byte)'x');
        }

        System.out.println("finished write");

        for (int i = length / 2; i < length/2 + 6; i++) {
            System.out.print((char)mappedByteBuffer.get(i));
        }
    }

}

```



## 压缩

Java中IO库支持读写压缩格式的数据流，按字节方式进行处理。

![image-20190501141518436](https://ws4.sinaimg.cn/large/006tNc79gy1g2lsh33mhbj31am0h6qeb.jpg)







## 序列化的控制

通过实现Externalizable接口，可以控制序列化。Externalizable继承了Serializable接口。同时新增了writeExternal、readExternal方法。writeExternal会在序列化的时候自动调用，readExternal会在反序列化的时候自动调用。

```java
public class Blip1 implements Externalizable {

    public Blip1() {
        System.out.println("Blip1 Constructor");
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        System.out.println("Blip1 writeExternal");
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        System.out.println("Blip1 readExternal");
    }
}

class Blip2 implements Externalizable {

    public Blip2() {
        System.out.println("Blip2 Constructor");
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        System.out.println("Blip2 writeExternal");
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        System.out.println("Blip2 readExternal");
    }
}


public class Blips {

    public static void main(String[] args) throws Exception {
        final String path = "Blips.out";

        Blip1 b1 = new Blip1();
        Blip2 b2 = new Blip2();

        ObjectOutputStream o = new ObjectOutputStream(
                new FileOutputStream(path)
        );

        o.writeObject(b1);
        o.writeObject(b2);
        o.close();

        ObjectInputStream in = new ObjectInputStream(
                new FileInputStream(path)
        );
        // 反序列的类，一定要收public的构造函数
        b1 = (Blip1)in.readObject();
        b2 = (Blip2)in.readObject();
    }

}
```

*使用Externalizable接口，那么反序列的类，一定要收public的构造函数*

对于Serializable对象，对象完全以它存储的二进制来构建，而不调用构造器。而对于一个Externalizable对象，默认构造器都会被调用，然后调用readExternal()。

```java
public class Blips3 implements Externalizable {

    private int i1;

    private int i2;

    private int i3;

    public Blips3() {
        System.out.println("Blips3 Constructor");
    }

    public Blips3(int i1, int i2, int i3) {
        this.i1 = i1;
        this.i2 = i2;
        this.i3 = i3;
    }

    @Override
    public String toString() {
        return "Blips3{" +
                "i1=" + i1 +
                ", i2=" + i2 +
                ", i3=" + i3 +
                '}';
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(i1);
        out.writeInt(i2);
        out.writeInt(i3);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        // 数据的读取顺序和写入顺序一致
        i1 = in.readInt();
        i2 = in.readInt();
        i3 = in.readInt();
    }

    public static void main(String[] args) throws Exception {
        Blips3 b3 = new Blips3(1, 2, 3);

        final String path = "Blips.out";

        ObjectOutputStream out = new ObjectOutputStream(
                new FileOutputStream(path)
        );
        out.writeObject(b3);
        out.close();

        ObjectInputStream in = new ObjectInputStream(
                new FileInputStream(path)
        );
        b3 = (Blips3) in.readObject();
        System.out.println(b3);
    }
}

```



###transient

transient可以告诉系统，某个字段不用仅需序列化、反序列化。



###序列化 深拷贝

可以用过一个字节数组来使用对象序列化，从而实现对任何可Serializable对象的深拷贝。深拷贝意味着我们复制的是整个对象网，而不仅仅是基本对象及其引用。

```java
class House implements Serializable {

}

class Animal implements Serializable {
    private String name;

    private House house;

    public Animal(String name, House house) {
        this.name = name;
        this.house = house;
    }

    @Override
    public String toString() {
        return "Animal{" +
                "name='" + name + '\'' +
                ", house=" + house +
                '}';
    }
}

public class MyWord {

    public static void main(String[] args) throws Exception {
        House house = new House();

        List<Animal> animals = new ArrayList<>();
        animals.add(new Animal("one", house));
        animals.add(new Animal("two", house));
        animals.add(new Animal("three", house));

        ByteArrayOutputStream buf1 = new ByteArrayOutputStream();

        ObjectOutputStream out = new ObjectOutputStream(buf1);
        out.writeObject(animals);
        out.writeObject(animals);

        ByteArrayOutputStream buf2 = new ByteArrayOutputStream();
        ObjectOutputStream out2 = new ObjectOutputStream(buf2);
        out2.writeObject(animals);


        ObjectInputStream in1 = new ObjectInputStream(
                new ByteArrayInputStream(buf1.toByteArray())
        );

        ObjectInputStream in2 = new ObjectInputStream(
                new ByteArrayInputStream(buf2.toByteArray())
        );

        List animals1 = (List)in1.readObject();
        List animals2 = (List)in1.readObject();

        // 系统无法知道in2内的对象是in1的对象的别名（无法知道2个对象是一样的），因此in2会产生出完全不同的对象网
        List animals3 = (List)in2.readObject();

        System.out.println(animals1);
        System.out.println(animals2);
        System.out.println(animals3);
    }

}
```



static的字段，需要手动进行序列化

```java
abstract class Shape implements Serializable {

    public static final int RED = 1, BLUE = 2, GREEN = 3;

    private int xPos, yPos, dimension;

    private static Random rand = new Random(47);

    private static int counter = 0;

    public abstract void setColor(int newColor);

    public abstract int getColor();

    public Shape(int xPos, int yPos, int dimension) {
        this.xPos = xPos;
        this.yPos = yPos;
        this.dimension = dimension;
    }

    @Override
    public String toString() {
        return "Shape{" +
                "xPos=" + xPos +
                ", yPos=" + yPos +
                ", dimension=" + dimension +
                '}';
    }

    public static Shape randomFactory() {
        int xVal = rand.nextInt(100);
        int yVal = rand.nextInt(100);
        int dim = rand.nextInt(100);

        switch (counter++ %3) {
            default:
            case 0: return new Circle(xVal, yVal, dim);
            case 1: return new Squre(xVal, yVal, dim);
            case 2: return new Line(xVal, yVal, dim);
        }
    }
}

class Circle extends Shape {
    private static int color;

    public Circle(int xPos, int yPos, int dimension) {
        super(xPos, yPos, dimension);
        color = RED;
    }

    public int getColor() {
        return color;
    }

    public void setColor(int color) {
        Circle.color = color;
    }

    @Override
    public String toString() {
        return "Circle{color=" + color + "}" + super.toString();
    }
}


class Squre extends Shape {
    private static int color;

    public Squre(int xPos, int yPos, int dimension) {
        super(xPos, yPos, dimension);
        color = RED;
    }

    public int getColor() {
        return color;
    }

    public void setColor(int color) {
        Squre.color = color;
    }

    @Override
    public String toString() {
        return "Squre{color=" + color + "}" + super.toString();
    }
}

class Line extends Shape {

    private static int color;

    public static void serializeStaticState(ObjectOutputStream outputStream)
    throws IOException {
        outputStream.writeInt(color);
    }

    public static void deserializeStaticState(ObjectInputStream inputStream)
    throws IOException {
        color = inputStream.readInt();
    }

    public Line(int xPos, int yPos, int dimension) {
        super(xPos, yPos, dimension);
    }
    public int getColor() {
        return color;
    }

    public void setColor(int color) {
        Line.color = color;
    }

    @Override
    public String toString() {
        return "Line{color=" + color + "}" + super.toString();
    }
}

public class StoreCADState {

    public static void main(String[] args) throws Exception {
        List<Class<? extends Shape>> shapeTypes = new ArrayList<>();
        shapeTypes.add(Circle.class);
        shapeTypes.add(Squre.class);
        shapeTypes.add(Line.class);

        List<Shape> shapes = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            shapes.add(Shape.randomFactory());
        }

        for (int i = 0; i < 10; i++) {
            shapes.get(i).setColor(Shape.GREEN);
        }

        ObjectOutputStream outputStream = new ObjectOutputStream(
                new FileOutputStream(
                        "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/serli/test.txt")
        );

        outputStream.writeObject(shapeTypes);
        Line.serializeStaticState(outputStream);

        outputStream.writeObject(shapes);
    }

}

```



```java
public class RecoverCADState {

    public static void main(String[] args) throws Exception {
        ObjectInputStream inputStream = new ObjectInputStream(
                new FileInputStream(
                        "/Users/huangyuan/Desktop/Study/code/java-demo/src/main/java/huangy/serli/test.txt")
        );

        List<Class<? extends Shape>> shapeTypes = (List<Class<? extends Shape>>)inputStream.readObject();
        Line.deserializeStaticState(inputStream);

        List<Shape> shapes = (List<Shape>)inputStream.readObject();

        System.out.println(shapeTypes);
        System.out.println("-----------------");

        System.out.println(shapes);
    }

}
```





###Preferences

Preferences是对象序列化的一种，用于存储和读取用户偏好以及程序配置项的设置，可以自动存储和读取信息。不过它只能用于基本类型和字符串，并且每个字符串存储长度不能超过8k。

Preferences是一个键-值集合，类似于映射，存储在一个节点层次结构中。





