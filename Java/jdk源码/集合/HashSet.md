# HashSet



## 概述

HashSet一般用于去重，底层使用HashMap来实现





## 源码



### add()

![image-20200411183737881](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdq0gnvu17j316m07wwge.jpg)

插入元素的时候，其实往底层map里面插入元素，key为插入的元素，value为一个Object类型的对象。