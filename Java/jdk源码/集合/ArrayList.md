#ArrayList



## 总结

1）arrayList可以存放null。
2）arrayList本质上就是一个elementData数组。
3）arrayList区别于数组的地方在于能够自动扩展大小，其中关键的方法就是gorw()方法。
4）arrayList中removeAll(collection c)和clear()的区别就是removeAll可以删除批量指定的元素，而clear是全是删除集合中的元素。
5）arrayList由于本质是数组，所以它在数据的查询方面会很快，而在插入删除这些方面，性能下降很多，有移动很多数据才能达到应有的效果
6）arrayList实现了RandomAccess，所以在遍历它的时候推荐使用for循环。



##RandomAccess

RandomAccess 是一个标志接口，表明实现这个这个接口的 List 集合是支持快速随机访问的。也就是说，实现了这个接口的集合是支持 **快速随机访问** 策略的。

同时，官网还特意说明了，如果是实现了这个接口的 **List**，那么使用for循环的方式获取数据会优于用迭代器获取数据。

```java
for (int i=0, n=list.size(); i < n; i++)
 list.get(i);

for (Iterator i=list.iterator(); i.hasNext(); )
 i.next();
```





## 参考

https://www.cnblogs.com/zhangyinhua/p/7687377.html