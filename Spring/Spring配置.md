# Spring配置



## Bean的范围

bean标签的scope属性决定了bean的范围：

- singleton：单例模式，即该bean对应的类只有一个实例;在spring 中是scope(作用范围)参数的默认值 
- prototype：表示**每次从容器中取出bean时，都会生成一个新实例**
- request：基于web，**表示每来一个HTTP请求时，都会生成一个新实例**。同时该bean仅在当前HTTP request内有效
- session：表示在**每一个session中只有一个该对象**
- 自定义bean的作用域：
  在spring2.0中作用域是可以任意扩展的，你可以自定义作用域，甚至你也可以重新定义已有的作用域（但是你不能覆盖singleton和prototype），spring的作用域由接口org.springframework.beans.factory.config.Scope来定义，自定义自己的作用域只要实现该接口即可，下面给个实例：
  我们建立一个线程的scope，该scope在表示一个线程中有效，代码如下：

```java
publicclass MyScope implements Scope { 
      
      privatefinal ThreadLocal threadScope = new ThreadLocal() {
          protected Object initialValue() {
             returnnew HashMap(); 
           } 
     }; 
     
     public Object get(String name, ObjectFactory objectFactory) { 
       Map scope = (Map) threadScope.get(); 
       Object object = scope.get(name); 
       if(object==null) { 
         object = objectFactory.getObject(); 
         scope.put(name, object); 
       } 
       return object; 
     } 
     
     public Object remove(String name) { 
       Map scope = (Map) threadScope.get(); 
       return scope.remove(name); 
     }
     
     publicvoid registerDestructionCallback(String name, Runnable callback) { 
     }
     
     public String getConversationId() {
       // TODO Auto-generated method stub
       returnnull;
     } 
}
```

### 注意

scope配置为request、session的话，需要在web.xml中添加如下配置，才能开启：

```java
<listener-class>org.springframework.web.context.request.RequestContextListener</listener-class>
```

