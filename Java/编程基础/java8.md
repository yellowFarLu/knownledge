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







## 扩展

https://blog.csdn.net/lu930124/article/details/77595585/