[TOC]

### Spring IOC



在 Spring 中，那些组成应用程序的主体及由 Spring IOC 容器所管理的对象，被称之为 **bean**。简单地讲，bean 就是由 IOC 容器初始化、装配及管理的对象，除此之外，bean 就与应用程序中的其他对象没有什么区别了。而 bean 的定义以及 bean 相互间的依赖关系将通过配置元数据来描述。



#### Bean作用域

Scope 用来声明容器中的对象所应该处的限定场景或者说该对象的存活时间，即容器在对象进入其相应的 scope之前，生成并装配这些对象，在该对象不再处于这些 scope 的限定之后，容器通常会销毁 Spring 的 IoC 容器这些对象。

**Spring 的单例是基于 BeanFactory 也就是 Spring 容器的，单例 Bean 在此容器内只有一个，Java 的单例是基于 JVM，每个 JVM 内只有一个实例**。

在大多数情况下单例 bean 是很理想的方案，不过有时候使用的类是易变的，它们**会保持一些状态**，因此重用是不安全的。在这种情况下，将 class 声明为单例的就不是那么明智了。因为对象会被污染，所以 Spring 定义了多种作用域的bean。

##### 1. 作用域分类

|     Scope      |                         Description                          |
| :------------: | :----------------------------------------------------------: |
| **singleton**  | （**默认的**）在每个 Spring IoC 容器中，一个 bean 定义对应只会有唯一一个 bean 实例 |
| **prototype**  |              一个 bean 定义可以有多个 bean 实例              |
|  **request**   | 一个 bean 定义对应于单个 **HTTP 请求**的生命周期，仅适用于 **WebApplicationContext** 环境。 |
|  **session**   |    一个 bean 定义对应于单个 **HTTP Session** 的生命周期。    |
| global session |           仅仅在基于 portlet 的 web 应用中才有意义           |

五种作用域中，**request、session** 和 **global session** 三种作用域仅在基于 web 的应用中使用，只能用在基于 web 的 Spring ApplicationContext 环境。

##### 2. singleton

在 Spring 的 IoC 容器中只存在**一个实例**，所有对该对象的引用将**共享**这个实例。该实例从容器启动，并因为第一次被请求而初始化之后，将一直存活到容器退出，也就是说，它与 IoC 容器“几乎”拥有相同的“寿命”。标记为 singleton 的 bean是由容器来保证这种类型的 bean 在**同一个容器中**只存在一个共享实例；而 Singleton 模式则是保证在同一个 **Classloader** **中**只存在一个这种类型的实例。

在不指定 @Scope 的情况下，**所有**的 bean 都是**单实例**的 bean, 而且是**==饿汉加载==**，也就是容器启动就进行实例创建。

如果对 singleton 的 bean 指定为 **@Lazy 懒加载**，那么容器启动的时候不创建对象，而在**第一次使用**的时候才会创建该对象。

````java
@Bean
@Lazy
public Person person() {
    return new Person();
}
````

##### 3. prototype

当一个 bean 的作用域为 prototype 时，表示一个 bean **定义**对应**多个对象实例**。 prototype 作用域的 bean 会导致在每次对该 bean 请求（将其**注入**到另一个 bean 中，或者以程序的方式调用容器的 getBean() 方法）时都会**创建一个新**的 bean 实例。prototype 是原型类型，在**创建容器的时候并没有实例化**，而是当获取 bean 的时候才会去创建一个对象，而且每次获取到的对象是一个**新的对象**。

虽然这种类型的对象的**实例化以及属性设置**等工作都是由容器负责的，但是只要准备完毕，并且对象实例返回给请求方之后，**容器就不再拥有当前返回对象的引用**，请求方需要**自己负责**当前返回对象的后继生命周期的管理工作，包括该对象的销毁。

根据经验，对**==有状态的== bean 应该使用 prototype 作用域**（比如保存每个顾客信息的对象），而对无状态的 bean 则应该使用 singleton 作用域。  在 XML 中将 bean 定义成 prototype ，可以这样配置：

```java
<bean id="account" class="com.foo.DefaultAccount" scope="prototype"/>  
<bean id="account" class="com.foo.DefaultAccount" singleton="false"/> 
```

也可通过 **@Scope 注解**进行配置。

singleton 与 prototype 用法最大区别：**有无状态**。

##### 4. request

每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。**request 只适用于 Web 程序，每一次 HTTP 请求都会产生一个新的 bean，同时该 bean 仅在当前 HTTP request 内有效，当请求结束后，该对象的生命周期即告结束。** 在 XML 中将 bean 定义成 request ，可以这样配置：

```java
<bean id="loginAction" class=cn.csdn.LoginAction" scope="request"/>
```

##### 5. session

**会话**：用户打开浏览器会话开始，直到关闭浏览器会话才会结束。一次会话期间只会创建一个 session 对象。服务器会为每个会话创建一个 session 对象，所以 session 中的数据可供当前会话中所有 servlet 共享。

每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。session 只适用于 Web 程序，session 作用域表示该针对每一次 HTTP 请求都会产生一个新的 bean，同时该 bean 仅在**当前 HTTP session 内有效**。与 request 作用域一样，可以根据需要放心的更改所创建实例的内部状态，而别的 HTTP session 中根据 userPreferences 创建的实例，将不会看到这些特定于某个 HTTP session 的状态变化。当 HTTP session 最终被废弃的时候，在该 HTTP session 作用域内的 bean 也会被废弃掉。

```xml
<bean id="userPreferences" class="com.foo.UserPreferences" scope="session"/>
```

##### 6. globalSession

global session 作用域类似于标准的 HTTP session 作用域，不过仅仅在基于 portlet 的 web 应用中才有意义。Portlet 规范定义了全局 Session 的概念，它被所有构成某个 portlet web 应用的各种不同的 portlet 所共享。在 global session  作用域中定义的 bean 被限定于全局 portlet Session 的生命周期范围内。

```xml
<bean id="user" class="com.foo.Preferences "scope="globalSession"/>
```

##### 7. @Scope注解

如下所示为一个POJO类使用 @Scope 注解。使用 ConfigurableBeanFactory 只能提供单例 (SCOPE_SINGLETON)和原型 (SCOPE_PROTOTYPE)。在 MVC 中，还可以使用 WebApplicationContext 去提供 MVC 下特有的**作用域**形式，如请求（SCOPE_REQUEST），会话（SCOPE_SESSION）。

```java
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Profile;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)  	// 设置bean作用域
// @Scope(WebApplicationContext.SCOPE_REQUSET)  	// 设置bean作用域(For MVC)
public class TestBean {
}
```



#### Bean的装配

Spring 为实现 Bean 的信息定义，提供了**基于注解、基于配置类、基于 XML、基于 Groovy** 这几种 bean 装配方式，各种配置方式可混合使用。基于 XML 与 Groovy 配置这里就不涉及了。 

##### 1. 基于配置类的装配

通过**配置类**对一个 bean 进行装配。

- @**Configuration** 说明为一个**配置类**。
- @**ComponentScan** 标明采用何种策略去**扫描装配** Bean，默认扫描配置类当前所在的包及其子包。可以自定义扫描路径。
- 将 **@Bean** 标注方法返回的 bean 装配到 IOC 容器中。

看一个关于数据源的 Bean 装配的实例。

```java
@Configuration
@ComponentScan(basePackages = "com.springboot.chapter3.*") 
@ImportResource(value = {"classpath:spring-other.xml"})
public class AppConfig {
    
	@Bean(name = "dataSource", destroyMethod = "close")
	public DataSource getDevDataSource() {
		Properties props = new Properties();
		props.setProperty("driver", "com.mysql.jdbc.Driver");
		props.setProperty("url", "jdbc:mysql://localhost:3306/dev_spring_boot");
		props.setProperty("username", "root");
		props.setProperty("password", "123456");
		DataSource dataSource = null;
		try {
			dataSource = BasicDataSourceFactory.createDataSource(props);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return dataSource;
	}
}
```

##### 2. 基于注解的装配

定义一个 POJO 类，直接在此类里面**注入属性**。

- @**Component** 标明这个类被装配到容器中。
- @**Value** 字段中注入**属性值**。

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

// 指定user为bean名称 如果不指定则把类名首字母小写当做类名称
@Component("user")      
public class User {

    // @Value注入属性值
	@Value("1")         
	private Long id;
	@Value("user_name_1")
	private String userName;
	@Value("note_1")
	private String note;

	// Getters and Setters
}
```

##### 3. 装配方式对比

几种配置方式对比：

![1573547101474](assets/1573547101474.png)

##### 4. @Conditional条件装配Bean

**满足一定条件才装配 Bean**，否则不装配，比如数据库配置信息不全就不装配。@Conditional 注解用于条件装配 Bean，需要配合 Condition 接口使用。

```java
@Configuration
@ComponentScan(basePackages = "com.springboot.chapter3.*")
@ImportResource(value = {"classpath:spring-other.xml"})
public class AppConfig {
	
	@Bean(name = "dataSource", destroyMethod = "close")
    // 传入的类需实现Condition接口
	@Conditional(DatabaseConditional.class)      
	public DataSource getDataSource(            
        // 使用@Value去取配置文件中的值并注入
			@Value("${database.driverName}") String driver,
			@Value("${database.url}") String url,
			@Value("${database.username}") String username, 
			@Value("${database.password}") String password
		) {
		Properties props = new Properties();
		props.setProperty("driver", driver);
		props.setProperty("url", url);
		props.setProperty("username", username);
		props.setProperty("password", password);
		DataSource dataSource = null;
		try {
			dataSource = BasicDataSourceFactory.createDataSource(props);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return dataSource;
	}
}	
```

上述的 **@Conditional(DatabaseConditional.class)** 传入了 DatabaseConditional 类，传入 @Conditional 注解的类需要实现 Condition 接口。

```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class DatabaseConditional implements Condition {

    /**
	 * 数据库装配条件 不符合条件的就不装配
	 * @param context 条件上下文
	 * @param 
	 */
	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		Environment env = context.getEnvironment();
        // 此处是判断条件
		return env.containsProperty("database.driverName") && env.containsProperty("database.url") 
				&& env.containsProperty("database.username") && env.containsProperty("database.password");
	}
}
```

只有满足了**上述自定义的条件**，才会**装配** DataSource 。



#### 属性设置

##### 1. @Value注解设置属性

使用 **@Value 注解**可以注入为 bean 设置属性，@Value 内可以直接设置属性值，也可以使用 Spring 表达式，也可以引用配置文件中的属性值。

```java
public class Person {
    // 通过普通的方式
    @Value("Tom")
    private String firstName;
    // spel方式来赋值
    @Value("#{28-8}")
    private Integer age;
    // 通过读取外部配置文件的值
    @Value("${person.lastName}")
    private String lastName;
}
```

##### 2.. 属性配置文件

###### (1) application.properties

是**默认**的配置文件，yml 文件也是类似的效果。

```properties
# application.properties中
database.url = jdbc:mysql://localhost:3306/test_db
database.username = root
database.password = 123456
```

可以在类中注入属性文件中的值。

```java
@Component
public class DatabaseProperties{
    
    @Value("${database.url}")   // 此处引用配置文件中的属性并注入
    private String url = null;
}
```

如果都用上述的方法可能需要写很多次，因此可以用如下方法直接指定属性文件中的值，如 database，然后用全限定名去定位值，如 database.url。

```java
@Component
@ConfigurationProperties("database")    // 使用此注解传入字符串database，会去配置文件寻找对应的属性自动注入
public class DatabaseProperties{
    
    private String url = null;
    // Getters and Setters
}
```

###### (2) 自定义属性配置文件

如把数据库连接信息放入 **jdbc.properties** 中，然后使用 **@PropertySource** 去定义对应的**属性配置文件**, @PropertySource 注解需要在配置类中标注，value 值可以有多个。ignoreResourceNotFound 指示找不到文件就忽略。

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.springboot.chapter"})
 // 指定配置文件路径
@PropertySource(value = {"classpath:jdbc.properties"}, ignoreResourceNotFound = ture) 
public class IoCTest{
    public static void main(String[] args){
        SpringApplication.run(IoCTest.class, args);
    }
}
```



#### 依赖注入DI

##### 1. 概述

**定义**：在依赖注入的模式下，创建被调用者得工作不再由调用者来完成，创建被调用者实例的工作通常由 Spring 容器完成，然后注入调用者。**创建对象时，向类里的属性设置值**。

所谓依赖注入，就是把底层类作为参数传入上层类，实现上层类对下层类的控制。DI 依赖注入，向类里面属性注入值 ，依赖注入不能单独存在，需要在 IOC 基础上完成操作。

> **IOC与DI的区别**

1. IOC 是**控制反转**，即把**对象创建交给 Spring 配置**。
2. DI 是**依赖注入**，向**类里面属性注入值**。
3. 两者的关系，**依赖注入不能单独存在，需要在 IOC 基础上完成操作**。

##### 2. 依赖注入方式

一般而言，依赖注入可以分为 3 种方式。

- **构造器**注入。
- **setter** 注入。
- 接口注入。
- **注解**注入(@Autowired) 

**构造器注入和 setter 注入和注解注入**是主要的方式，而接口注入是从别的地方注入的方式。

###### (1) 构造器注入

构造器注入**依赖于构造方法**实现，而构造方法可以是**有参数的或者是无参数**的。我们一般是通过类的构造方法来创建类对象，Spring 也可以采用**反射的方式**，通过使用构造方法来完成注入，这就是构造器注入的原理。

```java
public class Role {
    private Long id;
    private String roleName;
    private String note;

    public Role(String roleName, String note) {
        this.roleName = roleName;
        this.note = note;
    }

    /******** setter and getter *******/
}
```

这个时候是**没有办法利用无参数的构造方法**去创建对象的，为了使 Spring 能够正确创建这个对象。

```xml
<bean id="role1" class="com.nano.pojo.Role">
    <constructor-arg index="0" value="总经理"/>
    <constructor-arg index="1" value="公司管理者"/>
</bean>
```

**constructor-arg** 元素用于定义类构造方法的**参数**，其中 **index** 用于定义参数的**位置**，而 value 则是**设置值**，通过这样的定义 Spring 便知道使用 Role(String,String) 这样的**构造方法**去创建对象了。这样注入比较简单的，但是缺点也很明显，如果参数很多，那么这种构造方法就比较复杂了，这个时候应该考虑 setter 注入。

###### (2) setter注入

setter 注入利用 JavaBean 规范所定义的 setter 方法来完成注入，灵活且可读性高。它消除了使用构造器注入时出现**多个参数的可能性**，首先可以把构造方法声明为**无参数**的，然后使用 setter 注入为其设置对应的值，其实也是通过 Java **反射技术得以现实**的。这里假设先在代码清单中为 Role 类加入一个没有参数的构造方法，然后做代码清单的配置。

```xml
<bean id="role2" class="com.nano.pojo.Role">
    <property name="roleName" value="高级工程师"/>
    <property name="note" value="重要人员"/>
</bean
```

这样 Spring 就会通过**反射调用无参构造方法生成对象**，同时通过反射对应的 setter 注入配置的值了。这种方式是基于 xml 配置时 Spring 最为主要的注入方式。

###### (3) 接口注入

有些时候资源**并非来自于自身系统**，而是来自于外界，比如数据库连接资源完全可以在 Tomcat 下配置，然后通过 JNDI 的形式去获取它，这样数据库连接资源是属于开发工程外的资源，这个时候可以采用接口注入的形式来获取它。

比如在 Web 工程中，配置的数据源往往是通过服务器（比如Tomcat）去配置的，这个时候可以用 JNDI 的形式通过接口将它注入 Spring IoC 容器中来。

###### (4) 注解注入@Autowired

现在一般都用这个了吧。

> **注解@Autowired**

getBean() 方法支持根据类型和名称来获取对应的 bean。@Autowired 注解首先根据类型去**寻找对应的 bean**，找不到再根据**属性名称和 bean 名称**来寻找 bean。默认必须找到对应 Bean，否则报错（可以使用 required = false 关闭必须装配）。

@Autowired 可以标注在**属性**上，也可以标注在方法上，还可以标注在入参上。

```java
@Autowired	
private Animal animal = null;
```

```java
@Override
@Autowired
public void setAnimal(Animal animal) {
    this.animal = animal;
}
```

```java
public BussinessPerson(@Autowired Animal animal) {
    this.animal = animal;
}
```

> **使用@Primary与@Qualifier消除歧义问题**

两个接口：动物与人接口。

```java
public interface Person {
	public void service();
	public void setAnimal(Animal animal);
}
```

```java
public interface Animal {
	public void use();
}
```

实现接口：

**狗类**

```java
@Component
public class Dog implements Animal {
	@Override
	public void use() {
		System.out.println("狗【" + Dog.class.getSimpleName()+"】是看门用的。");
	}
}
```

```java
@Component
public class BussinessPerson implements Person{
    // 自动注入实现了动物接口的类
    @Autowired	
    private Animal animal = null;

    @Override
    public void service() {
        this.animal.use();
    }

    @Override
    public void setAnimal(Animal animal) {
        this.animal = animal;
    }
}
```

上述 BussinessPerson 中自动注入实现了动物接口的类，此时容器中实现了 Animal 接口的**只有 Dog 类**，因此成功注入 Dog 的实例。如果再实现一个动物类。

**猫类**

```java
@Component
public class Cat implements Animal {
	@Override
	public void use() {
		System.out.println("猫【" + Cat.class.getSimpleName()+"】是抓老鼠。");
	}
}
```

此时 Dog 类和 Cat 类都实现了 Animal 接口。BussinessPerson 的自动注入会报错，因为**不知道注入哪一个**实例。产生注入失败是因为**按类型查找**，动物 Animal 接口有多个类型，这就是存在歧义。

**注解 @Primary** 可以修改**优先权**。比如在 Cat 类上使用此注解。

```java
@Component
@Primary
public class Cat implements Animal {...}
```

此时容器会**优先**注入 Cat 实例到 Animal 中。@Primary 也可以用在多个类上，此时也会有歧义，可以使用 **@Qualifier 注解**。@Qualifier 注解的配置项 **value** 需要一个字符串去定义，它可以与 @Autowired 一起去通过**类型域名称一起寻找 Bean**。如下。此时注入的就是 Dog 类的实例。

```java
@Autowired
@Qualifier("dog")
Animal animal = null;
```



#### Bean的生命周期

Bean**定义**、Bean**初始化**、Bean**生存期**、Bean**销毁**。

##### 1. Spring初始化Bean流程

- **资源定位**(例如 @ComponentScan 所定义的扫描包)。
- Bean 定义(将 Bean 的定义保存到 BeanDefinition 的实例中)。
- 发布 Bean 定义( IOC 容器装载 Bean 的定义)。
- 实例化(创建 Bean 的实例对象)。
- 依赖注入(例如 @Autowired 类的各类资源)。

![1564994604731](assets/1564994604731.png)

##### 2. Spring Bean的生命周期

![1564994754829](assets/1564994754829.png)

- Bean 容器找到**配置文件**中 Spring Bean 的定义。
- Bean 容器利用**反射**创建一个 Bean 的**实例**。
- 如果涉及到一些属性值 利用 set 方法**设置一些属性值**。
- 如果 Bean 实现了 **BeanNameAware** 接口，调用 setBeanName() 方法，传入 **Bean 的名字**。
- 如果 Bean 实现了 **BeanClassLoaderAware** 接口，调用 setBeanClassLoader() 方法，传入 ClassLoader 对象的实例。
- 如果 Bean 实现了 **BeanFactoryAware** 接口，调用 setBeanFactory() 方法，传入 **BeanFactory 对象**的实例。
- 与上面的类似，如果实现了**其他 *Aware 接口**，就调用相应的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 **BeanPostProcessor** 对象，执行 **postProcessBeforeInitialization**() 方法。
- 如果 Bean 实现了 **InitializingBean** 接口，执行 **afterPropertiesSet**() 方法。
- 如果 Bean 在配置文件中的定义包含 **init-method** 属性，执行指定的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 **BeanPostProcessor** 对象，执行 **postProcessAfterInitialization**() 方法
- 当要销毁 Bean 的时候，如果 Bean 实现了 **DisposableBean** 接口，执行 **destroy**() 方法。
- 当要销毁 Bean 的时候，如果 Bean 在配置文件中的定义包含 **destroy-method** 属性，执行指定的方法。

![image-20200528144152364](assets/image-20200528144152364.png)

<img src="../../JavaNotes/H Spring/assets/image-20200528144205341.png" alt="image-20200528144205341" style="zoom:90%;" />

若容器注册了以上各种**接口**，程序那么将会按照以上的流程进行。下面将仔细讲解各接口作用。

##### 3. 各个接口方法分类

Bean 的完整生命周期经历了各种**方法调用**，这些方法可以划分为以下几类：

- **Bean 自身的方法**：这个包括了 Bean 本身调用的方法和通过配置文件中 **\<bean>** 的 **init-method** 和 **destroy-method** 指定的**初始化以及销毁**方法。

- **Bean 级生命周期接口方法**：这个包括了 BeanNameAware、BeanFactoryAware、InitializingBean 和DiposableBean 这些接口的方法。

- **容器级生命周期接口方法**：这个包括了 **InstantiationAwareBeanPostProcessor** 和 **BeanPostProcessor** 这两个接口实现，一般称它们的实现类为“**后处理器**”。

- **工厂后处理器接口方法**：这个包括了 AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer 等等非常有用的**工厂后处理器接口**的方法。工厂后处理器也是**容器级**的。在应用上下文装配配置文件之后立即调用。

##### 4. 初始化initialization和销毁destroy方法

有时需要在 Bean 属性值 set 好之后和 Bean 销毁之前做一些事情，比如检查 Bean 中某个属性是否被正常的设置好值了。Spring 框架提供了多种方法让我们可以在 Spring Bean 的生命周期中执行 **initialization 和 pre-destroy** 方法。

**1. 实现InitializingBean和DisposableBean接口**

这两个接口都只包含**一个方法**。通过实现 **InitializingBean** 接口的 **afterPropertiesSet**() 方法可以在 Bean **属性值设置好之后**做一些操作，实现 **DisposableBean** 接口的 **destroy**() 方法可以在销毁 Bean 之前做一些操作。

例子如下：

```java
public class GiraffeService implements InitializingBean, DisposableBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("执行InitializingBean接口的afterPropertiesSet方法");
    }
    @Override
    public void destroy() throws Exception {
        System.out.println("执行DisposableBean接口的destroy方法");
    }
}
```

这种方法比较简单，但是**不建议使用**。因为这样会将 Bean 的实现和 Spring 框架**耦合在一起**。

**2. 在bean的配置文件中指定init-method和destroy-method方法**

Spring 允许用户创建自己的 **init 方法和 destroy 方法**，只要在 Bean 的配置文件中指定 init-method 和 destroy-method 的值就可以在 Bean 初始化时和销毁之前执行一些操作。

例子如下：

```java
public class GiraffeService {
    // 通过<bean>的destroy-method属性指定的销毁方法
    public void destroyMethod() throws Exception {
        System.out.println("执行配置的destroy-method");
    }
    // 通过<bean>的init-method属性指定的初始化方法
    public void initMethod() throws Exception {
        System.out.println("执行配置的init-method");
    }
}
```

配置文件中的配置：

```xml
<bean name="giraffeService" class="com.giraffe.spring.service.GiraffeService" init-method="initMethod" destroy-method="destroyMethod">
</bean>
```

需要注意的是自定义的 init-method 和 post-method 方法可以抛异常但是不能有参数。这种方式比较推荐，因为可以自己创建方法，无需将 Bean 的实现直接依赖于 Spring 的框架。

**3. 使用@PostConstruct和@PreDestroy注解**

除了 xml 配置的方式，Spring 也支持用 **@PostConstruct** 和 **@PreDestroy** 注解来指定 init 和 destroy 方法。这两个注解均在 javax.annotation 包中。例子如下：

```java
public class GiraffeService {
    @PostConstruct
    public void initPostConstruct(){
        System.out.println("执行PostConstruct注解标注的方法");
    }
    @PreDestroy
    public void preDestroy(){
        System.out.println("执行preDestroy注解标注的方法");
    }
}
```

为了注解可以生效，需要在**配置文件**中定义org.springframework.context.annotation.**CommonAnnotationBeanPostProcessor**  或 context:annotation-config。配置文件：

```xml
<bean class="org.springframework.context.annotation.CommonAnnotationBeanPostProcessor" />
```

##### 5. 实现*Aware接口在Bean中使用Spring框架的一些对象

有些时候需要在 **Bean 的初始化**中使用 **Spring ==框架自身==的一些对象**来执行一些操作，比如获取 ServletContext 的一些参数，获取 ApplicaitionContext 中的 BeanDefinition 的名字，获取 Bean 在容器中的名字等等。**==为了让 Bean 可以获取到框架自身的一些对象，Spring 提供了一组名为 *Aware 的接口。==**

这些接口均继承于 org.springframework.beans.factory.**Aware 标记接口**，并提供一个将由 **Bean 实现的 set* 方法**， Spring 通过基于 **setter 的依赖注入方式**使相应的对象可以被 Bean 使用。

介绍一些**重要的 Aware 接口**：

- **ApplicationContextAware**：获得 **ApplicationContext** 对象，可以用来获取**所有 BeanDefinition** 的名字。
- **BeanFactoryAware**：获得 **BeanFactory** 对象，可以用来检测 Bean 的作用域。
- **BeanNameAware**：获得 Bean 在**配置文件**中定义的名字。
- **ResourceLoaderAware**：获得 ResourceLoader 对象，可以获得 **classpath** 中某个文件。
- **ServletContextAware**：在一个 MVC 应用中可以获取 **ServletContext** 对象，可以读取 context 中的参数。
- **ServletConfigAware**：在一个 MVC 应用中可以获取 **ServletConfig** 对象，可以读取 config 中的参数。

以下是实现上述接口的例子。

```java
public class GiraffeService implements ApplicationContextAware,
ApplicationEventPublisherAware, BeanClassLoaderAware, BeanFactoryAware,
BeanNameAware, EnvironmentAware, ImportAware, ResourceLoaderAware{
    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.println("执行setBeanClassLoader,ClassLoader Name = " + classLoader.getClass().getName());
    }
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("执行setBeanFactory,setBeanFactory:: giraffe bean singleton=" +  beanFactory.isSingleton("giraffeService"));
    }
    @Override
    public void setBeanName(String s) {
        System.out.println("执行setBeanName:: Bean Name defined in context=" + s);
    }
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("执行setApplicationContext:: Bean Definition Names="
                           + Arrays.toString(applicationContext.getBeanDefinitionNames()));
    }
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        System.out.println("执行setApplicationEventPublisher");
    }
    @Override
    public void setEnvironment(Environment environment) {
        System.out.println("执行setEnvironment");
    }
    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        Resource resource = resourceLoader.getResource("classpath:spring-beans.xml");
        System.out.println("执行setResourceLoader:: Resource File Name="
                           + resource.getFilename());
    }
    @Override
    public void setImportMetadata(AnnotationMetadata annotationMetadata) {
        System.out.println("执行setImportMetadata");
    }
}
```

##### 6. BeanPostProcessor

上面的 ***Aware** 接口是针对**==某个==实现这些接口的 Bean 定制初始化**的过程，Spring 同样可以针对容器中的**==所有 Bean==**，或者**==某些== Bean** 定制初始化过程，只需**提供一个实现 BeanPostProcessor 接口的类**即可。 该接口中包含两个方法， **postProcessBeforeInitialization** 和 **postProcessAfterInitialization**。 

```java
public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

postProcessBeforeInitialization 方法会在容器中的 **Bean 初始化之前执行，** postProcessAfterInitialization 方法在容器中的 **Bean 初始化之后**执行。

例子如下：

```java
public class CustomerBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("执行BeanPostProcessor的postProcessBeforeInitialization方法,beanName=" + beanName);
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("执行BeanPostProcessor的postProcessAfterInitialization方法,beanName=" + beanName);
        return bean;
    }
}
```

要将 BeanPostProcessor 的 Bean 像其他 Bean 一样定义在配置文件中配置。

```xml  
<bean class="com.giraffe.spring.service.CustomerBeanPostProcessor"/>
```

##### 7. 单例与非单例对象生命周期

其实很多时候并不会真的去实现上面说描述的那些接口，那么下面就除去那些接口，针对 bean 的单例和非单例来描述下 bean 的生命周期：

###### (1) 单例管理的对象

当 scope = "singleton"，即默认情况下，会在**启动容器时（即实例化容器时）时实例化**。但可以指定 Bean 的 **lazy-init= "true"** 来延迟初始化 bean，这时候只有在第一次获取 bean 时才会初始化 bean，即第一次请求该 bean 时才初始化。如下配置：

```xml
<bean id="ServiceImpl" class="cn.csdn.service.ServiceImpl" lazy-init="true"/>  
```

如果想对**所有的**默认单例 bean 都应用延迟初始化，可以在根节点 beans 设置 default-lazy-init 属性为 true，如下所示：

```xml
<beans default-lazy-init="true" …>
```

默认情况下，Spring 在读取 xml 文件的时候，就会**创建对象**。在创建对象的时候先**调用构造**器，然后调用 **init-method 属性值**中所指定的方法。对象在被销毁的时候，会调用 **destroy-method 属性值**中所指定的方法（例如调用 Container.destroy() 方法的时候）。写一个测试类，代码如下：

```java
public class LifeBean {
    private String name;  

    public LifeBean(){  
        System.out.println("LifeBean()构造函数");  
    }  
    public String getName() {  
        return name;  
    }  

    public void setName(String name) {  
        System.out.println("setName()");  
        this.name = name;  
    }  

    public void init(){  
        System.out.println("this is init of lifeBean");  
    }  

    public void destory(){  
        System.out.println("this is destory of lifeBean " + this);  
    }  
}
```

life.xml 配置如下：

```xml
<bean id="life_singleton" class="com.bean.LifeBean" scope="singleton" 
      init-method="init" destroy-method="destory" lazy-init="true"/>
```

测试代码：

```java
public class LifeTest {
    @Test 
    public void test() {
        AbstractApplicationContext container = 
        new ClassPathXmlApplicationContext("life.xml");
        LifeBean life1 = (LifeBean)container.getBean("life");
        System.out.println(life1);
        container.close();
    }
}
```

运行结果：

```
LifeBean()构造函数
this is init of lifeBean
com.bean.LifeBean@573f2bb1
……
this is destory of lifeBean com.bean.LifeBean@573f2bb1
```

###### (2) 非单例管理的对象

当`scope=”prototype”`时，容器也会延迟初始化 bean，Spring 读取xml 文件的时候，并不会立刻创建对象，而是在第一次请求该 bean 时才初始化（如调用getBean方法时）。在第一次请求每一个 prototype 的bean 时，Spring容器都会调用其构造器创建这个对象，然后调用`init-method`属性值中所指定的方法。对象销毁的时候，Spring 容器不会帮我们调用任何方法，因为是非单例，这个类型的对象有很多个，Spring容器一旦把这个对象交给你之后，就不再管理这个对象了。

为了测试prototype bean的生命周期 life.xml 配置如下：

```xml
<bean id="life_prototype" class="com.bean.LifeBean" scope="prototype" init-method="init" destroy-method="destory"/>
```

测试程序：

```java
public class LifeTest {
    @Test 
    public void test() {
        AbstractApplicationContext container = new ClassPathXmlApplicationContext("life.xml");
        LifeBean life1 = (LifeBean)container.getBean("life_singleton");
        System.out.println(life1);

        LifeBean life3 = (LifeBean)container.getBean("life_prototype");
        System.out.println(life3);
        container.close();
    }
}
```

运行结果：

```
LifeBean()构造函数
this is init of lifeBean
com.bean.LifeBean@573f2bb1
LifeBean()构造函数
this is init of lifeBean
com.bean.LifeBean@5ae9a829
……
this is destory of lifeBean com.bean.LifeBean@573f2bb1
```

可以发现，对于作用域为 prototype 的 bean ，其`destroy`方法并没有被调用。**如果 bean 的 scope 设为prototype时，当容器关闭时，`destroy` 方法不会被调用。对于 prototype 作用域的 bean，有一点非常重要，那就是 Spring不能对一个 prototype bean 的整个生命周期负责：容器在初始化、配置、装饰或者是装配完一个prototype实例后，将它交给客户端，随后就对该prototype实例不闻不问了。** 不管何种作用域，容器都会调用所有对象的初始化生命周期回调方法。但对prototype而言，任何配置好的析构生命周期回调方法都将不会被调用。**清除prototype作用域的对象并释放任何prototype bean所持有的昂贵资源，都是客户端代码的职责**（让Spring容器释放被prototype作用域bean占用资源的一种可行方式是，通过使用bean的后置处理器，该处理器持有要被清除的bean的引用）。谈及prototype作用域的bean时，在某些方面你可以将Spring容器的角色看作是Java new操作的替代者，任何迟于该时间点的生命周期事宜都得交由客户端来处理。

**Spring 容器可以管理 singleton 作用域下 bean 的生命周期，在此作用域下，Spring 能够精确地知道bean何时被创建，何时初始化完成，以及何时被销毁。而对于 prototype 作用域的bean，Spring只负责创建，当容器创建了 bean 的实例后，bean 的实例就交给了客户端的代码管理，Spring容器将不再跟踪其生命周期，并且不会管理那些被配置成prototype作用域的bean的生命周期。**

##### 8. demo

参考这个：https://www.cnblogs.com/zrtqsk/p/3735273.html





#### 参考资料

- [Spring 依赖注入]https://www.cnblogs.com/ooo0/p/10962360.html