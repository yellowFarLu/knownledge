# Java基础



## 数据类型



### 基本类型

![image-20190216171918836](https://ws3.sinaimg.cn/large/006tKfTcgy1g08dxrqn2rj30m20botbh.jpg)

- java的基础类型有哪些？
  - java总共8个基础数据类型，如下：
    - byte、char、boolean、shot、int、long、float、double
- **基本类型直接存储值**，并且存储于栈中，因此更加高效。
- **java中基本类型的存储空间大小是固定的**，不会随机器硬件架构的变化而变化。这是java更加具有移植性的原因之一。

- char占用2个字节、float占用4字节
- boolean类型占几个字节？
  - 《Java虚拟机规范》中是4字节，具体占用字节数看虚拟机实现。Java语言中的boolean值，在编译之后，都使用int类型来代替。
  - 为什么使用int，因为对于32位处理器来说，一次处理数据是32位，具有高效存取的特点
  - 参考：[boolean占用字节](https://www.cnblogs.com/wangtianze/p/6690665.html)





### 包装类型

- 基本类型都有对应的包装类型

- 基本类型与其对应的包装类型之间的赋值使用自动装箱与拆箱完成。
- 自动包装不能应用于数组。比如，int[]不能自动包装成Integer[]。

```java
Integer x = 2;     // 装箱 调用了 Integer.valueOf(2)
int y = x;         // 拆箱 调用了 x.intValue()
```







### 缓存池

包装类型为了提高性能，在类加载的时候预先初始化一些值，作为缓存。



**new Integer(123) 与 Integer.valueOf(123) 的区别在于：**

- new Integer(123)每次都会新建一个对象
- Integer.valueOf(123) 会使用缓存池中的对象，多次调用会取得同一个对象的引用。

```java
Integer x = new Integer(123);
Integer y = new Integer(123);
System.out.println(x == y);    // false

Integer z = Integer.valueOf(123);
Integer k = Integer.valueOf(123);
System.out.println(z == k);   // true
```

valueOf() 方法的实现比较简单，就是先判断值是否在缓存池中，如果在的话就直接返回缓存池的内容。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        // -128~127的顺序存放，因此要加上一段偏移量
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

**在 Java 8 中，Integer 缓存池的大小默认为 -128~127。**

编译器会在自动装箱的过程中调用 valueOf() 方法，因此多个值相同且值在缓存池范围内的 Integer 实例使用自动装箱来创建，那么就会引用相同的对象。

```java
Integer m = 123;
Integer n = 123;
System.out.println(m == n); // true
```

在使用基本类型对应的包装类型时，如果该数值范围在缓冲池范围内，就可以直接使用缓冲池中的对象。

- Boolean： 缓存了true或者false
- Character：缓存了0~128的整数对应的字符
- Byte：缓存了-128~127
- Shot：缓存了-128~127
- Integer：缓存了-128~127

在 jdk 1.8 所有的数值类缓冲池中，Integer 的缓冲池 IntegerCache 很特殊，这个缓冲池的下界是 - 128，上界默认是 127，**但是这个上界是可调的**，在启动 jvm 的时候，通过 -XX:AutoBoxCacheMax=<size> 来指定这个缓存池的大小，该选项在 JVM 初始化的时候会设定一个名为 java.lang.IntegerCache.high 系统属性，然后 IntegerCache 初始化的时候就会读取该系统属性来决定上界。





## 运算



### 参数传递

Java 的参数是以值传递的形式传入方法中，而不是引用传递。



### float 与 double

Java 不能隐式执行向下转型，因为这会使得精度降低。

1.1 字面量属于 double 类型，不能直接将 1.1 直接赋值给 float 变量，因为这是向下转型。

```java
float f = 1.1;  // 编译报错
```

1.1f 字面量才是 float 类型。

```java
float f = 1.1f;
```



### 隐式类型转换

因为字面量 1 是 int 类型，它比 short 类型精度要高，因此不能隐式地将 int 类型下转型为 short 类型。

```java
short s1 = 1;  // 这句话没报错，很奇怪，高精度赋值给低精度，向下转型，居然没有报错。
s1 = s1 + 1;  // 编译报错，因为s1 + 1的结果为int
```

**为什么 short s = 1 不报错?**

因为int没有超过shot的范围，则正常截取，也不会影响结果。

而double类型是不可预测的,可能很简单的数字都占满了所用的字节,比如:0.5,在内存中其实表示为:0.499999999999         这样的数字截取低位部分就是另一个数字了，因此不能截取。



但是使用 += 或者 ++ 运算符可以执行隐式类型转换。

```java
s1 += 1;
s1++;
```

上面的语句相当于将 s1 + 1 的计算结果进行了向下转型：

```java
s1 = (short) (s1 + 1);
```



### switch

从 Java 7 开始，可以在 switch 条件判断语句中使用 String 对象。

```java
String s = "a";
switch (s) {
    case "a":
        System.out.println("aaa");
        break;
    case "b":
        System.out.println("bbb");
        break;
}
```

switch 不支持 long，是因为 switch 的设计初衷是对那些只有少数的几个值进行等值判断，如果值过于复杂，那么还是用 if 比较合适。









## 继承

### 访问权限

可以对类或类中的成员（字段以及方法）加上访问修饰符。

- 类可见表示其它类可以用这个类创建实例对象。
- 成员可见表示其它类可以用这个类的实例对象访问到该成员；



下面是几种访问修饰符的概述：

**public**

- 所有类都可以访问

**protected**

- 同一个包或者子类可以访问 基类的protected成员。

**default**

- 同一个包内，才可以访问其他类的public、protected、default成员。
- 一些类处于相同目录，并且没有声明package包时，java将这样的文件归属到“默认包”中。因此可以相互访问。

**private**

- 只有本类可以访问。类不可以声明为private或者protected（内部类除外）。



设计良好的模块会隐藏所有的实现细节，把它的 API 与它的实现清晰地隔离开来。模块之间只通过它们的 API 进行通信，一个模块不需要知道其他模块的内部工作情况，这个概念被称为信息隐藏或封装。因此访问权限应当尽可能地使每个类或者成员不被外界访问。

如果子类的方法重写了父类的方法，那么**子类中该方法的访问级别不允许低于父类的访问级别**。这是为了确保可以使用父类实例的地方都可以使用子类实例，也就是确保满足里氏替换原则。

字段建议不是公有的，因为这么做的话就失去了对这个字段修改行为的控制，客户端可以对其随意修改。



### **抽象类**

抽象类和抽象方法都使用 abstract 关键字进行声明。如果一个类中包含抽象方法，那么这个类必须声明为抽象类。

抽象类和普通类最大的区别是，**抽象类不能被实例化**，需要继承抽象类才能实例化其子类。



### 接口

接口确定了对某一特定对象所能发出的请求

- 利用接口实现多继承
- 利用继承为接口添加新的方法
- 接口中的域都是static和final的，这些域的值被存储在接口的静态存储区域内

**从 Java 8 开始，接口也可以拥有默认的方法实现**，这是因为不支持默认方法的接口的维护成本太高了。在 Java 8 之前，如果一个接口想要添加新的方法，那么要修改所有实现了该接口的类。

接口的成员（字段 + 方法）默认都是 public 的，并且不允许定义为 private 或者 protected。

接口的字段默认都是 static 和 final 的。

```java
public interface InterfaceExample {

    void func1();

    default void func2(){
        System.out.println("func2");
    }

    int x = 123;
    // int y;               // Variable 'y' might not have been initialized
    public int z = 0;       // Modifier 'public' is redundant for interface fields
    // private int k = 0;   // Modifier 'private' not allowed here
    // protected int l = 0; // Modifier 'protected' not allowed here
    // private void fun3(); // Modifier 'private' not allowed here
}
```

 **抽象类 和 接口 比较**

- 从设计层面上看，抽象类提供了一种 IS-A 关系，那么就必须满足里式替换原则，即**子类对象必须能够替换掉所有父类对象**。而接口更像是一种 LIKE-A 关系，它只是提供一种方法实现契约，并不要求接口和实现接口的类具有 IS-A 关系。
- 从使用上来看，**一个类可以实现多个接口，但是不能继承多个抽象类。**
- **接口的字段只能是 static 和 final 类型的**，而抽象类的字段没有这种限制。
- 接口的成员只能是 public 的，而**抽象类的成员可以有多种访问权限**。



**使用选择**

使用接口：

- 需要让不相关的类都实现一个方法，例如不相关的类都可以实现 Compareable 接口中的 compareTo() 方法
- 需要使用多重继承
- 面向接口编程

使用抽象类：

- 需要在几个相关的类中共享代码
- 需要能控制继承来的成员的访问权限，而不是都为 public
- 需要继承非静态和非常量字段

在很多情况下，接口优先于抽象类。因为接口没有抽象类严格的类层次结构要求，可以灵活地为一个类添加行为。并且从 Java 8 开始，接口也可以有默认的方法实现，使得修改接口的成本也变的很低。



### super

- 访问父类的构造函数：可以使用 super() 函数访问父类的构造函数，从而委托父类完成一些初始化的工作。应该注意到，子类一定会调用父类的构造函数来完成初始化工作，一般是调用父类的默认构造函数，如果子类需要调用父类其它构造函数，那么就可以使用 super 函数。
- 访问父类的成员：如果子类重写了父类的某个方法，可以通过使用 super 关键字来引用父类的方法实现。



### 重写与重载



#### **重写**

**存在于继承体系中**，指子类实现了一个与父类在方法声明上完全相同的一个方法。

为了满足里式替换原则，重写有以下三个限制：

- 子类方法的访问权限必须大于等于父类方法；
- 子类方法的返回类型必须是父类方法返回类型或为其子类型。
- 子类方法抛出的异常类型必须是父类抛出异常类型或为其子类型。

使用 @Override 注解，可以让编译器帮忙检查是否满足上面的三个限制条件。

下面的示例中，SubClass 为 SuperClass 的子类，SubClass 重写了 SuperClass 的 func() 方法。其中：

- 子类方法访问权限为 public，大于父类的 protected。
- 子类的返回类型为 ArrayList，是父类返回类型 List 的子类。
- 子类抛出的异常类型为 Exception，是父类抛出异常 Throwable 的子类。
- 子类重写方法使用 @Override 注解，从而让编译器自动检查是否满足限制条件。

```java
class SuperClass {
    protected List<Integer> func() throws Throwable {
        return new ArrayList<>();
    }
}

class SubClass extends SuperClass {
    @Override
    public ArrayList<Integer> func() throws Exception {
        return new ArrayList<>();
    }
}
```

在调用一个方法时，先从本类中查找看是否有对应的方法，如果没有查找到再到父类中查看，看是否有继承来的方法。否则就要对参数进行转型，把参数类型转成父类之后看是否有对应的方法。总的来说，方法调用的优先级为：

- this.func(this)
- super.func(this)
- this.func(super)
- super.func(super)

```java
/*
    A
    |
    B
    |
    C
    |
    D
 */


class A {

    public void show(A obj) {
        System.out.println("A.show(A)");
    }

    public void show(C obj) {
        System.out.println("A.show(C)");
    }
}

class B extends A {

    @Override
    public void show(A obj) {
        System.out.println("B.show(A)");
    }
}

class C extends B {
}

class D extends C {
}
```

```java
public static void main(String[] args) {

    A a = new A();
    B b = new B();
    C c = new C();
    D d = new D();

    // 在 A 中存在 show(A obj)，直接调用
    a.show(a); // A.show(A)
    // 在 A 中不存在 show(B obj)，将 B 转型成其父类 A
    a.show(b); // A.show(A)
    // 在 B 中存在从 A 继承来的 show(C obj)，直接调用
    b.show(c); // A.show(C)
    // 在 B 中不存在 show(D obj)，但是存在从 A 继承来的 show(C obj)，将 D 转型成其父类 C
    b.show(d); // A.show(C)

    // 引用的还是 B 对象，所以 ba 和 b 的调用结果一样
    A ba = new B();
    ba.show(c); // A.show(C)
    ba.show(d); // A.show(C)
}
```



#### **重载**

存在于同一个类中，指一个方法与已经存在的方法名称上相同，但是参数类型、个数、顺序至少有一个不同。

应该注意的是，返回值不同，其它都相同不算是重载。





### 面向对象5个基本特性

- 万物皆为对象：把待解决的问题涉及到的物品，将其表示成程序中的对象。这些对象可以存储数据，还可以执行操作
- 程序是对象的集合，它们通过发送消息来告知彼此所要做的：要想请求一个对象，就必须对该对象发送一条消息。可以把消息当做对某个对象的方法调用。
- 每个对象都可以有由其他对象构成的存储：可以通过创建包含现有对象的方式来创建新的对象。因此，可以构建复杂的体系，同时将程序的复杂性隐藏在对象的简单性背后。
- 每个对象都有其类型：每个对象都是某个类的一个实例。每个类最重要的区别就是“可以发送怎样的消息给它”
- 某一特定类型的所有对象都可以接收同样的消息：





## Object 通用方法

### equals()

equals()方法具有如下特性：

- 自反性：`x.equals(x); // true`

- 对称性：`x.equals(y) == y.equals(x); // true`

- 传递性：

  ```java
  if (x.equals(y) && y.equals(z))
      x.equals(z); // true;
  ```

- 一致性：

  - 多次调用 equals() 方法结果不变

- 对任何不是 null 的对象 x 调用 x.equals(null) 结果都为 false



**==和equals**

- 对于基本类型，== 判断两个值是否相等，基本类型没有 equals() 方法。
- 对于引用类型，== 判断两个变量内存地址是否相等，而 equals() 判断引用的对象是否等价。



### hashCode()

hashCode() 返回散列值，而 equals() 是用来判断两个对象是否等价。

在覆盖 equals() 方法时应当总是覆盖 hashCode() 方法，保证等价的两个对象散列值也相等。



**为什么重写equals()一定要重写hashCode()？**

根据hashCode的规定，两个对象相等，那么他们的hashCode也一定相等。

假如只重写了equals方法，那么两个对象可能判断为相等，但是调用hashCode()方法返回的哈希值不相等，这就违反了前面说的规定。因此，重写了euqals方法一定要重写hashCode()方法。

参考: [为什么重写equals()一定要重写hashCode()方法](https://blog.csdn.net/xl_1803/article/details/80445481)





### clone()

clone() 是 Object 的 protected 方法，它不是 public。

应该注意的是，clone() 方法并不是 Cloneable 接口的方法，而是 Object 的一个 protected 方法。Cloneable 接口只是规定，如果一个类没有实现 Cloneable 接口又调用了 clone() 方法，就会抛出 CloneNotSupportedException。

#### 浅拷贝

拷贝对象和原始对象的属性引用同一个对象。

#### 深拷贝

拷贝对象和原始对象的属性引用不同对象。（也就是说引用不同的资源，拷贝的时候，连资源都拷贝了）



**clone() 的替代方案**

使用 clone() 方法来拷贝一个对象即复杂又有风险，它会抛出异常，并且还需要类型转换。Effective Java 书上讲到，最好不要去使用 clone()，可以使用拷贝构造函数或者拷贝工厂来拷贝一个对象。

```java
public class CloneConstructorExample {

    private int[] arr;

    public CloneConstructorExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    // 拷贝构造函数
    public CloneConstructorExample(CloneConstructorExample original) {
        arr = new int[original.arr.length];
        for (int i = 0; i < original.arr.length; i++) {
            arr[i] = original.arr[i];
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }
}
```





## 关键字



### final

- final修饰的引用不可以改变（但是引用指向的对象可以改变）
- final域能保证初始化过程的线程安全性
- final修饰的变量，值可以在运行时确定
- final修饰方法，该方法不能被重写
  - private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。
- final修饰类，该类不能被继承
- 类中所有private方法，都隐式的指定为final。因为子类无法取得private方法，自然也就无法覆盖它。final类中所有方法都被隐式的指定为final的



### static

- static不能修饰局部变量
- 即使没有显式的使用static关键字，构造函数实际上也是静态方法
- static{}  静态代码块 和 非静态代码块{} 的区别：
  - 静态代码块在使用该类时初始化，并且只初始化1次；非静态代码块，每次初始化对象，都会调用
  - 静态代码块使用使用静态变量、静态方法；非静态代码块可以使用静态、非静态变量。方法同理
- 静态变量：
  - 又称为类变量，也就是说这个变量属于类的。
  - 类所有的实例都共享静态变量，可以直接通过类名来访问它。
  - 静态变量在内存中只存在一份，**存在于方法区中**。
- 实例变量：
  - 每创建一个实例就会产生一个实例变量，它与该实例同生共死。
- 静态方法：
  - 静态方法在类加载的时候就存在了，它不依赖于任何实例。所以静态方法必须有实现，也就是说它不能是抽象方法。
  - 只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字。
- 静态语句块：
  - 静态语句块在类初始化时运行一次。
- 静态内部类：
  - 非静态内部类依赖于外部类的实例，而静态内部类不需要。
  - 静态内部类不能访问外部类的非静态的变量和 非静态的方法。
- **静态导包**
  - 在使用静态变量和方法时不用再指明 ClassName，从而简化代码，但可读性大大降低。



### 父类和派生类初始化

- 父类静态对象和静态代码块
- 子类静态对象和静态代码块
- 
- 父类非静态对象和非静态代码块
- 父类构造函数
- 
- 子类 非静态对象和非静态代码块
- 子类构造函数















## 类

类描述了相同特性（数据元素）和行为（功能）的对象的集合。在只有在第一次被使用时，才会加载，比如说创建类的对象 或者 使用类的static域。在进行类的加载的时候，会按static对象、static代码块在程序中的顺序进行初始化。

### 内部类

- 可以将一个类的定义放在另外一个类的内部，这就是内部类
- 内部类可以访问外围对象的所有成员。原因是当外围类的对象创建一个内部类的对象时，内部类会隐式的捕获一个执行外围类对象的引用，利用该引用来访问外围类的成员。



### 访问控制的原因

- 客户端程序员（使用类的程序员）无法触及它们不应该触及的部分
- 类库设计者可以改变类内部的工作方法，而不必担心影响到客户端程序员



### 设定边界的关键字

- public：任何人都可以访问该元素
- private：类的创建者（类内部编写代码的时候，可以访问private元素），类的内部方法 可以访问该元素（外部通过内部方法访问该元素）
- protected：和private差不多，差别仅在于继承的类可以访问父类的protected成员，不可以访问private成员。同一个包中，也可以访问其他类的protected属性。
- default默认访问权限：类可以访问在同一个包中的其他类的成员。

范围大小：public > protected > default > private



### 组合

使用现有的类，组成新的类，称为组合。优先使用组合，再使用继承。因为组合更加简单灵活。



### 函数调用

- 绑定：将一个方法调用 同一个方法主体 关联起来被称作**绑定**

- 前期绑定：若程序在执行前，方法和方法主体就已经绑定了，由编译器和链接程序实现，叫做**前期绑定**。一个非面向对象编程的编译器产生的函数调用，会引起前期绑定。

- 后期绑定：在运行时根据对象的类型进行绑定。Java中除了**static方法**和**final方法**（private方法被自动认为是final方法）之外，其他所有的方法都是后期绑定。**后期绑定是java中实现的多态的基础**。这个怎么实现呢？

  为了实现后期绑定，java使用一小段特殊代码来代替绝对地址的调用，特殊代码中：使用对象中存储的信息来计算方法体的地址。从而实现调用具体对象的某个方法。
  
  

### 类的成员

若类的某个成员没有初始化，java也会给其一个默认值。（java不会给局部变量一个默认值）

#### 成员方法

方法名、参数列表 合起来称为“方法签名”，它们唯一的标识出某个方法。

调用方法的行为，被称为发送消息给对象。

#### 重载函数的参数列表

- 直接写一个整数值，会被当做int处理

- 传入的参数类型（实际参数类型）小于方法中声明的形式参数类型，实际参数类型就会提升

- 在重载函数中，无法找到接收char参数的方法，就会把char直接提升到int

- 如果传入的参数类型比当前函数的参数类型大，就要做强制类型转换执行“窄化转换”。如:

  ```java
  void func(int);
  ....
  func((int)3.1f); 
  ```

- 根据方法名称、参数列表 区分重载方法。为什么不能根据返回值呢？  因为有些情况根据返回值，java无法判断调用哪一个函数。如：

  ```java
  void func();
  int func();
  
  // 调用的时候，无法判断调用哪一个函数
  func();     
  ```

  



### 类名冲突

java通过包路径 和 类名 区分类，如果这两个都一样，就会引起冲突。



## 继承

多个类具有相同的特征，可以把这些特征抽离出来，放到一个基类中，并且继承这个基类，子类称为**导出类**。任何发送给基类的消息，都可以发送给导出类。

把导出类转换成基类的过程，称为“向上转型”。

### 单根继承结构

所有类的继承，都继承自单一的基类的继承模式

优点：

- 保证所有对象都具备某些功能。因此你知道，在你的系统中你可以在每个对象上执行某些基本操作
- 所有对象都可以很容易地在堆上创建，而参数传递也得到了极大的简化
- 使**垃圾回收器**的实现变得容易得多
- 由于所有对象都保证具有其类型信息，因此不会因无法确定对象的类型而陷入僵局。这对于系统级操作（如异常处理）显得尤其重要，并且给编程带来了更大的灵活性。



## 存储

有5个不同的地方可以存储数据

### 寄存器

是最快的存储区，位于处理器内部。不能直接控制。存储数据量有限。

### 堆栈

![img](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1550315344150&di=d7ed0832e34104796ba241f684b19b2b&imgtype=0&src=http%3A%2F%2Fediterupload.eepw.com.cn%2Ffetch%2F20161101%2F322127_1_0.jpg)

实际上就是栈。堆栈位于通用的RAM（随机访问存储器）中，堆栈指针向下移动，则分配新的内存，若向上移动，则释放内存。

这是一种快速有效的分配存储方式，仅次于寄存器。

创建程序时，Java系统必须知道存储在堆栈中的数据的生命周期，以便上下移动堆栈指针，这一约束限制了程序的灵活性，因此java对象不存储于堆栈中。（对象的引用存储在堆栈中）。

**java虚拟机栈 和 操作系统的栈，有什么联系？**

个人认为java虚拟机栈是线程独有的，应该是属于操作系统栈 的一部分



### 堆

一种通用的内存池，也位于RAM区，用于存放所有的Java对象。堆不同于栈的好处：

- 编译器不需要知道存储的在堆里存活多次的时间，因此在堆里分配存储有很大的灵活性。
- 当需要一个对象时，只需要new写一行简单的代码。当执行这行代码时，会自动在堆里面进行存储分配。为这种灵活性会付出相应的代价，用堆进行存储和清理，可能比用堆栈花更多的时间



### 常量存储

常量值通常直接存放在方法区中。







### 数组

```java
int[] a = null; //声明数组，也是引用
a = new int[5]; //分配内存地址。
```

数组的引用，也就是a,当你在声明的时候，他会在栈中开辟一个地址空间。也就是第一步
第二步的作用，是在堆中开辟一系统连续的地址，具体的需要根据你的类型还有数组长度。

总结下：数组的引用保存在栈中。同时初始化实例的时候，在堆中开辟连续空间，栈中的引用指向堆的首地址。





## 对象的作用域

Java对象不具备和基本类型一样的生命周期。当new一个java对象时，它可以存活于作用域之外。如一下代码：

```java
{
    String s = new String("a string");
}  // 作用域终点
```

引用s在作用域终点就消失了，然而，s指向的String对象仍继续占据内存空间。（直到垃圾回收器回收）





## 对象的初始化过程

- java解释器查找类路径，定位到该类的class文件
- 载入class文件，执行静态变量的初始化
- 在堆上 为该对象分配足够的存储空间
- 初始化该存储空间：把对象中所有基本类型设置为默认值，比如int初始化为0。引用字段则设置为null
- 执行所有出现于字段定义出的初始化操作
- 执行构造器





## enum

实际上是一个类



## 多态

### 绑定

将一个方法调用 同 一个方法主体 关联起来被称作**绑定**。如果在程序执行前进行绑定，叫做**前期绑定**。

运行时根据对象的类型进行**绑定**，称为**后期绑定**，又称为**动态绑定**、**运行时绑定**。

java中除了static方法和final方法（private方法属于final方法的一种）之外，其他方法都是后期绑定。

java中多态是通过**动态绑定**实现的。

### 缺点

- 只有非private方法，才可以实现被覆盖，实现多态。
- 只有普通的方法调用是多态的，直接使用父类的属性，不具有多态性
- 如果某个类是静态的，它的行为不具有多态性。因为静态方法是和类关联的，并非和某个对象关联的。













## 字符编码

**不同的编码字节个数不一样的**

一个汉字在不同编码下占用字节数：

```java
1.GBK编码的字节数：2
2.UTF-8编码的字节数：3
3.Unicode编码的字节数：4
```



一个英文字母在不同编码下占用字节数：

```java
1.GBK编码的字节数：1
2.UTF-8编码的字节数：1
3.Unicode编码的字节数：4
```

因为Unicode编码下，一个字符无论什么情况下都占用4字节，为了节约内存空间，所以产生如UTF-8、GBK这样的编码。

```java
public class SizeDemo {

    public static void main(String[] args) {
        // 得到当前的系统属性
        String encoding = System.getProperty("file.encoding");
        System.out.println("当前编码:" + encoding);
        try {
            String str = "a";
            int len = str.getBytes().length;
            System.out.println("1.按操作系统默认编码来编码:" + len);

            len = str.getBytes("GBK").length;
            System.out.println("2.GBK编码的字节数："+ len);

            len = str.getBytes("UTF-8").length;
            System.out.println("3.UTF-8编码的字节数：" + len);

            len = str.getBytes("Unicode").length;
            System.out.println("4.Unicode编码的字节数：" + len);

        }  catch ( java.io.UnsupportedEncodingException e)  {
            System.out.println(e.getMessage().toString());
        }
    }

}
```

