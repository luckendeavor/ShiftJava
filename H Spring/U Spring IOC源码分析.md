[TOC]

### IOC源码分析

本文源码版本是 4.3.11.RELEASE。为了降低难度，本文所说的所有的内容都是基于 **xml** 的配置方式，实际使用已经很少人这么做了，至少不是纯 xml 配置，不过从理解源码的角度来看用这种方式来说无疑是最合适的。

IOC 总体来说有两处地方最重要，一个是**创建 Bean 容器，一个是初始化 Bean**。

#### 引入

这里使用 **ClassPathXmlApplicationContext** 进行分析。先来一个例子来看**怎么实例化 ApplicationContext**。

首先定义一个接口：

```java
public interface MessageService {
    String getMessage();
}
```

定义接口**实现类**：

```java
public class MessageServiceImpl implements MessageService {

    public String getMessage() {
        return "hello world";
    }
}
```

接下来在 **resources** 目录新建一个**配置文件**，文件名通常叫 application.xml 或 application-xxx.xml 就可以了：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd" default-autowire="byName">

    <bean id="messageService" class="com.javadoop.example.MessageServiceImpl"/>
</beans>
```

这样就可以跑起来了：

```java
public class App {
    public static void main(String[] args) {
        // 用配置文件来启动一个ApplicationContext
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
        System.out.println("context 启动成功");
        // 从context中取出我们的 Bean，而不是用new MessageServiceImpl()这种方式
        MessageService messageService = context.getBean(MessageService.class);
        // 这句将输出:hello world
        System.out.println(messageService.getMessage());
    }
}
```

重点是下面：

```
ApplicationContext context = new ClassPathXmlApplicationContext(...);
```

ClassPathXmlApplicationContext 从名字上就可以猜出是**在 ClassPath 中寻找 xml 配置文件**，根据 xml 文件内容来构建 **ApplicationContext**。当然，除了 ClassPathXmlApplicationContext 以外，也还有其他**构建 ApplicationContext** 的方案，大体的继承结构如下：

<img src="assets/image-20200528121417326.png?lastModify=1594803367" alt="image-20200528121417326" style="zoom:40%;" />

可以大致看一下类名，源码分析的时候不至于找不着看哪个类，因为 Spring 为了适应**各种使用场景**，提供的各个**接口**都可能有**很多的实现类**。一般分析就是**揪着一个完整的分支看完**。

可以看到 ClassPathXmlApplicationContext 兜兜转转了好久才到 **ApplicationContext 接口**，同样的也可以使用绿颜色的 **FileSystemXmlApplicationContext** 和 **AnnotationConfigApplicationContext** 这两个类。

- **FileSystemXmlApplicationContext** 的构造函数需要一个 **xml 配置文件**在**系统中的路径**，其他和 ClassPathXmlApplicationContext 基本上一样。

- **AnnotationConfigApplicationContext** 是**基于注解**来使用的，它不需要配置文件，采用 Java 配置类和各种注解来配置，是比较简单的方式。

这里为了简便则使用 ClassPathXmlApplicationContext 来进行分析，需要关注**怎么样通过配置文件来启动 Spring 的 ApplicationContext** ，也就是分析 IOC 的核心了。**ApplicationContext 启动过程中，会负责创建实例 Bean，往各个 Bean 中注入依赖**等。

#### BeanFactory简介

BeanFactory，从名字上也很好理解，**生产 bean 的工厂，它负责生产和管理各个 bean 实例**。**ApplicationContext 其实就是一个 BeanFactory**。下面是和 BeanFactory 接口相关的主要的继承结构：

<img src="assets/image-20200528122324788.png" alt="image-20200528122324788" style="zoom:40%;" />

**ApplicationContext 往下**的继承结构前面一张图已经有了，要点如下：

1. **ApplicationContext 继承了 ListableBeanFactory**，这个 Listable 的意思就是通过这个接口**可以获取多个 Bean**，最顶层 BeanFactory 接口的方法都是**获取单个 Bean** 的。
2. **ApplicationContext 继承了 HierarchicalBeanFactory**，Hierarchical 单词本身已经能说明问题了，也就是说我们可以在应用中**起多个 BeanFactory**，然后可以将各个 BeanFactory 设置为**父子关系**。
3. AutowireCapableBeanFactory 这个名字中的 Autowire 很常见，就是用来**自动装配 Bean 用的**，但是上图中的ApplicationContext 并没有继承它，不使用继承，不代表不可以**使用组合**，看到 ApplicationContext 接口定义中的最后一个方法 **getAutowireCapableBeanFactory**() 就知道了。
4. ConfigurableListableBeanFactory 也是一个**特殊的接口**，特殊之处在于它继承了**第二层所有的三个接口**，而 ApplicationContext 没有。这点之后会用到。

#### 启动过程分析

第一步从 **ClassPathXmlApplicationContext** 的**构造方法**说起。

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    private Resource[] configResources;

    // 如果已经有ApplicationContext并需要配置成父子关系，那么调用这个构造方法
    public ClassPathXmlApplicationContext(ApplicationContext parent) {
        super(parent);
    }
    //...
    public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) throws BeansException {

        super(parent);
        // 根据提供的路径，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
        setConfigLocations(configLocations);
        if (refresh) {
            // 注意：核心方法！！
            refresh(); 
        }
    }
    // ...
}
```

接下来就是 **refresh() 方法**，这里简单说下为什么是 refresh()，而不是 init() 这种名字的方法。因为 ApplicationContext **建立**起来以后，其实是可以通过调用 refresh() 这个方法**重建**的，refresh() 会将原来的 ApplicationContext **销毁**，然后再重新执行一次初始化操作。

refresh() 方法里面调用了那么多方法，非常重要。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   // 来个锁，不然refresh()还没结束又来个启动或销毁容器的操作，那就炸了
   synchronized (this.startupShutdownMonitor) {
       
      // 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
      prepareRefresh();

      // 这步比较关键，这步完成后，配置文件就会解析成一个个Bean定义，注册到BeanFactory中，
      // 当然这里说的Bean还没有初始化，只是配置信息都提取出来了，注册也只是将这些信息都保存
      // 到了注册中心中(说到底核心是一个beanName -> beanDefinition的map)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 设置BeanFactory的类加载器，添加几个BeanPostProcessor，手动注册几个特殊的bean
      prepareBeanFactory(beanFactory);

      try {
         // 需要知道BeanFactoryPostProcessor这个知识点，Bean如果实现了此接口，
         // 那么在容器初始化以后，Spring会负责调用里面的postProcessBeanFactory方法。
         // 这里是提供给子类的扩展点，到这里的时候，所有的Bean都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的BeanFactoryPostProcessor的实现类或做点什么事
         postProcessBeanFactory(beanFactory);
         // 调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory(factory)方法
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册BeanPostProcessor的实现类，注意看和BeanFactoryPostProcessor的区别
         // 此接口两个方法: postProcessBeforeInitialization和postProcessAfterInitialization
         // 两个方法分别在Bean初始化之前和初始化之后得到执行。注意，到这里Bean还没初始化
         registerBeanPostProcessors(beanFactory);

         // 初始化当前ApplicationContext的MessageSource，国际化就不展开了
         initMessageSource();

         // 初始化当前ApplicationContext的事件广播器，这里也不展开了
         initApplicationEventMulticaster();

         // 从方法名就可以知道，典型的模板方法(钩子方法)，
         // 具体的子类可以在这里初始化一些特殊的Bean（在初始化singleton beans之前）
         onRefresh();

         // 注册事件监听器，监听器需要实现ApplicationListener接口。这也不是我们的重点，过
         registerListeners();

         // 重点重点重点：初始化所有的Singleton Beans（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);

         // 最后广播事件，ApplicationContext初始化完成
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }
         // 销毁已经初始化的singleton的Beans，以免有些bean会一直占用资源
         destroyBeans();

         // 重置active标志位
         cancelRefresh(ex);

         // 把异常往外抛
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

下面一步步来看这个 **refresh**() 方法。

##### 1. 创建Bean容器前的准备工作

即 **prepareRefresh**() 方法，准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符。这个比较简单，直接看代码中的**几个注释**即可。

```java
protected void prepareRefresh() {
   // 记录启动时间，将active属性设置为true，closed属性设置为false，它们都是AtomicBoolean类型
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   if (logger.isInfoEnabled()) {
      logger.info("Refreshing " + this);
   }

   // Initialize any placeholder property sources in the context environment
   initPropertySources();

   // 校验xml配置文件
   getEnvironment().validateRequiredProperties();
   this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

##### 2. 创建Bean容器，加载并注册Bean

###### (1) obtainFreshBeanFactory方法

回到 refresh() 方法中的下一行 **obtainFreshBeanFactory**()。看方法名就是**获取新鲜的 BeanFactory**。注意，**这个方法是全文最重要的部分之一**，这里将会**初始化 BeanFactory、加载 Bean、注册 Bean** 等等。这步结束后，**Bean 并没有完成初始化**。这里指的是 Bean **实例**并未在这一步生成。

// **AbstractApplicationContext.java**

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 关闭旧的BeanFactory(如果有)，创建新的BeanFactory，加载Bean定义、注册Bean等等!!
    refreshBeanFactory();

    // 返回刚刚创建的BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

refreshBeanFactory() 方法由实现类实现。如下：

// **AbstractRefreshableApplicationContext**.java 120 行。

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
    // 如果ApplicationContext中已经加载过BeanFactory了，销毁所有Bean，关闭BeanFactory
    // 注意，应用中BeanFactory本来就是可以多个的，这里可不是说应用全局是否有BeanFactory，而是当前
    // ApplicationContext是否有BeanFactory
    if (hasBeanFactory()) {
        // 销毁已存在的Bean
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 初始化一个DefaultListableBeanFactory，为什么用这个，我们马上说。
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 用于BeanFactory的序列化，我想不部分人应该都用不到
        beanFactory.setSerializationId(getId());

        // 下面这两个方法很重要，别跟丢了，具体细节之后说
        // 设置BeanFactory的两个配置属性：是否允许Bean覆盖、是否允许循环引用
        customizeBeanFactory(beanFactory);

        // 加载Bean到BeanFactory中
        loadBeanDefinitions(beanFactory);
        // 加锁设置beanFactory
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

看到这里就应该**站在高处看 ApplicationContext** 了，ApplicationContext 继承自 BeanFactory，但是它**不应该被理解为 BeanFactory 的实现类**，而是说**其内部持有一个实例化的 BeanFactory（DefaultListableBeanFactory）**。以后所有的 BeanFactory 相关的操作其实是**委托给这个实例**来处理的。

为什么选择实例化 **DefaultListableBeanFactory** ？前面有个很重要的接口 **ConfigurableListableBeanFactory**，它**实现**了 BeanFactory 下面一层的**所有三个接口**，之前的继承图再看一下：

<img src="assets/image-20200528125958383.png" alt="image-20200528125958383" style="zoom:40%;" />

可以看到 ConfigurableListableBeanFactory 只有一个**实现类 DefaultListableBeanFactory**，而且实现类 DefaultListableBeanFactory 还通过实现右边的 AbstractAutowireCapableBeanFactory 通吃了右路。所以结论就是，最底下这个家伙 **==DefaultListableBeanFactory 基本上是最牛的 BeanFactory==** 了，这也是为什么这边会使用这个类来实例化的原因。

如果想要在程序运行的时候**动态**往 Spring IOC 容器**注册新的 bean**，就会使用到这个类。那怎么在**运行时获得这个实例**呢？

之前说过 **ApplicationContext 接口**能获取到 **AutowireCapableBeanFactory**，就是最右上角那个，然后它**向下转型**就能得到 **DefaultListableBeanFactory** 了。

###### (2) BeanDefinition

在继续往下之前需要**先了解 BeanDefinition**。**BeanFactory 是 Bean 容器，那么 Bean 又是什么呢？**这里的 BeanDefinition 就是所说的 Spring 的 Bean，自己定义的各个 Bean 其实**会转换成一个个 BeanDefinition 存在于 Spring 的 BeanFactory 中**。Bean 在代码层面上可以**简单认为是 BeanDefinition 的实例**。

> **BeanDefinition 中保存了 Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等等。**

###### (3) BeanDefinition接口定义

看下 BeanDefinition 的接口定义：

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

    // 可以看到默认只提供sington和prototype两种，
    // 其实还有request,session,globalSession,application,websocket这几种，
    // 不过它们属于基于web的扩展
    String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
    String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

    // 比较不重要直接跳过
    int ROLE_APPLICATION = 0;
    int ROLE_SUPPORT = 1;
    int ROLE_INFRASTRUCTURE = 2;

    // 设置父Bean，这里涉及到bean继承，不是java继承。
    // 一句话就是：继承父Bean的配置信息而已
    void setParentName(String parentName);

    // 获取父Bean
    String getParentName();

    // 设置Bean的类名称，将来是要通过反射来生成实例的
    void setBeanClassName(String beanClassName);
    // 获取Bean的类名称
    String getBeanClassName();

    //设置bean的scope
    void setScope(String scope);
    String getScope();

    // 设置是否懒加载
    void setLazyInit(boolean lazyInit);
    boolean isLazyInit();

    // 设置该Bean依赖的所有的Bean，注意，这里的依赖不是指属性依赖(如 @Autowire 标记的)，
    // 是 depends-on="" 属性设置的值。
    void setDependsOn(String... dependsOn);
    // 返回该 Bean 的所有依赖
    String[] getDependsOn();

    // 设置该Bean是否可以注入到其他Bean中，只对根据类型注入有效，
    // 如果根据名称注入，即使这边设置了false，也是可以的
    void setAutowireCandidate(boolean autowireCandidate);
    // 该Bean是否可以注入到其他Bean中
    boolean isAutowireCandidate();

    // 主要的。同一接口的多个实现，如果不指定名字的话，Spring会优先选择设置primary为true的bean
    void setPrimary(boolean primary);
    boolean isPrimary();

    // 如果该Bean采用工厂方法生成，指定工厂名称。对工厂不熟悉的读者，请参加附录
    // 一句话就是：有些实例不是用反射生成的，而是用工厂模式生成的
    void setFactoryBeanName(String factoryBeanName);
    // 获取工厂名称
    String getFactoryBeanName();
    // 指定工厂类中的工厂方法名称
    void setFactoryMethodName(String factoryMethodName);
    // 获取工厂类中的工厂方法名称
    String getFactoryMethodName();

    // 构造器参数
    ConstructorArgumentValues getConstructorArgumentValues();

    // Bean中的属性值，后面给bean注入属性值的时候会说到
    MutablePropertyValues getPropertyValues();

    // 是否singleton
    boolean isSingleton();

    // 是否prototype
    boolean isPrototype();

    // 如果这个Bean是被设置为abstract，那么不能实例化，
    // 常用于作为 父bean 用于继承，其实也很少用......
    boolean isAbstract();

    int getRole();
    String getDescription();
    String getResourceDescription();
    BeanDefinition getOriginatingBeanDefinition();
}
```

这个 BeanDefinition 其实已经包含很多的信息了。这里接口虽然那么多，但是**没有类似 getInstance() 这种方法来获取定义的类的实例**，**真正定义的类**生成的实例到哪里去了呢？

有了 BeanDefinition 的概念以后，再往下看 **refreshBeanFactory**() 方法中的剩余部分：

```java
customizeBeanFactory(beanFactory);
loadBeanDefinitions(beanFactory);
```

虽然只有两个方法，但还有很多。

###### (4) customizeBeanFactory

customizeBeanFactory(beanFactory) 比较简单，就是**配置是否允许 BeanDefinition 覆盖、是否允许循环引用**。

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    if (this.allowBeanDefinitionOverriding != null) {
        // 是否允许Bean定义覆盖
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.allowCircularReferences != null) {
        // 是否允许Bean间的循环依赖
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```

BeanDefinition 的**覆盖问题**可能会有开发者碰到这个坑，就是在配置文件中**定义 bean 时使用了相同的 id 或 name**，默认情况下，allowBeanDefinitionOverriding 属性为 null，如果在同一配置文件中重复了，会抛错，但是如果不是同一配置文件中，会发生覆盖。

循环引用也很好理解：A 依赖 B，而 B 依赖 A。或 A 依赖 B，B 依赖 C，而 C 依赖 A。

默认情况下，**Spring 允许循环依赖**，当然如果在 A 的构造方法中依赖 B，在 B 的构造方法中依赖 A 是不行的。对于覆盖问题，很多人都希望禁止出现 Bean 覆盖，可是 Spring 默认是不同文件的时候可以覆盖的。之后的源码中还会出现这两个属性，有个印象就可，它们不是非常重要。

###### (5) 加载Bean: loadBeanDefinitions

接下来是**==最重要==**的 **loadBeanDefinitions**(beanFactory) 方法了，这个方法将**根据配置加载各个 Bean，然后放到 BeanFactory 中**。

读取配置的操作在 **XmlBeanDefinitionReader** 中，它负责加载配置、解析。

**// AbstractXmlApplicationContext.java 80**

```java
/** 此方法将通过一个XmlBeanDefinitionReader实例来加载各个Bean*/
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 给这个BeanFactory 实例化一个XmlBeanDefinitionReader
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // 初始化BeanDefinitionReader，其实这个是提供给子类覆写的，
    initBeanDefinitionReader(beanDefinitionReader);
    // 重点来了，继续往下
    loadBeanDefinitions(beanDefinitionReader);
}
```

现在还在这个类中，接下来用刚刚**初始化的 Reader 开始来加载 xml 配置**，这里可以选择性跳过，不是很重要。

// AbstractXmlApplicationContext.java 120

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        // 往下看
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        // 2
        reader.loadBeanDefinitions(configLocations);
    }
}

// 上面虽然有两个分支，不过第二个分支很快通过解析路径转换为Resource以后也会进到这里
@Override
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int counter = 0;
    // 注意这里是个for循环，也就是每个文件是一个resource
    for (Resource resource : resources) {
        // 继续往下看
        counter += loadBeanDefinitions(resource);
    }
    // 最后返回counter，表示总共加载了多少的BeanDefinition
    return counter;
}

// XmlBeanDefinitionReader 303
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(new EncodedResource(resource));
}

// XmlBeanDefinitionReader 314
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isInfoEnabled()) {
        logger.info("Loading XML bean definitions from " + encodedResource.getResource());
    }
    // 用一个ThreadLocal来存放配置文件资源
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<EncodedResource>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // 核心部分是这里，往下面看
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        currentResources.remove(encodedResource);
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}

// 还在这个文件中，第388行
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {
    try {
        // 这里就不看了，将xml文件转换为Document对象
        Document doc = doLoadDocument(inputSource, resource);
        // 继续
        return registerBeanDefinitions(doc, resource);
    }
    catch (...
           }
           // 还在这个文件中，第505行
           // 返回值：返回从当前配置文件加载了多少数量的Bean
           public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
               BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
               int countBefore = getRegistry().getBeanDefinitionCount();
               // 这里
               documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
               return getRegistry().getBeanDefinitionCount() - countBefore;
           }
           // DefaultBeanDefinitionDocumentReader 90
           @Override
           public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
               this.readerContext = readerContext;
               logger.debug("Loading bean definitions");
               Element root = doc.getDocumentElement();
               // 从xml根节点开始解析文件
               doRegisterBeanDefinitions(root);
           }         
```

经过漫长的链路，**一个配置文件终于转换为一颗 DOM 树**了，注意，这里指的是**其中一个配置文件**，不是所有的，可以看到上面有个 for 循环的。下面开始**从根节点**开始解析：

**doRegisterBeanDefinitions**

```java
// DefaultBeanDefinitionDocumentReader 116
protected void doRegisterBeanDefinitions(Element root) {
    // 看名字就知道，BeanDefinitionParserDelegate必定是一个重要的类，它负责解析Bean定义，
    // 这里为什么要定义一个 parent? 看到后面就知道了，是递归问题，
    // 因为<beans/> 内部是可以定义<beans/> 的，所以这个方法的root其实不一定就是xml的根节点，也可以是嵌套在里面的<beans/>节点，从源码分析的角度把它当做根节点就好了
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);

    if (this.delegate.isDefaultNamespace(root)) {
        // 这块说的是根节点 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的，
        // 如果当前环境配置的profile不包含此profile，那就直接 return 了，不对此<beans/>解析
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                }
                return;
            }
        }
    }

    preProcessXml(root); // 钩子方法
    // 往下看
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root); // 钩子方法

    this.delegate = parent;
}
```

preProcessXml(root) 和 postProcessXml(root) 是给**子类用的钩子方法**，鉴于没有被使用到，直接跳过。

接下来，看核心解析方法 **parseBeanDefinitions**(root, this.delegate) :

```java
// default namespace涉及到的就四个标签<import/>、<alias/>、<bean/>和<beans/>，
// 其他的属于custom的
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    // 解析default namespace下面的几个元素
                    parseDefaultElement(ele, delegate);
                }
                else {
                    // 解析其他namespace的元素
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```

从上面的代码可以看到，对于每个配置来说，**分别进入**到 **parseDefaultElement**(ele, delegate); 和 delegate.parseCustomElement(ele); 这两个分支了。

parseDefaultElement(ele, delegate) 代表解析的节点是  **\<import />、\<alias />、\<bean />、\<beans />** 这几个。

这里的四个标签之所以是 **default** 的，是因为它们是处于这个 namespace 下定义的：

```java
http://www.springframework.org/schema/beans
```

不熟悉 namespace 的看下面贴出来的 xml，这里的第二行 **xmlns** 就是。

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns="http://www.springframework.org/schema/beans"
 xsi:schemaLocation="
 http://www.springframework.org/schema/beans
 http://www.springframework.org/schema/beans/spring-beans.xsd"
 default-autowire="byName">
```

而对于其他的标签，将进入到 delegate.parseCustomElement(element) 这个分支。如经常会使用到的 **\<mvc />、\<task />、\<context />、\<aop />** 等。

这些属于扩展，如果需要使用上面这些 "非 default" 标签，那么上面的 xml 头部的地方也要引入相应的 namespace 和 .xsd 文件的路径，如下所示。同时代码中需要**提供相应的 parser 来解析**，如 MvcNamespaceHandler、TaskNamespaceHandler、ContextNamespaceHandler、AopNamespaceHandler 等。

假如读者想分析 **<context:property-placeholder location="classpath:xx.properties" />** 的**实现原理**，就应该到 **ContextNamespaceHandler** 中找答案。

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns="http://www.springframework.org/schema/beans"
xmlns:context="http://www.springframework.org/schema/context"
xmlns:mvc="http://www.springframework.org/schema/mvc"
xsi:schemaLocation="
     http://www.springframework.org/schema/beans 
     http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/context
     http://www.springframework.org/schema/context/spring-context.xsd
     http://www.springframework.org/schema/mvc   
     http://www.springframework.org/schema/mvc/spring-mvc.xsd  
 "
default-autowire="byName">
```

同理，要是碰到 **\<dubbo />**  这种标签，那么就应该搜一搜是不是有 DubboNamespaceHandler 这个处理类。

回来看看处理 **default 标签**的方法：

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        // 处理<import />标签
        importBeanDefinitionResource(ele);
    }
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        // 处理<alias />标签定义
        // <alias name="fromName" alias="toName"/>
        processAliasRegistration(ele);
    }
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        // 处理<bean />标签定义，这也算是重点吧
        processBeanDefinition(ele, delegate);
    }
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // 如果碰到的是嵌套的<beans />标签，需要递归
        doRegisterBeanDefinitions(ele);
    }
}
```

挑**重点 \<bean/> 标签**出来说。

###### (6) processBeanDefinition解析bean标签

下面是 processBeanDefinition **解析** \<bean/> 标签：

// DefaultBeanDefinitionDocumentReader 298

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 将<bean />节点中的信息提取出来，然后封装到一个BeanDefinitionHolder中，细节往下看
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);

    // 下面的几行先不要看，跳过先，跳过先，跳过先，后面会继续说的

    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // Register the final decorated instance.
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" + bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

继续往下看怎么解析之前，先看下 \<bean/> 标签中**可以定义哪些属性**：

|           Property           |                             含义                             |
| :--------------------------: | :----------------------------------------------------------: |
|            class             |                         类的全限定名                         |
|           **name**           |           可指定 id、name(用逗号、分号、空格分隔)            |
|          **scope**           |                            作用域                            |
|    constructor arguments     |                         指定构造参数                         |
|        **properties**        |                         设置属性的值                         |
|     **autowiring mode**      |           no(默认值)、byName、byType、 constructor           |
| **lazy-initialization mode** | **是否懒加载**(如果被非懒加载的bean依赖了那么其实也就不能懒加载了) |
|  **initialization method**   |           bean **属性设置完成**后，会调用这个方法            |
|    **destruction method**    |                  bean **销毁后**的回调方法                   |

简单地说就是像下面这样子：

```xml
<bean id="exampleBean" name="name1, name2, name3" class="com.javadoop.ExampleBean"
      scope="singleton" lazy-init="true" init-method="init" destroy-method="cleanup">

    <!--可以用下面三种形式指定构造参数-->
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg index="0" value="7500000"/>

    <!--property的几种情况-->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>
```

除了上面举例出来的这些，还有 factory-bean、factory-method、\<lockup-method />、\<replaced-method />、\<meta />、\<qualifier /> 等。

再继续往里看**怎么解析 bean 元素**，是怎么转换到 **BeanDefinitionHolder** 的。

**// BeanDefinitionParserDelegate 428**

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}

public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
    String id = ele.getAttribute(ID_ATTRIBUTE);
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    List<String> aliases = new ArrayList<String>();

    // 将name属性的定义按照 “逗号、分号、空格” 切分，形成一个 别名列表数组，
    // 当然，如果你不定义name属性的话，就是空的了
    // 我在附录中简单介绍了一下id和name的配置，大家可以看一眼，有个20秒就可以了
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    String beanName = id;
    // 如果没有指定id, 那么用别名列表的第一个名字作为beanName
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
        if (logger.isDebugEnabled()) {
            logger.debug("No XML 'id' specified - using '" + beanName +
                         "' as bean name and " + aliases + " as aliases");
        }
    }

    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }

    // 根据 <bean ...>...</bean>中的配置创建BeanDefinition，然后把配置中的信息都设置到实例中,
    // 细节后面细说，先知道下面这行结束后，一个BeanDefinition实例就出来了。
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);

    // 到这里，整个<bean/> 标签就算解析结束了，一个BeanDefinition就形成了。
    if (beanDefinition != null) {
        // 如果都没有设置id和name，那么此时的beanName就会为null，进入下面这块代码产生
        // 如果读者不感兴趣的话，我觉得不需要关心这块代码，对本文源码分析来说，这些东西不重要
        if (!StringUtils.hasText(beanName)) {
            try {
                // 按照我们的思路，这里containingBean是null的
                if (containingBean != null) {
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                        beanDefinition, this.readerContext.getRegistry(), true);
                }
                else {
                    // 如果我们不定义id和name，那么我们引言里的那个例子：
                    // 1.beanName为：com.javadoop.example.MessageServiceImpl#0
                    // 2.beanClassName为：com.javadoop.example.MessageServiceImpl

                    beanName = this.readerContext.generateBeanName(beanDefinition);

                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                        beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                        !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        // 把beanClassName设置为Bean的别名
                        aliases.add(beanClassName);
                    }
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Neither XML 'id' nor 'name' specified - " +
                                 "using generated bean name [" + beanName + "]");
                }
            }
            catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        // 返回 BeanDefinitionHolder
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }

    return null;
}
```

然后，再看看怎么根据配置**创建 BeanDefinition 实例**的：

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
    Element ele, String beanName, BeanDefinition containingBean) {

    this.parseState.push(new BeanEntry(beanName));

    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }

    try {
        String parent = null;
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }
        // 创建 BeanDefinition，然后设置类信息而已，很简单，就不贴代码了
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);

        // 设置 BeanDefinition 的一堆属性，这些属性定义在 AbstractBeanDefinition 中
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

        /**
         * 下面的一堆是解析<bean>......</bean>内部的子元素，
         * 解析出来以后的信息都放到bd的属性中
         */
        // 解析 <meta />
        parseMetaElements(ele, bd);
        // 解析 <lookup-method />
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        // 解析 <replaced-method />
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
        // 解析 <constructor-arg />
        parseConstructorArgElements(ele, bd);
        // 解析 <property />
        parsePropertyElements(ele, bd);
        // 解析 <qualifier />
        parseQualifierElements(ele, bd);

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));

        return bd;
    }
    catch (ClassNotFoundException ex) {
        error("Bean class [" + className + "] not found", ele, ex);
    }
    catch (NoClassDefFoundError err) {
        error("Class that bean class [" + className + "] depends on not found", ele, err);
    }
    catch (Throwable ex) {
        error("Unexpected failure during bean definition parsing", ele, ex);
    }
    finally {
        this.parseState.pop();
    }

    return null;
}
```

到这里已经完成了根据 **\<bean/>  配置**创建了一个 **BeanDefinitionHolder** 实例。注意，**==是一个==**。

回到解析 \<bean/> 的**入口方法**:

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 将<bean/>节点转换为BeanDefinitionHolder，就是上面说的一堆
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 如果有自定义属性的话，进行相应的解析，先忽略
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 把这步叫做注册Bean吧
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                                     bdHolder.getBeanName() + "'", ele, ex);
        }
        // 注册完成后，发送事件，本文不展开说这个
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

这里已经**根据一个 \<bean /> 标签产生了一个** **BeanDefinitionHolder** 的实例，这个实例里面也就是一个 **BeanDefinition** 的实例和它的 **beanName、aliases 这三个信息**，注意，关注点始终在 BeanDefinition 上：

```java
public class BeanDefinitionHolder implements BeanMetadataElement {

    private final BeanDefinition beanDefinition;

    private final String beanName;

    private final String[] aliases;
    //...
}
```

然后准备**注册这个 BeanDefinition**，最后，把这个**注册事件发送**出去。下面说说注册 Bean。

###### (7) 注册Bean

**// BeanDefinitionReaderUtils 143**

```java
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

    String beanName = definitionHolder.getBeanName();
    // 注册这个Bean
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 如果还有别名的话，也要根据别名全部注册一遍，不然根据别名就会找不到Bean了
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            // alias -> beanName 保存它们的别名信息，这个很简单，用一个map保存一下就可以了，
            // 获取的时候，会先将alias转换为beanName，然后再查找
            registry.registerAlias(beanName, alias);
        }
    }
}
```

别名注册的放一边，毕竟它很简单，看看**怎么注册 Bean**。

**// DefaultListableBeanFactory 793**

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(...);
        }
    }

    // old? 还记得 “允许 bean 覆盖” 这个配置吗？allowBeanDefinitionOverriding
    BeanDefinition oldBeanDefinition;

    // 之后会看到，所有的Bean注册后会放入这个beanDefinitionMap中
    oldBeanDefinition = this.beanDefinitionMap.get(beanName);

    // 处理重复名称的Bean定义的情况
    if (oldBeanDefinition != null) {
        if (!isAllowBeanDefinitionOverriding()) {
            // 如果不允许覆盖的话，抛异常
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription());
            // ...  
        }
        else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
            // log...用框架定义的Bean覆盖用户自定义的Bean 
        }
        else if (!beanDefinition.equals(oldBeanDefinition)) {
            // log...用新的Bean覆盖旧的Bean
        }
        else {
            // log...用同等的Bean覆盖旧的Bean，这里指的是equals方法返回true的Bean
        }
        // 覆盖
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        // 判断是否已经有其他的Bean开始初始化了.
        // 注意，"注册Bean" 这个动作结束，Bean依然还没有初始化，我们后面会有大篇幅说初始化过程，
        // 在Spring容器启动的最后，会 预初始化 所有的 singleton beans
        if (hasBeanCreationStarted()) {
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        else {
            // 最正常的应该是进到这个分支。

            // 将 BeanDefinition 放到这个 map 中，这个 map 保存了所有的 BeanDefinition
            this.beanDefinitionMap.put(beanName, beanDefinition);
            // 这是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
            this.beanDefinitionNames.add(beanName);
            // 这是个 LinkedHashSet，代表的是手动注册的 singleton bean，
            // 注意这里是 remove 方法，到这里的 Bean 当然不是手动注册的
            // 手动指的是通过调用以下方法注册的 bean ：
            //     registerSingleton(String beanName, Object singletonObject)
            // 这不是重点，解释只是为了不让大家疑惑。Spring 会在后面"手动"注册一些 Bean，
            // 如 "environment"、"systemProperties" 等 bean，我们自己也可以在运行时注册 Bean 到容器中的
            this.manualSingletonNames.remove(beanName);
        }
        // 这个不重要，在预初始化的时候会用到，不必管它。
        this.frozenBeanDefinitionNames = null;
    }

    if (oldBeanDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

总结一下，到这里已经初始化了 Bean 容器，**\<bean/>   配置也相应的转换为了一个个 BeanDefinition**，然后**注册了各个 BeanDefinition 到注册中心，并且发送了注册事件**。

到这里是一个**分水岭**，前面的内容都还算比较简单，不过应该也比较繁琐，大家要清楚地知道前面都做了哪些事情。

##### 3. Bean容器实例化完成后

再次**回到 refresh() 方法**，前面说完了 obtainFreshBeanFactory() 方法。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   // 来个锁，不然refresh()还没结束又来个启动或销毁容器的操作，那就炸了
   synchronized (this.startupShutdownMonitor) {
       
      // 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
      prepareRefresh();

      // 这步比较关键，这步完成后，配置文件就会解析成一个个Bean定义，注册到BeanFactory中，
      // 当然这里说的Bean还没有初始化，只是配置信息都提取出来了，注册也只是将这些信息都保存
      // 到了注册中心中(说到底核心是一个beanName -> beanDefinition的map)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 设置BeanFactory的类加载器，添加几个BeanPostProcessor，手动注册几个特殊的bean
      prepareBeanFactory(beanFactory);

      try {
         // 需要知道BeanFactoryPostProcessor这个知识点，Bean如果实现了此接口，
         // 那么在容器初始化以后，Spring会负责调用里面的postProcessBeanFactory方法。
         // 这里是提供给子类的扩展点，到这里的时候，所有的Bean都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的BeanFactoryPostProcessor的实现类或做点什么事
         postProcessBeanFactory(beanFactory);
         // 调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory(factory)方法
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册BeanPostProcessor的实现类，注意看和BeanFactoryPostProcessor的区别
         // 此接口两个方法: postProcessBeforeInitialization和postProcessAfterInitialization
         // 两个方法分别在Bean初始化之前和初始化之后得到执行。注意，到这里Bean还没初始化
         registerBeanPostProcessors(beanFactory);

         // 初始化当前ApplicationContext的MessageSource，国际化就不展开了
         initMessageSource();

         // 初始化当前ApplicationContext的事件广播器，这里也不展开了
         initApplicationEventMulticaster();

         // 从方法名就可以知道，典型的模板方法(钩子方法)，
         // 具体的子类可以在这里初始化一些特殊的Bean（在初始化singleton beans之前）
         onRefresh();

         // 注册事件监听器，监听器需要实现ApplicationListener接口。这也不是我们的重点，过
         registerListeners();

         // 重点重点重点：初始化所有的Singleton Beans（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);

         // 最后广播事件，ApplicationContext初始化完成
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }
         // 销毁已经初始化的singleton的Beans，以免有些bean会一直占用资源
         destroyBeans();

         // 重置active标志位
         cancelRefresh(ex);

         // 把异常往外抛
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

##### 4. 准备Bean容器: prepareBeanFactory

Spring 把在 xml 配置的 **bean 都注册**以后，会**"手动"注册一些特殊的 bean**。这里简单介绍下 **prepareBeanFactory**(factory) 方法：

```java
/**
 * Configure the factory's standard context characteristics,
 * such as the context's ClassLoader and post-processors.
 * @param beanFactory the BeanFactory to configure
 */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // 设置BeanFactory的类加载器，我们知道BeanFactory需要加载类，也就需要类加载器，
   // 这里设置为加载当前ApplicationContext类的类加载器
   beanFactory.setBeanClassLoader(getClassLoader());

   // 设置BeanExpressionResolver
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // 添加一个BeanPostProcessor，这个processor比较简单：
   // 实现了Aware接口的beans在初始化的时候，这个processor负责回调，
   // 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
   // 注意：它不仅仅回调 ApplicationContextAware，
   //   还会负责回调 EnvironmentAware、ResourceLoaderAware 等，看下源码就清楚了
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

   // 下面几行的意思就是，如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，
   // Spring 会通过其他方式来处理这些依赖。
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   /**
    * 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
    * 之前我们说过，"当前 ApplicationContext 持有一个 BeanFactory"，这里解释了第一行。
    * ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
    * 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext
    * 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean
    */
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // 这个 BeanPostProcessor 也很简单，在 bean 实例化后，如果是 ApplicationListener 的子类，
   // 那么将其添加到 listener 列表中，可以理解成：注册 事件监听器
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // 这里涉及到特殊的 bean，名为：loadTimeWeaver，这不是我们的重点，忽略它
   // tips: ltw 是 AspectJ 的概念，指的是在运行期进行织入，这个和 Spring AOP 不一样，
   //    感兴趣的读者请参考我写的关于 AspectJ 的另一篇文章 https://www.javadoop.com/post/aspectj
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   /**
    * 从下面几行代码我们可以知道，Spring 往往很 "智能" 就是因为它会帮我们默认注册一些有用的 bean，
    * 我们也可以选择覆盖
    */

   // 如果没有定义 "environment" 这个 bean，那么 Spring 会 "手动" 注册一个
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   // 如果没有定义 "systemProperties" 这个 bean，那么 Spring 会 "手动" 注册一个
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   // 如果没有定义 "systemEnvironment" 这个 bean，那么 Spring 会 "手动" 注册一个
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```

这里 Spring 对一些**特殊的 bean 进行了处理**，读者如果暂时还不能消化它们也没有关系，慢慢往下看。

##### 5. 初始化所有的Singleton Beans

**重点**是 **finishBeanFactoryInitialization(beanFactory);** 这个巨头了，这里会负责**初始化所有的 Singleton beans。**

注意后面的描述中都会使用**初始化**或**预初始化**来代表这个阶段，Spring 会在这个阶段完成所有的 singleton beans 的实例化。

总结一下，到目前为止 **BeanFactory 已经创建完成**，并且**所有的实现了 BeanFactoryPostProcessor** 接口的 Bean 都已经**初始化**并且其中的 **postProcessBeanFactory**(factory) 方法已经得到**回调执行**了。而且 Spring 已经“手动”注册了一些**特殊的 Bean**，如 `environment`、`systemProperties` 等。

剩下的就是**初始化 singleton beans** 了，它们是**单例**的，如果**没有设置懒加载**，那么 Spring 会在接下来**初始化所有**的 singleton beans。

**// AbstractApplicationContext.java 834**

```java
// 初始化剩余的 singleton beans
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {

    // 首先初始化名字为conversionService的Bean。
    // 看代码这里没有初始化Bean啊！
    // 注意了，初始化的动作包装在beanFactory.getBean(...) 中，这里先不说细节，先往下看吧
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
            @Override
            public String resolveStringValue(String strVal) {
                return getEnvironment().resolvePlaceholders(strVal);
            }
        });
    }

    // 先初始化LoadTimeWeaverAware类型的Bean
    // 之前也说过，这是AspectJ相关的内容，放心跳过
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // 没什么别的目的，因为到这一步的时候，Spring 已经开始预初始化 singleton beans 了，
    // 肯定不希望这个时候还出现 bean 定义解析、加载、注册。
    beanFactory.freezeConfiguration();

    // 开始初始化
    beanFactory.preInstantiateSingletons();
}
```

从上面最后一行往里看，就又回到 **DefaultListableBeanFactory** 这个类了，这个类大家应该都不陌生了吧。

###### (1) preInstantiateSingletons

// DefaultListableBeanFactory 728

```java
@Override
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Pre-instantiating singletons in " + this);
    }
    // this.beanDefinitionNames 保存了所有的 beanNames
    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

    // 下面这个循环，触发所有的非懒加载的singleton beans的初始化操作
    for (String beanName : beanNames) {

        // 合并父Bean中的配置，注意 <bean id="" class="" parent="" /> 中的 parent，用的不多吧，
        // 考虑到这可能会影响大家的理解，我在附录中解释了一下 "Bean 继承"，不了解的请到附录中看一下
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);

        // 非抽象、非懒加载的singletons。如果配置了 'abstract = true'，那是不需要初始化的
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 处理FactoryBean(读者如果不熟悉 FactoryBean，请移步附录区了解)
            if (isFactoryBean(beanName)) {
                // FactoryBean的话，在beanName前面加上‘&’符号。再调用getBean，getBean方法别急
                final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                // 判断当前FactoryBean是否是SmartFactoryBean的实现，此处忽略，直接跳过
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                        @Override
                        public Boolean run() {
                            return ((SmartFactoryBean<?>) factory).isEagerInit();
                        }
                    }, getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                                   ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {

                    getBean(beanName);
                }
            }
            else {
                // 对于普通的 Bean，只要调用 getBean(beanName) 这个方法就可以进行初始化了
                getBean(beanName);
            }
        }
    }

    // 到这里说明所有的非懒加载的 singleton beans 已经完成了初始化
    // 如果我们定义的 bean 是实现了 SmartInitializingSingleton 接口的，那么在这里得到回调，忽略
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

接下来就进入到 **getBean**(beanName) 方法了，这个方法经常用来从 BeanFactory 中**获取一个 Bean**，而初始化的过程也封装到了这个方法里。

###### (2) getBean

在继续前进之前，读者应该具备 **FactoryBean** 的知识。

**// AbstractBeanFactory 196**

```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

// 在剖析初始化Bean的过程，但是 getBean 方法我们经常是用来从容器中获取 Bean 用的，注意切换思路，
// 已经初始化过了就从容器中直接返回，否则就先初始化再返回
@SuppressWarnings("unchecked")
protected <T> T doGetBean(
    final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
    throws BeansException {
    // 获取一个 “正统的” beanName，处理两种情况，一个是前面说的 FactoryBean(前面带 ‘&’)，
    // 一个是别名问题，因为这个方法是 getBean，获取 Bean 用的，你要是传一个别名进来，是完全可以的
    final String beanName = transformedBeanName(name);

    // 注意跟着这个，这个是返回值
    Object bean; 

    // 检查下是不是已经创建过了
    Object sharedInstance = getSingleton(beanName);

    // 这里说下 args 呗，虽然看上去一点不重要。前面我们一路进来的时候都是 getBean(beanName)，
    // 所以args传参其实是null的，但是如果args不为空的时候，那么意味着调用方不是希望获取Bean，而是创建 Bean
    if (sharedInstance != null && args == null) {
        if (logger.isDebugEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.debug("...");
            }
            else {
                logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // 下面这个方法：如果是普通 Bean 的话，直接返回 sharedInstance，
        // 如果是 FactoryBean 的话，返回它创建的那个实例对象
        // (FactoryBean 知识，读者若不清楚请移步附录)
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        if (isPrototypeCurrentlyInCreation(beanName)) {
            // 创建过了此 beanName 的 prototype 类型的 bean，那么抛异常，
            // 往往是因为陷入了循环引用
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 检查一下这个 BeanDefinition 在容器中是否存在
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // 如果当前容器不存在这个 BeanDefinition，试试父容器中有没有
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // 返回父容器的查询结果
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }

        if (!typeCheckOnly) {
            // typeCheckOnly 为 false，将当前 beanName 放入一个 alreadyCreated 的 Set 集合中。
            markBeanAsCreated(beanName);
        }

      /*
       * 稍稍总结一下：
       * 到这里的话，要准备创建 Bean 了，对于 singleton 的 Bean 来说，容器中还没创建过此 Bean；
       * 对于 prototype 的 Bean 来说，本来就是要创建一个新的 Bean。
       */
        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 先初始化依赖的所有 Bean，这个很好理解。
            // 注意，这里的依赖指的是 depends-on 中定义的依赖
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 检查是不是有循环依赖，这里的循环依赖和我们前面说的循环依赖又不一样，这里肯定是不允许出现的，不然要乱套了，读者想一下就知道了
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 注册一下依赖关系
                    registerDependentBean(dep, beanName);
                    // 先初始化被依赖项
                    getBean(dep);
                }
            }

            // 如果是 singleton scope 的，创建 singleton 的实例
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            // 执行创建 Bean，详情后面再说
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            // 如果是 prototype scope 的，创建 prototype 的实例
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    // 执行创建 Bean
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            // 如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            beforePrototypeCreation(beanName);
                            try {
                                // 执行创建 Bean
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                                                    "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                                    "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                                    ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // 最后，检查一下类型对不对，不对的话就抛异常，对的话就返回了
    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        try {
            return getTypeConverter().convertIfNecessary(bean, requiredType);
        }
        catch (TypeMismatchException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to convert bean '" + name + "' to required type '" +
                             ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

大家应该也猜到了，接下来当然是分析 **createBean** 方法：

```java
protected abstract Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException;
```

第三个参数 args 数组代表**创建实例需要的参数**，不就是给构造方法用的参数，或者是工厂 Bean 的参数嘛，不过要注意，在初始化阶段，args 是 null。

这回要到一个新的类了 **AbstractAutowireCapableBeanFactory**，看类名，AutowireCapable？类名是不是也说明了点问题了。

主要是为了以下场景，采用 @**Autowired** 注解注入属性值：

```java
public class MessageServiceImpl implements MessageService {
    @Autowired
    private UserService userService;

    public String getMessage() {
        return userService.getMessage();
    }
}
<bean id="messageService" class="com.javadoop.example.MessageServiceImpl" />
```

以上这种属于混用了 **xml 和 注解** 两种方式的配置方式，Spring 会处理这种情况。

// AbstractAutowireCapableBeanFactory 447

```java
/**
 * Central method of this class: creates a bean instance,
 * populates the bean instance, applies post-processors, etc.
 * @see #doCreateBean
 */
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
   if (logger.isDebugEnabled()) {
      logger.debug("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // 确保 BeanDefinition 中的 Class 被加载
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // 准备方法覆写，这里又涉及到一个概念：MethodOverrides，它来自于 bean 定义中的 <lookup-method /> 
   // 和 <replaced-method />，如果读者感兴趣，回到 bean 解析的地方看看对这两个标签的解析。
   // 我在附录中也对这两个标签的相关知识点进行了介绍，读者可以移步去看看
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // 让 InstantiationAwareBeanPostProcessor 在这一步有机会返回代理，
      // 在 《Spring AOP 源码分析》那篇文章中有解释，这里先跳过
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean; 
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }
   // 重头戏，创建 bean
   Object beanInstance = doCreateBean(beanName, mbdToUse, args);
   if (logger.isDebugEnabled()) {
      logger.debug("Finished creating instance of bean '" + beanName + "'");
   }
   return beanInstance;
}
```

###### (3) 创建Bean

继续往里看 **doCreateBean** 这个方法：

```java
/**
 * Actually create the specified bean. Pre-creation processing has already happened
 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
 * <p>Differentiates between default bean instantiation, use of a
 * factory method, and autowiring a constructor.
 * @param beanName the name of the bean
 * @param mbd the merged bean definition for the bean
 * @param args explicit arguments to use for constructor or factory method invocation
 * @return a new instance of the bean
 * @throws BeanCreationException if the bean could not be created
 * @see #instantiateBean
 * @see #instantiateUsingFactoryMethod
 * @see #autowireConstructor
 */
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
    throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 说明不是 FactoryBean，这里实例化 Bean，这里非常关键，细节之后再说
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 这个就是 Bean 里面的 我们定义的类 的实例，很多地方我直接描述成 "bean 实例"
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    // 类型
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;

    // 建议跳过吧，涉及接口：MergedBeanDefinitionPostProcessor
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // MergedBeanDefinitionPostProcessor，这个我真不展开说了，直接跳过吧，很少用的
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // 下面这块代码是为了解决循环依赖的问题，以后有时间，我再对循环依赖这个问题进行解析吧
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                         "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 这一步也是非常关键的，这一步负责属性装配，因为前面的实例只是实例化了，并没有设值，这里就是设值
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 还记得 init-method 吗？还有 InitializingBean 接口？还有 BeanPostProcessor 接口？
            // 这里就是处理 bean 初始化完成后的各种回调
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        // 
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                                                               "Bean with name '" + beanName + "' has been injected into other beans [" +
                                                               StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                                               "] in its raw version as part of a circular reference, but has eventually been " +
                                                               "wrapped. This means that said other beans do not use the final version of the " +
                                                               "bean. This is often the result of over-eager type matching - consider using " +
                                                               "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

到这里已经分析完了 doCreateBean 方法，总的来说，已经**说完了整个初始化**流程。

接下来挑 **doCreateBean** 中的三个**细节**出来说说。一个是**创建 Bean 实例的 createBeanInstance 方法，一个是依赖注入的 populateBean 方法，还有就是回调方法 initializeBean。** 

注意了，接下来的这三个方法要认真说那也是极其复杂的，很多地方就点到为止了。

###### (4) 创建Bean实例

先看看 **createBeanInstance** 方法。需要说明的是，这个方法如果每个分支都分析下去，必然也是极其复杂冗长的就挑重点说。此方法的目的就是实例化指定的类。

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 确保已经加载了此 class
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    // 校验一下这个类的访问权限
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                        "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    if (mbd.getFactoryMethodName() != null)  {
        // 采用工厂方法实例化，不熟悉这个概念的读者请看附录，注意，不是 FactoryBean
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // 如果不是第一次创建，比如第二次创建 prototype bean。
    // 这种情况下，我们可以从第一次创建知道，采用无参构造函数，还是构造函数依赖注入 来完成实例化
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            // 构造函数依赖注入
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            // 无参构造函数
            return instantiateBean(beanName, mbd);
        }
    }

    // 判断是否采用有参构造函数
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        // 构造函数依赖注入
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // 调用无参构造函数
    return instantiateBean(beanName, mbd);
}
```

挑个简单的**无参构造函数**构造实例来看看：

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        if (System.getSecurityManager() != null) {
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {

                    return getInstantiationStrategy().instantiate(mbd, beanName, parent);
                }
            }, getAccessControlContext());
        }
        else {
            // 实例化
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        // 包装一下，返回
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
}
```

可以看到，关键的地方在于：

```java
beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
```

这里会进行**实际的实例化过程**，进去看看:

// SimpleInstantiationStrategy 59

```java
@Override
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {

    // 如果不存在方法覆写，那就使用 java 反射进行实例化，否则使用 CGLIB,
    // 方法覆写 请参见附录"方法注入"中对 lookup-method 和 replaced-method 的介绍
    if (bd.getMethodOverrides().isEmpty()) {
        Constructor<?> constructorToUse;
        synchronized (bd.constructorArgumentLock) {
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                final Class<?> clazz = bd.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    if (System.getSecurityManager() != null) {
                        constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
                            @Override
                            public Constructor<?> run() throws Exception {
                                return clazz.getDeclaredConstructor((Class[]) null);
                            }
                        });
                    }
                    else {
                        constructorToUse = clazz.getDeclaredConstructor((Class[]) null);
                    }
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                }
                catch (Throwable ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }
        // 利用构造方法进行实例化
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // 存在方法覆写，利用 CGLIB 来完成实例化，需要依赖于 CGLIB 生成子类，这里就不展开了。
        // tips: 因为如果不使用 CGLIB 的话，存在 override 的情况 JDK 并没有提供相应的实例化支持
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

到这里实例化就算完成了。开始说怎么**进行属性注入**。

###### (5) bean属性注入

看完了 createBeanInstance(...) 方法，来看看 **populateBean**(...) 方法，该方法**负责进行属性设值，处理依赖**。

// AbstractAutowireCapableBeanFactory 1203

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // bean 实例的所有属性都在这里了
    PropertyValues pvs = mbd.getPropertyValues();

    if (bw == null) {
        if (!pvs.isEmpty()) {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // 到这步的时候，bean实例化完成（通过工厂方法或构造方法），但是还没开始属性设值，
    // InstantiationAwareBeanPostProcessor 的实现类可以在这里对 bean 进行状态修改，
    // 我也没找到有实际的使用，所以我们暂且忽略这块吧
    boolean continueWithPropertyPopulation = true;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 如果返回false，代表不需要进行后续的属性设值，也不需要再经过其他的 BeanPostProcessor 的处理
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }

    if (!continueWithPropertyPopulation) {
        return;
    }

    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // 通过名字找到所有属性值，如果是bean依赖，先初始化依赖的 bean。记录依赖关系
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // 通过类型装配。复杂一些
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

    if (hasInstAwareBpps || needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 这里有个非常有用的 BeanPostProcessor 进到这里: AutowiredAnnotationBeanPostProcessor
                    // 对采用 @Autowired、@Value 注解的依赖进行设值，这里的内容也是非常丰富的，不过本文不会展开说了，感兴趣的读者请自行研究
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }
    // 设置 bean 实例的属性值
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

###### (6) initializeBean

**属性注入完成后，这一步其实就是处理各种回调**了，这块代码比较简单。

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                invokeAwareMethods(beanName, bean);
                return null;
            }
        }, getAccessControlContext());
    }
    else {
        // 如果bean实现了BeanNameAware、BeanClassLoaderAware或BeanFactoryAware接口，回调
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // BeanPostProcessor的postProcessBeforeInitialization回调
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 处理bean中定义的init-method，
        // 或者如果bean实现了InitializingBean接口，调用afterPropertiesSet()方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }

    if (mbd == null || !mbd.isSynthetic()) {
        // BeanPostProcessor 的 postProcessAfterInitialization 回调
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

BeanPostProcessor 的两个回调都发生在这边，只不过中间处理了 **init-method**。



#### 参考资料

- IOC源码分析（牛皮）：https://javadoop.com/post/spring-ioc#toc_10