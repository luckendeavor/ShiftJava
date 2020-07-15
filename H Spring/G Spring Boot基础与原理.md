

[TOC]

### Spring Boot基础

#### 基础

##### 1. 什么是Spring Boot

Spring Boot 是由 Pivotal 团队提供的全新框架，其设计目的是用来**简化新 Spring 应用的初始搭建以及开发过程**。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。Spring 有很多子项目，比如 core、context、bean、mvc 等。Spring-Boot 正是为了**解决繁复的代码配置**而产生的。Spring-Boot 也是基于 java-base 开发的代码，及不用 xml 文件配置，所有代码都由 Java 来完成。还可以加入 Groovy 的动态语言执行。

##### 2. POM文件

###### (1) 基本依赖

一般的 Spring Boot 依赖如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.9.RELEASE</version>
</parent>
```

其父项目是：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>1.5.9.RELEASE</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
</parent>
```

它才真正管理 Spring Boot 应用里面的所有**依赖的版本**。如果这个项目里面有依赖的版本号，那么导入依赖的时候就不需要写版本号，否则需要指定版本号。

###### (2) 启动器

Spring Boot 将所有的**功能场景**都抽取出来，做成一个个的 **starters（启动器）**，只需要在项目里面引入这些 starter 相关场景的所有依赖都会导入进来。要用什么功能就导入什么场景的启动器。比如：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

spring-boot-starter：启动器；导入了 **web 模块**正常运行所依赖的组件。

###### (3) 主入口类及注解分析

```java
@SpringBootApplication
public class HelloApplication {
    public static void main(String[] args) {
        SpringApplication.run(HelloWorldMainApplication.class,args);
    }
}
```

**@SpringBootApplication**：Spring Boot 应用标注在某个类上说明这个类是 SpringBoot 的**主配置类**，SpringBoot 就应该运行这个类的 main 方法来启动 SpringBoot 应用。@SpringBootApplication 源码如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

@**SpringBootConfiguration**：表示这是一个 Spring Boot 的配置类。

@**Configuration**：表示这是一个普通的**配置类**；配置类也是容器中的一个组件。

@**EnableAutoConfiguration**：开启自动配置功能。

```java
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

@**AutoConfigurationPackage**：自动配置包。

@**Import**(AutoConfigurationPackages.Registrar.class)：Spring 的底层注解 @Import，给容器中**导入一个组件**；导入的组件由 AutoConfigurationPackages.Registrar.class；==将**主配置类**（@SpringBootApplication 标注的类）的**所在包及下面所有子包里面的所有组件扫描到 Spring 容器；**==

@**Import**(EnableAutoConfigurationImportSelector.class)；

**EnableAutoConfigurationImportSelector**：是导入哪些组件的选择器，将所有需要导入的组件以全类名的方式返回，这些组件就会被添加到容器中，会给容器中导入非常多的**自动配置类（xxxAutoConfiguration）**；就是给容器中导入这个场景需要的所有组件，并配置好这些组件；

![自动配置类](assets/搜狗截图20180129224104.png)

有了**自动配置类**，免去了手动编写配置注入功能组件等的工作：

```java
SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class, classLoader)；
```

==Spring Boot 在启动的时候从**类路径下的 META-INF/spring.factories 中获取 EnableAutoConfiguration 指定的值**，将这些**值作为自动配置类**导入到容器中，自动配置类就生效，进行自动配置工作；==J2EE 的整体整合解决方案和自动配置都在 spring-boot-autoconfigure-1.5.9.RELEASE.jar 中。

###### (4) 项目结构

默认生成的 Spring Boot 项目中主程序已经生成好了。

**resources 文件夹**中目录结构：

- **static**：保存所有的**静态资源**； js css  images。
- **templates**：保存所有的**模板页面**；（Spring Boot 默认 jar 包使用嵌入式的 Tomcat，默认不支持 JSP 页面）；可以使用模板引擎（freemarker、thymeleaf）。
- **application.properties**：Spring Boot 应用的配置文件；可以修改一些默认设置。



#### 配置文件

##### 1. 概述

配置文件的作用：**修改 SpringBoot 自动配置的默认值**。

SpringBoot 使用一个**全局配置文件**，配置文件名是固定的：

- application.properties
- application.yml

可以导入配置文件处理器，编写配置就有提示。

```xml
<!--导入配置文件处理器，配置文件进行绑定就会有提示-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

##### 2. YAML语法

###### (1) 基本语法

**k:(空格)v**：表示**一对键值对**（空格必须有）。以**空格**的缩进来控制层级关系；只要是左对齐的一列数据，都是同一个层级的。属性和值也是大小写敏感。

```yaml
server:
    port: 8081
    path: /hello
```

对象、Map（属性和值）写法：

```yaml
friends:
	lastName: zhangsan
	age: 20
```

单行写法：

```yaml
friends: {lastName: zhangsan,age: 18}
```

数组（List、Set）写法：用 **- 值表示数组中的一个元素**。

```yaml
pets:
 - cat
 - dog
 - pig
```

单行写法：

```yaml
pets: [cat,dog,pig]
```

##### 3. 配置文件值注入

###### (1) @ConfigurationProperties注入属性值

一个自定义的配置文件如下：

```yaml
person:
    lastName: hello
    age: 18
    boss: false
    birth: 2017/12/12
    maps: {k1: v1,k2: 12}
    lists:
      - lisi
      - zhaoliu
    dog:
      name: 小狗
      age: 12
```

Person 类如下。这里将配置文件中配置的每一个属性的值，映射到这个组件中，**@ConfigurationProperties** 告诉SpringBoot 将本类中的**所有属性和配置文件中相关的配置进行绑定**，prefix = "person" 配置文件中哪个下面的所有属性进行一一映射，只有这个组件是容器中的组件，才能容器提供的 @ConfigurationProperties 功能。

```java
@Component
@ConfigurationProperties(prefix = "person")
public class Person {
    private String lastName;
    private Integer age;
    private Boolean boss;
    private Date birth;
    private Map<String,Object> maps;
    private List<Object> lists;
    private Dog dog;
}
```

###### (2) @PropertySource&@ImportResource&@Bean

@**PropertySource**：加载指定的**配置文件**。

```java
@PropertySource(value = {"classpath:person.properties"})
@Component
@ConfigurationProperties(prefix = "person")
public class Person {
    private String lastName;
    private Integer age;
    private Boolean boss;
}
```

@**ImportResource**：导入 Spring 的配置文件，让配置文件里面的内容生效；因为 pring Boot 里面没有 Spring 的配置文件，自己编写的配置文件也不能自动识别，想让 Spring 的配置文件生效，并加载进来就需要把 @**ImportResource** 标注在一个**配置类**上。

```java
@ImportResource(locations = {"classpath:beans.xml"})	// 导入Spring的配置文件让其生效
```

##### 4. 配置文件加载位置

Springboot 启动会扫描以下位置的 application.properties 或者 application.yml 文件作为 Spring boot 的**默认配置文件**

- file:./config/
- file:./
- classpath:/config/
- classpath:/

优先级**由高到底**，高优先级的配置会覆盖低优先级的配置。

还可以通过 spring.config.location 来改变默认的配置文件位置。项目打包好以后，可以使用**命令行参数的形式**，启动项目的时候来**指定配置文件的新位置**；指定配置文件和默认加载的这些配置文件共同起作用形成互补配置；

```java
java -jar spring-boot-02-config-02-0.0.1-SNAPSHOT.jar --spring.config.location=G:/application.properties
```

https://docs.spring.io/spring-boot/docs/1.5.9.RELEASE/reference/htmlsingle/#boot-features-external-config)



#### 自动配置原理

##### 1. 自动配置原理

SpringBoot 启动的时候加载**主配置类**，**开启**了自动配置功能 ==@**EnableAutoConfiguration**==。

@EnableAutoConfiguration 作用：

 - 利用 **EnableAutoConfigurationImportSelector** 给容器中导入一些组件。
- 可以查看 **selectImports**() 方法的内容；
- List\<String> configurations = getCandidateConfigurations(annotationMetadata，attributes);获取**候选**的配置。

**SpringFactoriesLoader.loadFactoryNames()** 扫描所有 jar 包类路径下的 **META-INF/spring.factories** 文件，把扫描到的这些**文件的内容包装成 properties 对象**，从 properties 中获取到 **EnableAutoConfiguration.class** 类（类名）对应的值，然后把他们**添加在容器**中。

**==将 类路径下 META-INF/spring.factories 里面配置的所有EnableAutoConfiguration的值加入到了容器中；==**

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\
org.springframework.boot.autoconfigure.elasticsearch.jest.JestAutoConfiguration,\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration,\
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration,\
org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration,\
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastJpaDependencyAutoConfiguration,\
org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration,\
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration,\
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration,\
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration,\
org.springframework.boot.autoconfigure.jooq.JooqAutoConfiguration,\
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.embedded.EmbeddedLdapAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.LdapAutoConfiguration,\
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration,\
org.springframework.boot.autoconfigure.mobile.DeviceResolverAutoConfiguration,\
org.springframework.boot.autoconfigure.mobile.DeviceDelegatingViewResolverAutoConfiguration,\
org.springframework.boot.autoconfigure.mobile.SitePreferenceAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.reactor.ReactorAutoConfiguration,\
org.springframework.boot.autoconfigure.security.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.SecurityFilterAutoConfiguration,\
org.springframework.boot.autoconfigure.security.FallbackWebSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.OAuth2AutoConfiguration,\
org.springframework.boot.autoconfigure.sendgrid.SendGridAutoConfiguration,\
org.springframework.boot.autoconfigure.session.SessionAutoConfiguration,\
org.springframework.boot.autoconfigure.social.SocialWebAutoConfiguration,\
org.springframework.boot.autoconfigure.social.FacebookAutoConfiguration,\
org.springframework.boot.autoconfigure.social.LinkedInAutoConfiguration,\
org.springframework.boot.autoconfigure.social.TwitterAutoConfiguration,\
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.jta.JtaAutoConfiguration,\
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration,\
org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.HttpMessageConvertersAutoConfiguration,\
org.springframework.boot.autoconfigure.web.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.ServerPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.web.WebClientAutoConfiguration,\
org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.WebSocketAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.WebSocketMessagingAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration
```

每一个这样的  xxxAutoConfiguration 类都是容器中的一个**组件**。

以 **HttpEncodingAutoConfiguration（Http编码自动配置）**为例解释自动配置原理：

```java
// 表示这是一个配置类，以前编写的配置文件一样，也可以给容器中添加组件
@Configuration   
// 启动指定类的ConfigurationProperties功能；将配置文件中对应的值和HttpEncodingProperties绑定起来；并把HttpEncodingProperties加入到ioc容器中
@EnableConfigurationProperties(HttpEncodingProperties.class)  
// Spring底层@Conditional注解（Spring注解版），根据不同的条件，如果满足指定的条件，整个配置类里面的配置就会生效；判断当前应用是否是web应用，如果是，当前配置类生效
@ConditionalOnWebApplication 
// 判断当前项目有没有这个类CharacterEncodingFilter；SpringMVC中进行乱码解决的过滤器；
@ConditionalOnClass(CharacterEncodingFilter.class)  
// 判断配置文件中是否存在某个配置  spring.http.encoding.enabled；如果不存在，判断也是成立的
// 即使我们配置文件中不配置pring.http.encoding.enabled=true，也是默认生效的；
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)  
public class HttpEncodingAutoConfiguration {

    // 已经和SpringBoot的配置文件映射了
    private final HttpEncodingProperties properties;

    // 只有一个有参构造器的情况下，参数值就会从容器中拿，上面已有一个条件装配，保证容器中必有这个类
    public HttpEncodingAutoConfiguration(HttpEncodingProperties properties) {
        this.properties = properties;
    }

    // 给容器中添加一个组件，这个组件的某些值需要从properties中获取
    @Bean   
    @ConditionalOnMissingBean(CharacterEncodingFilter.class) // 判断容器没有这个组件
    public CharacterEncodingFilter characterEncodingFilter() {
        CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
        filter.setEncoding(this.properties.getCharset().name());
        filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
        filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
        return filter;
    }
    
    @Bean
    public HttpEncodingAutoConfiguration.LocaleCharsetMappingsCustomizer localeCharsetMappingsCustomizer() {
        return new HttpEncodingAutoConfiguration.
            LocaleCharsetMappingsCustomizer(this.properties);
    }
    //...
}
```

一但这个配置类生效；这个配置类就会给容器中**添加各种组件**；这些组件的属性是从对应的 properties 类中获取的，这些类里面的每一个属性又是和配置文件绑定的。所有在配置文件中能配置的**属性都是在 xxxxProperties 类中封装**，配置文件能配置什么就可以参照某个功能对应的这个属性类。

```java
// 从配置文件中获取指定的值和bean的属性进行绑定
@ConfigurationProperties(prefix = "spring.http.encoding")  
public class HttpEncodingProperties {
    public static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");
    //...
}
```

**总结：**

**xxxxAutoConfigurartion**：自动配置类，给容器中添加组件。

**xxxxProperties**：封装配置文件中**相关属性**。

##### 2. @Conditional派生注解

作用：必须是 @Conditional 指定的**条件成立，才给容器中装配组件**，配置配里面的所有内容才生效。

|      @Conditional扩展注解       |           作用（判断是否满足当前指定条件）           |
| :-----------------------------: | :--------------------------------------------------: |
|       @ConditionalOnJava        |             系统的 Java 版本是否符合要求             |
|       @ConditionalOnBean        |                 容器中存在指定 Bean                  |
|    @ConditionalOnMissingBean    |                容器中不存在指定 Bean                 |
|    @ConditionalOnExpression     |                 满足 SpEL 表达式指定                 |
|       @ConditionalOnClass       |                   系统中有指定的类                   |
|   @ConditionalOnMissingClass    |                  系统中没有指定的类                  |
|  @ConditionalOnSingleCandidate  | 容器中只有一个指定的 Bean，或者这个 Bean 是首选 Bean |
|     @ConditionalOnProperty      |            系统中指定的属性是否有指定的值            |
|     @ConditionalOnResource      |             类路径下是否存在指定资源文件             |
|  @ConditionalOnWebApplication   |                   当前是 web 环境                    |
| @ConditionalOnNotWebApplication |                  当前不是 web 环境                   |
|       @ConditionalOnJndi        |                   JNDI 存在指定项                    |




