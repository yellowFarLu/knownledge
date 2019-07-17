### 标签

标签是后面跟有冒号的标识符，如 label:



标签和continue一起使用，会直接跳转到标签处，并重新进入标签后面的那个循环

```java
public static void main(String[] args) {
    lable:

    for (int i = 0; i < 10; i++) {
        for (int j = 0; j < 10; j++) {
            System.out.println("in for, i=" + i);
            continue lable;
        }
    }

    System.out.println("out for");
}
```

执行结果：

in for, i=0
in for, i=1
in for, i=2
in for, i=3
in for, i=4
in for, i=5
in for, i=6
in for, i=7
in for, i=8
in for, i=9
out for



标签和break一起使用，会直接中断所有迭代，并回到label处，并不重新进入迭代

```java
public static void main(String[] args) {
    lable:

    for (int i = 0; i < 10; i++) {
        for (int j = 0; j < 10; j++) {
            System.out.println("in for, i=" + i);
            break lable;
        }
    }

    System.out.println("out for");
}
```

执行结果：

in for, i=0
out for



**Java中使用标签的唯一理由，是有循环嵌套存在，并且想从多重嵌套中break或者continue**

