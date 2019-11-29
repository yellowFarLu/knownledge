# String



## 概念

不可变的，每一次修改实际上生成新的字符串，并且该字符串的值是修改后的值。new String都是在堆上创建字符串对象

String 被声明为 final，因此它不可被继承。(Integer 等包装类也不能被继承）

在 Java 8 中，String 内部使用 char 数组存储数据。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
}
```

**String如何保证不可变？**

- value 数组被声明为 final，这意味着 value 数组初始化之后就不能再引用其它数组
- 并且String 内部没有改变 value 数组的方法，因此可以保证 String 不可变。



### 类如何实现不可变

- 所有域是final的，并且是private，并且提供修改对象状态的方法
- 把类声明为final，保证类不可以被扩展，不可以被子类修改方法的行为从而改变域的值



### 不可变的好处

**1、线程安全**

String 不可变性天生具备线程安全，可以在多个线程中安全地使用。



**2、类的实现简单**

由于对象是不可变的，天然支持线程安全，因此不用使用额外的代码来保证线程安全





## 几种字符串类

**String、StringBuilder、StringBuffer的区别**

- 可变性
  - String不可变
  - StringBuilder、StringBuffer可变
- 线程安全
  - String不可变，因此是线程安全的
  - StringBuilder不是线程安全的
  - StringBuffer 是线程安全的，内部使用 synchronized 进行同步



**StringBuilder、StringBuffer的区别**

- StringBuffer 与 StringBuilder 中的方法和功能完全是等价的，
- StringBuffer 中的方法都采用了 synchronized 关键字进行修饰，因此是线程安全的，

而 StringBuilder 没有这个修饰，可以被认为是线程不安全的。 

- 在单线程程序下，StringBuilder效率更快，因为它不需要加锁，不具备多线程安全

而StringBuffer则每次都需要判断锁，效率相对更低





## 字符串常量池

字符串常量池（String Pool）保存着所有字符串字面量（literal strings），这些字面量在编译时期就确定。不仅如此，还可以使用 String 的 intern() 方法在运行过程中将字符串添加到 字符串常量池 中。

intern() 方法的实现如下：

- JDK1.6的实现
  - 当调用 intern() 方法时，编译器会先判断常量池中这个字符串是否存在，如果存在，直接返回该常量的引用；如果不存在，将该字符串添加到常量池中，并返回指向该常量的引用。
- JDK 1.7的实现
  - intern()方法先去查询常量池中是否有已经存在，如果存在，则返回常量池中的引用，这一点与之前没有区别，区别在于，如果在常量池找不到对应的字符串，则不会再将字符串拷贝到常量池，而只是在**常量池中生成一个对原字符串的引用**。简单的说，就是往常量池放的东西变了：原来在常量池中找不到时，复制一个副本放到常量池，1.7后则是将在堆上的地址引用复制到常量池。

下面示例中，s1 和 s2 采用 new String() 的方式新建了两个不同字符串，而 s3 和 s4 是通过 s1.intern() 方法取得一个字符串引用。intern() 首先把 s1 引用的字符串放到 String Pool 中，然后返回这个字符串引用。因此 s3 和 s4 引用的是同一个字符串。

```java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
String s4 = s1.intern();
System.out.println(s3 == s4);           // true
```

如果是采用 "bbb" 这种字面量的形式创建字符串，会自动地将字符串放入 String Pool 中。

```java
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);  // true
```















## API



### intern()

含义：

- JDK1.6的实现：当调用 intern() 方法时，编译器会先判断常量池中这个字符串是否存在，如果存在，直接返回该常量的引用；如果不存在，将该字符串添加到常量池中，并返回指向该常量的引用。

![20170412203247146](https://ws1.sinaimg.cn/large/006tKfTcgy1g1d29slva5j30b40730t8.jpg)



![20170412203303115](https://ws2.sinaimg.cn/large/006tKfTcgy1g1d2apj49dj30b008fjs3.jpg)

- 通过字面量赋值创建字符串（如：String str=”twm”）时，会先在常量池中查找是否存在相同的字符串，若存在，则将栈中的引用直接指向该字符串；若不存在，则在常量池中生成一个字符串，再将栈中的引用指向该字符串。

  ![20170412203323021](https://ws1.sinaimg.cn/large/006tKfTcgy1g1d2bzecl6j30b7096mxs.jpg)

- 常量字符串的“+”操作，编译阶段直接会合成为一个字符串。如string str=”JA”+”VA”，在编译阶段会直接合并成语句String str=”JAVA”，于是会去常量池中查找是否存在”JAVA”,从而进行创建或引用。

- 对于final字段，编译期直接进行了常量替换（而对于非final字段则是在运行期进行赋值处理的）。 
  final String str1=”ja”; 
  final String str2=”va”; 
  String str3=str1+str2; 

  在编译时，直接替换成了String str3=”ja”+”va”，根据第三条规则，再次替换成String str3=”JAVA”

- 常量字符串和变量拼接时（如：String str3=baseStr + “01”;）会调用stringBuilder.append()在堆上创建新的对象。

- JDK 1.7后，intern方法还是会先去查询常量池中是否有已经存在，如果存在，则返回常量池中的引用，这一点与之前没有区别，区别在于，如果在常量池找不到对应的字符串，则不会再将字符串拷贝到常量池，而只是在**常量池中生成一个对原字符串的引用**。简单的说，就是往常量池放的东西变了：原来在常量池中找不到时，复制一个副本放到常量池，1.7后则是将在堆上的地址引用复制到常量池。

  ![20170412203343662](https://ws2.sinaimg.cn/large/006tKfTcgy1g1d2elz915j30bg08zdgo.jpg)

```java
举例说明：
String str2 = new String("str")+new String("01");
str2.intern();
String str1 = "str01";
System.out.println(str2==str1);

在JDK 1.7下，当执行str2.intern();时，因为常量池中没有“str01”这个字符串，所以会在常量池中生成一个对堆中的“str01”的引用(注意这里是引用 ，就是这个区别于JDK 1.6的地方。在JDK1.6下是生成原字符串的拷贝)，而在进行String str1 = “str01”;字面量赋值的时候，常量池中已经存在一个引用，所以直接返回了该引用，因此str1和str2都指向堆中的同一个字符串，返回true。



String str2 = new String("str")+new String("01");
String str1 = "str01";
str2.intern();
System.out.println(str2==str1);

将中间两行调换位置以后，因为在进行字面量赋值（String str1 = “str01″）的时候，常量池中不存在，所以str1指向的常量池中的位置，而str2指向的是堆中的对象，再进行intern方法时，对str1和str2已经没有影响了，所以返回false。
```

```java
public class HYString {

    public static void main(String[] args) {

        /*
         * 原本没有计算机软件的字符串
         */
        String str1 = new StringBuilder("计算机").append("软件").toString();
        System.out.println(str1.intern() == str1);

        /*
         * 系统之前就会加载Java字符串
         */
        String str2 = new StringBuilder("ja").append("va").toString();
        System.out.println(str2.intern() == str2);
    }

}

输出
  true
  fasle
```

参考 https://blog.csdn.net/soonfly/article/details/70147205









### format()

```java
public class HYFormat {

    private double total = 0;

    private Formatter f = new Formatter(System.out);

    public void printfTitle() {
        /*
         * 格式  %[正负号][宽度][.precision]
         * 1、负号表示左对齐，正号表示右对齐
         * 2、宽度控制一个域的最小尺寸，不够则会添加空格
         * 3、precision用于指明最大尺寸，应用于不同数据转换时，precision的意义也不同。
         *  （1）precision应用于String时，它表示输出String字符的最大数量
         *  （2）precision应用于浮点数时，它表示小数部分要显示出来的位数（默认是6位）。小数过多则舍入，太少则补0。
         *
         */
        f.format("%-15s %5s %10s\n", "Item", "Qty", "Price");
        f.format("%-15s %5s %10s\n", "----", "---", "-----");
    }

    public void print(String name, int qty, double price) {
        f.format("%-15.15s %5d %10.2f\n", name, qty, price);
        total += price;
    }

    public void printTotal() {
        f.format("%-15s %5s %10.8f\n", "Tax", "", total*0.06);
        f.format("%-15s %5s %10s\n", "", "", "-------");
        f.format("%-15s %5s %10.2f\n", "Total", "", total*1.06);
    }

    public static void main(String[] args) {
        HYFormat hyFormat =  new HYFormat();
        hyFormat.printfTitle();

        hyFormat.print("Jack's Magic Beans", 4, 4.25);

        hyFormat.print("Priness Peas", 3, 5.1);

        hyFormat.print("Three beas", 1, 14.29);

        hyFormat.printTotal();
    }

}
```













