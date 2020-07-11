[TOC]

### Spring基础

#### 基本概念

##### 1. 什么是Spring框架?

Spring 是一种轻量级**开发框架**，旨在提高开发人员的开发效率以及系统的可维护性。

一般说 Spring 框架指的都是 **Spring Framework**，它是**很多模块**的集合，使用这些模块可以很方便地协助我们进行开发。这些模块是：**核心容器、数据访问/集成,、Web、AOP（面向切面编程）、工具、消息和测试模块**。比如：Core Container 中的 Core 组件是Spring 所有组件的核心，Beans 组件和 Context 组件是实现IOC和依赖注入的基础，AOP组件用来实现面向切面编程。

##### 2. Spring模块

下图是 Spring4.x 版本。目前最新的 5.x 版本中 Web 模块的 Portlet 组件已经被废弃掉，同时增加了用于异步响应式处理的 **WebFlux** 组件。

<img src="https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/Spring主要模块.png" alt="Spring主要模块" style="zoom: 67%;" />

- **Spring Core：** 核心模块，Spring 其他所有的功能都需要依赖于该类库。主要提供 **IoC 依赖注入**功能。
- **Spring  Aspects** ： 该模块为与 AspectJ 的集成提供支持。
- **Spring AOP** ：提供了**面向切面的编程**实现。
- **Spring JDBC** : Java **数据库**连接。
- **Spring JMS** ：Java **消息**服务。
- **Spring ORM** : 用于支持 Hibernate 等 **ORM** 工具。
- **Spring Web** : 为创建 Web 应用程序提供支持。
- **Spring Test** : 提供了对 JUnit 和 TestNG 测试的支持。



#### Spring EL

- @Value 中的 **${...}** 代表**占位符**，它会读取上下文的属性值装配到属性中, 如属性文件中的值。
- **\#{...}**  代表启用 Spring 表达式，它将具有**运算**的功能。

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
 
@Component("customerBean")
public class Customer {
 
    @Value("#{'lei'.toUpperCase()}")        	// Spring EL
    private String name;
 
    @Value("#{priceBean.getSpecialPrice()}")    // Spring EL
    private double amount;
    
    @Value("${database.driverName}")        	// Spring EL
    String driver = null;
    
    // getter and setter...省略
 
    @Override
    public String toString() {
        return "Customer [name=" + name + ", amount=" + amount + "]";
    }
}
```









