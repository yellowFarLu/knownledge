# http转dubbo



## 背景

测试同学想对dubbo服务进行测试，但是他们并不会编写消费者代码，为了方便测试，因此将dubbo暴露为http请求，测试同学就可以使用postman进行测试了。

有同学可能问：我将dubbo请求外面使用springmvc包装多一层，测试同学也能够使用http请求测试dubbo服务啊，为什么还要专门http转dubbo？     因为dubbo接口数量很多，总不能每一个接口都包装一下，需要一个统一简洁的办法。

<br/><br/>



## 泛化调用



### 泛化调用实现http转dubbo

编写接入层，接收http请求，转化为dubbo请求，然后获取dubbo调用结果，将其转化为http响应，返回给客户端。

http转dubbo使用dubbo泛化调用的功能。泛化调用指的是dubbo消费者并没有提供者的接口。下面是通过泛化调用实现http转dubbo。

```java
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.alibaba.dubbo.common.utils.IOUtils;
import com.alibaba.dubbo.config.ApplicationConfig;
import com.alibaba.dubbo.config.ReferenceConfig;
import com.alibaba.dubbo.config.RegistryConfig;
import com.alibaba.dubbo.rpc.service.GenericService;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

public class DubboToHttpServlet extends HttpServlet {

    private static final long serialVersionUID = 1L;

    private static Logger logger = LoggerFactory.getLogger(DubboToHttpServlet.class);

    private ApplicationConfig application = new ApplicationConfig("dubbo-to-http");
    private Map<String, GenericService> serviceCache = new HashMap<String, GenericService>();

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        logger.info("servlet start");

        // 1. 解析请求体
        logger.info("parsing request body");
        JSONObject requestJson = JSON.parseObject(IOUtils.read(new InputStreamReader(request.getInputStream())));
        logger.info("parsed request body. body json: {}", requestJson);

        // 2. 获取泛化服务接口
        logger.info("fetching generic service");
        GenericService service = this.fetchGenericService(requestJson);
        logger.info("fetched generic service. service: {}", service);

        // 3. 组装调用参数
        String method = requestJson.getString("method");
        String[] parameterTypes = this.toArray(requestJson.getJSONArray("paramTypes"));
        Object[] args = requestJson.getJSONArray("paramValues").toArray(new Object[] {});

        // 4. 调用接口
        logger.info("invoking remote service");
        String result = JSON.toJSONString(service.$invoke(method, parameterTypes, args));
        logger.info("invoked remote service. return: {}", result);

        // 5. 返回
        response.setCharacterEncoding("UTF-8");
        response.setContentType("application/json; charset=utf-8");
        PrintWriter out = response.getWriter();
        out.append(result).flush();
        out.close();

        logger.info("servlet end");
    }

    @Override
    public void destroy() {
        RegistryConfig.destroyAll();
    }

    // 获取泛化服务接口. 如有缓存, 从缓存取
    private GenericService fetchGenericService(JSONObject requestJson) {
        String serviceInterface = requestJson.getString("interface");
        String serviceGroup = requestJson.getString("group");
        String serviceVersion = requestJson.getString("version");

        String serviceCacheKey = serviceInterface + serviceGroup + serviceVersion;
        GenericService service = serviceCache.get(serviceCacheKey);
        if (service != null) {
            logger.info("fetched generic service from cache");
            return service;
        }

        logger.info("initing generic service");
        ReferenceConfig<GenericService> reference = new ReferenceConfig<GenericService>();
        reference.setApplication(application);
        reference.setInterface(serviceInterface);
        reference.setGroup(serviceGroup);
        reference.setVersion(serviceVersion);
        reference.setGeneric(true);
        service = reference.get();
        serviceCache.put(serviceCacheKey, service);

        return service;
    }

    // List<Object> -> String[]
    private String[] toArray(List<Object> list) {
        String[] array = new String[list.size()];
        for (int i = 0; i < array.length; i++) {
            array[i] = list.get(i).toString();
        }
        return array;
    }
}
```

```java
客户端http调用消息格式：
{
    "registry": "zookeeper://***.**.***:2181",
    "interface": "com.ihome.basicbiz.*****Service",
    "version": "",
    "group": "",
    "method": "payout",
    "paramTypes": ["com.ihome.basicbiz****.TrustaccPayoutParam", "java.lang.String"],
    "paramValues": [{
        "payeeBankObProvinceName": "",
        "payeeBankObCity": "",
        "payeeBankObCityName": "",
        "payeeBankObSubbranch": "",
        "trustaccSource": "CASHIER",
        "tradeType": 2,
        "transcoreTradeType": 2,
        "useDesc": "",
        "remark": "",
        "payinPayeeBankcardNo": "",
        "transType": "",
        "detailList": [{
            "customerId": 222001,
            "amount": 100,
            "projectCode": "1101000010438",
            "outPayNo": "OUT_PAY_NO_001",
            "outFreezeNo": ""
        }],
        "productNo": "",
     
    }, "123"]
}
```



#### 优点

- 提供者不需要改动



#### 缺点

- 需要传递参数类型，参数值。参数传递复杂，不方便调用







### 泛化调用实战

泛化调用的情况下，提供者无需改动，因此下面介绍消费方的改动

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo" xmlns:dubbbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <dubbo:reference
            id="demoService"
            interface="com.huang.yuan.api.service.DemoService"
            version="1.0"
            timeout="1000000"
            generic="true">
    </dubbo:reference>
</beans>
```

注意，这时候的消费者并没有引入提供者的jar包。所以interface属性会飘红是正常的，另外需要配generic="true"，表示接口支持泛化调用。

然后就是泛化调用的代码，如下：

```java
import com.alibaba.dubbo.rpc.service.GenericService;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * @author huangy on 2020-04-08
 */
public class TestMain {

    /**
     * Spring泛化调用
     */
    public static void main(String[] args) {

        ClassPathXmlApplicationContext context =
                new ClassPathXmlApplicationContext("classpath:/spring/applicationContext.xml");
        context.start();

        GenericService demoService = (GenericService) context.getBean("demoService");

        Object result = demoService.$invoke(
                "test", new String[] { "java.lang.String" }, new Object[]{"你好"});

        System.out.println(result);
    }

}
```

启动提供者后，消费者进行泛化调用，输出结果如下：

```java
{result=你好, success=true, errorMessage=success, errorCode=0, class=com.huang.yuan.api.model.ModelResult}
```



### 原理

进行远程调用的时候，携带接口名称、参数。然后提供方根据接口名称、参数找到对应的invoker进行远程调用。

具体源码没有找到。





<br/><br/>





## 提供方支持http暴露

泛化调用的话，传递参数复杂，因此有了另外一种调用方式，也就是提供方provider直接支持http暴露服务，这样子调用方可以直接使用http调用dubbo服务了。



### 实现

一开始我们的服务是通过tomcat进行暴露的，如果访问dubbo的话，过程就是 `http --> nginx ---> tomcat ---> springmvc ---> dubbo` 这个过程，而现在我们需要做的就是将这个过程中的SpringMVC这一块进行移除，变成 `http --> nginx ---> tomcat ---> dubbo`，从而直接支持http被dubbo处理。于是我通过SpringMVC的处理机制将Controller这一块移除掉，达到了我们的目的，接下来看如何一步一步实现的.

假设url = `/dubbo/*`



- 通过包装Servlet统一处理对接的http请求

  - ```java
    <servlet>
            <servlet-name>GatewayServlet</servlet-name>
            <servlet-class>com.xxx.gateway.dubbo.web.GatewayServlet</servlet-class>
        </servlet>
        <servlet-mapping>
            <servlet-name>GatewayServlet</servlet-name>
            <url-pattern>/dubbo/*</url-pattern>
        </servlet-mapping>
    ```

- 通过http参数获取dubbo服务的接口，版本等等信息

  - ```java
    String inf = servletRequest.getParameter("serviceName");
    Assert.notNull(inf, "接口不能为空!");
    
    String method = servletRequest.getParameter("method");
    Assert.notNull(method, "方法不能为空!");
    
    String uGroup = servletRequest.getParameter("group");
    
    String vVersion = servletRequest.getParameter("version");
    Assert.notNull(vVersion, "版本不能为空!");
    ```

- 获取dubbo服务

  - ```java
    String[] beanNamesForType = this.applicationContext.getBeanNamesForType(ServiceConfig.class);
        Object ref = null;
        for (String service : beanNamesForType) {
            ServiceConfig serviceConfig = (ServiceConfig) this.applicationContext.getBean(service);
            String version = serviceConfig.getVersion();
            String anInterface = serviceConfig.getInterface();
            String group = serviceConfig.getGroup();
            if (!inf.equalsIgnoreCase(anInterface)) {
                continue;
            }
            if (!vVersion.equalsIgnoreCase(version)) {
                continue;
            }
            if (uGroup != null && !group.equalsIgnoreCase(uGroup)) {
                continue;
            }
            ref = serviceConfig.getRef();
            break;
        }
    ```

- 利用SpringMVC的的处理机制将Controller移除

  - ```java
    try {
                HttpServletRequest req = (HttpServletRequest) servletRequest;
                HttpServletResponse resp = (HttpServletResponse) servletResponse;
                ServletInvocableHandlerMethod invocableMethod = new ServletInvocableHandlerMethod(handlerMethod);
                WebDataBinderFactory binderFactory = getDataBinderFactory(invocableMethod);
    
                invocableMethod.setDataBinderFactory(binderFactory);
                ServletWebRequest webRequest = new ServletWebRequest(req, resp);
                ModelAndViewContainer mavContainer = new ModelAndViewContainer();
    
                HandlerMethodArgumentResolverComposite handlerMethodArgumentResolverComposite = new HandlerMethodArgumentResolverComposite();
                handlerMethodArgumentResolverComposite.addResolvers(getDefaultArgumentResolvers());
                invocableMethod.setHandlerMethodArgumentResolvers(handlerMethodArgumentResolverComposite);
    
                List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
                HandlerMethodReturnValueHandlerComposite returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
                invocableMethod.setHandlerMethodReturnValueHandlers(returnValueHandlers);
    
                invocableMethod.invokeAndHandle(webRequest, mavContainer);
                methodMap.put(handlerMethod, invocableMethod);
            } catch (Exception e) {
                fail(servletResponse, e);
            }
    ```

- 如何突破传递参数的问题。 利用SpringMVC的 `HandlerMethodArgumentResolver` 即可解析
  到这一步，http转dubbo就处理好了。







### 优点

- 传参简单，仅需要传递参数本身，不需要传递参数类型



### 缺点

- 提供者的代码需要支持dubbo暴露为http服务













## 参考

[使用http请求dubbo](https://zhuanlan.zhihu.com/p/72596602)

[http转dubbo调用（泛化方式）](https://blog.csdn.net/yufeng702/article/details/72911575)

