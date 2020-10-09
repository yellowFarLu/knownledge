# spring-mvc



## 流程

- tomcat启动后，事件通知回调spring进行初始化
- 注册Handler
  - 创建Handler并且封装一层适配器
  - 把url和Handler注册到HandlerMapping中（HandlerMapping其实就是一个Map）
  - Controller里面的一个方法对应于一个Handler
  - 代码入口 org.springframework.web.servlet.handler.AbstractDetectingUrlHandlerMapping#initApplicationContext
- 请求执行
  - 请求到达前端控制器DispatcherServlet
  - DispatcherServlet到HandlerMapping查找Handler（通过url找到对应Handler）
  - DispatcherServlet调用适配器去执行Handler
    - 这里解释下为什么会多一层适配器，其实就是为了扩展。能够在请求调用前后进行一些操作。
  - handler执行完毕后，把ModelAndView提交给适配器
  - 适配器再把ModelAndView提交给DispatcherServlet
  - DispatcherServlet请求视图解析器去进行视图解析 （根据逻辑视图名解析成真正的视图，如jsp)
    - ViewResolver 的配置，从而将逻辑视图名解析为具体视图技术
  - DispatcherServlet渲染视图
  - 响应结果



![image-20191027190105898](https://tva1.sinaimg.cn/large/006y8mN6gy1g8cynmdehlj31660kkgvl.jpg)



## 上传文件

参考 ：https://blog.51cto.com/13902811/2175448



## 参考

[spring-mvc流程详解](https://www.cnblogs.com/leskang/p/6101368.html)

[spring-mvc源码解析](https://blog.csdn.net/win7system/article/details/90674757)

[4种HandlerMapping的使用方式](https://www.cnblogs.com/zhao307/p/5555597.html)

