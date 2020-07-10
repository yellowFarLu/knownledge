# Excel表列序号



## 问题

![image-20200627172838257](https://tva1.sinaimg.cn/large/007S8ZIlgy1gg6z6myverj30r20kimy5.jpg)

https://leetcode-cn.com/problems/excel-sheet-column-number/





## 分析

观察题目，A-Z这个规则实际上是26进制的表示，要转换为10进制的表达方式。

复习一下n进制转化为10进制的公式，比如说256这个n进制转化为10进制，则：

10进制整数 = 2\*n^2 + 5\*n^1 + 6\*n^0



## 代码

```java
public int titleToNumber(String s) {
        int result = 0;

        int pow = 1;

        for (int i = s.length() - 1; i >= 0; i--) {

            int charToInt = s.charAt(i) - 'A' + 1;

            result = result + (charToInt * pow);

            pow = pow * 26;
        }

        return result;
}
```

