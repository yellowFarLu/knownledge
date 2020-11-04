### 背景

Dubbo能够集成到spring中使用，问题是，spring怎么知道怎样去解析Dubbo的东西？



### 概览

大概可以分为4个步骤：

- Dubbo配置文件的定位（以我们常用的applicationContext.xml，里面引用Dubbo-config.xml为例子）

- 读取配置文件中Dubbo的配置

- 根据命名空间获取Dubbo命名空间处理器

- Dubbo命名空间解析器解析





### Dubbo配置文件的定位

我们常用的配置文件格式是xml，所以会用到org.springframework.web.context.support.XmlWebApplicationContext进行资源的加载。在该类的loadBeanDefinitions方法中进行beanDefinition的加载。

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 通过给定的beanFactory生成XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// 通过上下文配置是beanDefinitionReader
		// 加载环境
		beanDefinitionReader.setEnvironment(getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// 允许子类去提供自定义的初始化reader
		initBeanDefinitionReader(beanDefinitionReader);

		// 执行真正的加载beanDefinition
		loadBeanDefinitions(beanDefinitionReader);
	}
```



```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		/*
		 * 处理import标签，会取出resource字段的值，加载resource路径下的配置文件。并且把配置文件中的标签解析成beanDefinition
		 * 比如说处理 resource的值为"classpath*:/spring/Dubbo-config.xml"，则去该路径下加载配置文件Dubbo-config.xml
		 */
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```



### 读取配置文件中Dubbo的配置（DOM文件）

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			// 读取文件流程（将配置文件读取称为dom对象）
			Document doc = doLoadDocument(inputSource, resource);
			// 解析dom，注册beanDefinition
			return registerBeanDefinitions(doc, resource);
		}...
	}
```



**文件是获取到了，怎么进行解析呢？**

spring的做法是：Dubbo要集成到spring里面，那么Dubbo就得告诉spring怎么去解析。

Ioc容器初始化的时候，会将xml的内容转成BeanDefinition。这时候遇到Dubbo标签，就会获取到该标签对应的uri，然后通过uri获取“Dubbo命名空间处理器”，Dubbo命名空间处理器会把Dubbo标签解析成BeanDefinition并且注册到Ioc容器中。



### 根据命名空间获取Dubbo命名空间处理器

- Spring从META-INF/spring.handlers路径下面加载 命名空间uri 及 实现类名称 的 映射

- Dubbo实现了org.springframework.beans.factory.xml.NamespaceHandler接口org.apache.Dubbo.config.spring.schema.DubboNamespaceHandler。并且在META-INF/spring.handlers路径下面添加了映射文件

  ![Snip20181208_3](https://ws2.sinaimg.cn/large/006tNbRwgy1fxzpfmgl3vj31y80ng458.jpg)

- 解析XML文件的时候，发现Dubbo标签，尝试去获取Dubbo命名空间处理器

```java
@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		// 获取要解析元素的命名空间
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
		// 获取对应命名空间的NamespaceHandler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		// 利用NamespaceHandler进行解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```



- 获取Dubbo命名空间处理器的工具 NamespaceHandlerResolver

```java
/**
 * 根据映射文件中包含的映射将名称空间URI解析为实现类
 */
@FunctionalInterface
public interface NamespaceHandlerResolver {

	/**
	 * 解析命名空间URI并返回NamespaceHandler的具体实现
	 */
	@Nullable
	NamespaceHandler resolve(String namespaceUri);
}
```



实现类：org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver，**Dubbo命名空间处理器也是通过该类加载**，下面看下具体加载过程：

```java
public NamespaceHandler resolve(String namespaceUri) {
		// 加载命名空间处理器对应的映射关系(懒加载)
		Map<String, Object> handlerMappings = getHandlerMappings();
		// 根据命名空间的url，获取命名空间处理器
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			String className = (String) handlerOrClassName;
			try {
				// 获取该类的class对象
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
				// 实例化该类的对象
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
				// 调用init初始化命名空间处理器
				namespaceHandler.init();
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}...
			}
		}
	}
```



### Dubbo命名空间解析器解析

在org.springframework.beans.factory.xml.NamespaceHandlerSupport找到对应的解析器进行解析

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
		// 找到元素对应的解析器
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		// 进行解析
		return (parser != null ? parser.parse(element, parserContext) : null);
	}
```



在org.apache.Dubbo.config.spring.schema.DubboBeanDefinitionParser中进行真正的解析，并且注入到spring容器中

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
        ...
             /**
         * 如果是Dubbo:protocol标签，
         * Dubboh还会检查所有已经包含protocol属性的BeanDefinition，并且该protocol属性对应的值是ProtocolConfig对象的bean，
         * 将其属性的protocol值设置成当前的protocol bean的引用：
         *
         * Dubbo解析标签，按照顺序来的
         */
        if (ProtocolConfig.class.equals(beanClass)) {
            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
                if (property != null) {
                    Object value = property.getValue();
                    /**
                     * 如果我们设置protocol属性是"xxx"，这里的value就是字符串类型。
                     * 这里的用意应该是，当value是protocolConfig类型时，设置  将其属性的protocol值设置成当前的protocol bean的引用：
                     */
                    if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                        definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                    }
                }
            }
        } else if (ServiceBean.class.equals(beanClass)) {
            // 如果是Dubbo服务提供者的Dubbo:service标签，则还会设置ref属性为对应接口class的实现类bean
            String className = element.getAttribute("class");
            if (className != null && className.length() > 0) {
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                parseProperties(element.getChildNodes(), classDefinition);
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        } else if (ProviderConfig.class.equals(beanClass)) {
            // 如果是ProviderConfig类型的元素，还会对service类型的子元素进行解析、注册
            parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        } else if (ConsumerConfig.class.equals(beanClass)) {
            // 如果是ConsumerConfig类型的元素，还会对reference类型的子元素进行解析、注册
            parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
        }
        ...
```





经过解析之后，Spring就把Dubbo相关的beanDefinition注册到Ioc容器中了。而在真正进行依赖注入的时候，实际上注入的是服务的一个引用proxy实例。通过该proxy进行远程服务调用。详见：《Dubbo依赖注入远程服务》

![Snip20181208_1](https://ws3.sinaimg.cn/large/006tNbRwgy1fxzfdg4hfaj31pg0u0tn0.jpg)

