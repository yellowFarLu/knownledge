# Java8



## 匿名类的lambda简写

比如说有如下匿名类：

```java
Consumer<Integer> consumer1 = new Consumer<Integer>() {
  @Override
  public void accept(Integer x) {
    int a = x + 2;
    System.out.println(a);// 12
    System.out.println(a + "_");// 12_
  }
};
```

使用lambda表达式进行简写，可以改为

```java
Consumer<Integer> consumer1 = x -> {
  int a = x + 2;
  System.out.println(a);// 12
  System.out.println(a + "_");// 12_
};
```

可以看到实现类的类名被省略了，并且唯一要实现的方法签名也被省略了。



### 进行方法调用

```java
public static void main(String[] args) {

  // 写法一
  Consumer<Integer> consumer = x -> {
    System.out.println(x);
  };

  ifInit(consumer);

  // 写法二
  ifInit(x -> System.out.println(x.longValue()));
}

static void ifInit(Consumer<Integer> consumer) {
  consumer.accept(10);
}
```







## Java8中的->

在java8之中，->可以表示匿名接口定义，也可以进行方法调用。在进行方法调用的时候，如果该方法调用作为另外一个方法A的入参，则自动转换为方法A的入参类型。

比如说有如下代码：

```java
List<String> arr = new ArrayList<>().stream().map(p-> p.toString()).collect(Collectors.toList());
```

点击进去可以看到箭头这个实现类实现的接口为

![image-20210110152704103](https://tva1.sinaimg.cn/large/008eGmZEly1gmimqwf596j31c40scn1v.jpg)

而点击进去map方法，查看入参的类型正是Function

![image-20210110152756717](https://tva1.sinaimg.cn/large/008eGmZEly1gmimrqizxtj31bk0isn16.jpg)

由此得证明。



## paralleStream

参考：https://www.cnblogs.com/pengzhizhong/p/10191842.html





## flatMap

参考：





## Collectors.toMap

【强制】在使用`java.util.stream.Collectors`类的`toMap()`方法转为`Map`集合时，一定要使用含有参数类型为`BinaryOperator`，参数名为`mergeFunction`的方法，否则当出现相同key值时会抛出IllegalStateException异常。
说明：参数mergeFunction的作用是当出现key重复时，自定义对value的处理策略。
正例：

```java
List<Pair<String, Double>> pairArrayList = new ArrayList<>(3);
pairArrayList.add(new Pair<>("version", 6.19));
pairArrayList.add(new Pair<>("version", 10.24));
pairArrayList.add(new Pair<>("version", 13.14));
Map<String, Double> map = pairArrayList.stream().collect(
// 生成的map集合中只有一个键值对：{version=13.14}
Collectors.toMap(Pair::getKey, Pair::getValue, (v1, v2) -> v2));

// (v1, v2) -> v2 其实是告诉JDK在遇到重复元素的时候怎么处理，这里是取第二个。
```

反例：

```java
String[] departments = new String[] {"iERP", "iERP", "EIBU"};
// 抛出IllegalStateException异常
Map<Integer, String> map = Arrays.stream(departments)
    .collect(Collectors.toMap(String::hashCode, str -> str));
```

参考：https://blog.csdn.net/neweastsun/article/details/80294811





## Filter过滤

```java
public static void main(String[] args) {
  OuterTenantInfo one = new OuterTenantInfo();
  one.setOuterUid(1);
  one.setName("one");

  OuterTenantInfo two = new OuterTenantInfo();
  two.setOuterUid(2);
  two.setName("two");

  OuterTenantInfo three = new OuterTenantInfo();
  three.setOuterUid(3);
  three.setName("three");

  List<OuterTenantInfo> accounts = new ArrayList<>();
  accounts.add(one);
  accounts.add(two);
  accounts.add(three);

  List<OuterTenantInfo> newList = accounts.stream().filter(item -> item.getOuterUid() > 1).collect(Collectors.toList());
  System.out.println(newList);
}
```

输出：

```json
[OuterTenantInfo{outerUid=2, name='two'}, OuterTenantInfo{outerUid=3, name='three'}]
```

https://blog.csdn.net/xinyang12138/article/details/86526540





## 扩展

https://blog.csdn.net/lu930124/article/details/77595585/