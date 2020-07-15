[TOC]

### Spring注解总结

#### 启动注解@SpringBootApplication

@SpringBootApplication 用于标注程序主启动类。

```java
@SpringBootApplication
public class SpringBootApplication {
    public static void main(java.lang.String[] args) {
        SpringApplication.run(SpringSecurityJwtGuideApplication.class, args);
    }
}
```

**@SpringBootApplication** 源码如下：

```java
package org.springframework.boot.autoconfigure;
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
   //......
}
```

**@SpringBootConfiguration** 源码如下：

```java
package org.springframework.boot;
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```

- **@EnableAutoConfiguration**：启用 SpringBoot 的**自动配置机制**。
- **@ComponentScan**： 扫描被 @Component (@Service,@Controller)注解的 bean，**注解默认会扫描该类所在的包下所有的类**。
- **@Configuration**：允许在 Spring 上下文中注册额外的 bean 或导入其他配置类。



#### Bean相关注解

##### 1. @Autowired

自动导入对象到类中，被注入进的类同样要被 Spring 容器管理比如：Service 类注入到 Controller 类中。

##### 2. @Component,@Repository,@Service, @Controller

一般使用 @Autowired 注解让 Spring 容器**自动装配 bean**。要想把类标识成可**用于 @Autowired 注解自动装配**的 bean 的类，可以采用以下注解实现：

- **@Component**：**通用**的注解，可标注任意类为 Spring 组件。如果一个 Bean 不知道属于哪个层，可以使用@Component 注解标注。
- **@Repository**：对应持久层即 Dao 层，主要用于数据库相关操作。
- **@Service**：对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
- **@Controller**：对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。

##### 3. @Configuration

一般用来声明**配置类**，可以使用 @Component 注解替代，不过使用 @Configuration 注解声明配置类更加语义化。

```java
@Configuration
public class AppConfig {
    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

##### 4. @Scope

声明 Spring Bean 的**作用域**，使用方式如下：

```java
@Bean
@Scope("singleton")
public Person personSingleton() {
    return new Person();
}
```

**四种常见的 Bean 作用域：**

- **singleton**：唯一 bean 实例，Spring 中的 bean 默认都是**单例**的。
- **prototype**：每次请求都会创建一个新的 bean 实例。
- **request**：每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
- **session**：每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。

#### HTTP请求相关注解

##### 1. GET请求

**@GetMapping**("users") 等价于@RequestMapping(value="/users", method = RequestMethod.GET)

```java
@GetMapping("/users")
public ResponseEntity<List<User>> getAllUsers() {
    return userRepository.findAll();
}
```

##### 2. POST请求

**@PostMapping**("users") 等价于@RequestMapping(value="/users", method = RequestMethod.POST)

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserCreateRequest userCreateRequest) {
    return userRespository.save(user);
}
```

##### 3. PUT请求

**@PutMapping**("/users/{userId}") 等价于 @RequestMapping(value="/users/{userId}", method = RequestMethod.PUT)

```java
@PutMapping("/users/{userId}")
public ResponseEntity<User> updateUser(@PathVariable(value = "userId") Long userId,
                                       @Valid @RequestBody UserUpdateRequest userUpdateRequest) {
    //......
}
```

##### 4. DELETE请求

**@DeleteMapping**("/users/{userId}")等价于 @RequestMapping(value="/users/{userId}", method = RequestMethod.DELETE)

```java
@DeleteMapping("/users/{userId}")
public ResponseEntity deleteUser(@PathVariable(value = "userId") Long userId){
    //......
}
```

##### 5. PATCH请求

用得少，一般都是 PUT 不够用了之后才用 PATCH 请求去更新数据。

```java
@PatchMapping("/profile")
public ResponseEntity updateStudent(@RequestBody StudentUpdateRequest studentUpdateRequest) {
    studentRepository.updateDetail(studentUpdateRequest);
    return ResponseEntity.ok().build();
}
```

#### 参数传递相关注解

##### 1. @PathVariable和@RequestParam

**@PathVariable 用于获取路径参数，@RequestParam 用于获取查询参数。**

举个简单的例子：

```java
@GetMapping("/user/{userId}/teachers")
public List<Teacher> getKlassRelatedTeachers(
    @PathVariable("klassId") Long klassId,
    @RequestParam(value = "type", required = false) String type) {
    //...
}
```

如果请求的 url 是：/user/{123456}/teachers?type=web，那么服务获取到的数据就是：userId=123456, type=web。

##### 2. @RequestBody

用于**读取 Request 请求**（可能是 POST, PUT, DELETE, GET 请求）的 **body 部分**并且 **Content-Type** 为 **application/json 格式**的数据，接收到数据之后会自动将数据**绑定到 Java 对象**上去。系统会使用**HttpMessageConverter** 或者自定义的 HttpMessageConverter 将请求的 body 中的 json 字符串**转换为 Java 对象**。

有一个注册的接口：

```java
@PostMapping("/registry")
public ResponseEntity registry(@RequestBody @Valid User user) {
    userService.save(user);
    return ResponseEntity.ok().build();
}
```

UserRegisterRequest对象：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {
    @NotBlank
    private String userName;
    @NotBlank
    private String password;
    @FullName
    @NotBlank
    private String fullName;
}
```

发送 post 请求到这个接口，并且 body 携带 JSON 数据：

```json
{"userName":"coder","fullName":"shuangkou","password":"123456"}
```

后端就可以直接把 json 格式的数据解析并**映射**到 User 类上。

注意：一个请求方法只可以有一个 @RequestBody，但是可以有**多个** @RequestParam 和 @PathVariable。 



#### 配置信息读取注解

加入一个配置文件如下：

```yaml
name: Jack

my-profile:
  name: Lucy
  email: 11234@163.com

library:
  location: 火星
  books:
    - name: 计算机网络
      description: test
    - name: 操作系统
      description: test
    - name: 编译原理
      description: test
```

##### 1. @value

**常用**，使用 **@Value("${property}")** 读取比较简单的配置信息：

```java
@Value("${name}")
String name;
```

##### 2. @ConfigurationProperties

常用，通过 **@ConfigurationProperties** 读取配置信息**并与 bean 绑定**。

```java
@Component
// 这里的前缀与配置文件中的前缀对应
@ConfigurationProperties(prefix = "library")
class LibraryProperties {
    @NotEmpty
    private String location;
    private List<Book> books;

    @Setter
    @Getter
    @ToString
    static class Book {
        String name;
        String description;
    }
    //getter/setter
}
```

可以像使用普通的 Spring bean 一样，将其注入到类中使用。

##### 3. PropertySource

用得少，@PropertySource 读取**指定 properties 文件**。

```java
@Component
@PropertySource("classpath:website.properties")
class WebSite {
    @Value("${url}")
    private String url;
    //getter/setter
}
```



#### 参数校验

##### 1. 概述

**JSR(Java Specification Requests）** 是一套 JavaBean 参数校验的标准，它定义了很多常用的**校验注解**，可以直接将这些注解加在 JavaBean 的属**性上面**，就可以在需要校验的时候进行校验。

校验的时候实际用的是 **Hibernate Validator** 框架。目前最新版的 Hibernate Validator 6.x 是 Bean Validation 2.0（JSR 380）的**参考实现**。SpringBoot 项目的 spring-boot-**starter-web 依赖**中已经有 **hibernate-validator** 包，不需要引用相关依赖。

<img src="assets/c7bacd12-1c1a-4e41-aaaf-4cad840fc073.png" style="zoom:50%;" />

注意： 所有的参数校验相关的注解都**推荐使用 JSR 注解**，即 **javax.validation.constraints**，而不是org.hibernate.validator.constraints 的注解。

##### 2. 常用校验注解

|              注解               |                           含义                           |
| :-----------------------------: | :------------------------------------------------------: |
|          **@NotEmpty**          |          被注释的字符串的不能为 null 也不能为空          |
|          **@NotBlank**          |    被注释的字符串非 null，并且必须包含一个非空白字符     |
|            **@Null**            |                 被注释的元素必须为 null                  |
|          **@NotNull**           |                被注释的元素必须不为 null                 |
|         **@AssertTrue**         |                 被注释的元素必须为 true                  |
|        **@AssertFalse**         |          @AssertFalse 被注释的元素必须为 false           |
|   **@Pattern(regex=,flag=)**    |           被注释的元素必须符合指定的正则表达式           |
|           **@Email**            |              被注释的元素必须是 Email 格式               |
|         **@Min(value)**         | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
|         **@Max(value)**         | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
|     **@DecimalMin(value)**      | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
|     **@DecimalMax(value)**      | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
|      **@Size(max=, min=)**      |           被注释的元素的大小必须在指定的范围内           |
| **@Digits (integer, fraction)** |   被注释的元素必须是一个数字，其值必须在可接受的范围内   |
|            **@Past**            |             被注释的元素必须是一个过去的日期             |
|           **@Future**           |             被注释的元素必须是一个将来的日期             |

##### 3. 验证请求体(RequestBody)

使用上述的注解对前端传入的参数进行校验，如下：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Person {

    @NotNull(message = "classId不能为空")
    private String classId;

    @Size(max = 33)
    @NotNull(message = "name不能为空")
    private String name;

    @Pattern(regexp = "((^Man$|^Woman$|^UGM$))", message = "sex值不在可选范围")
    @NotNull(message = "sex不能为空")
    private String sex;

    @Email(message = "email格式不正确")
    @NotNull(message = "email不能为空")
    private String email;

}
```

在需要验证的**参数**上加上了 **@Valid** 注解，如果验证失败，它将抛出**MethodArgumentNotValidException** 异常。

```java
@RestController
@RequestMapping("/api")
public class PersonController {
	
    @PostMapping("/person")
    public ResponseEntity<Person> getPerson(@RequestBody @Valid Person person) {
        return ResponseEntity.ok().body(person);
    }
}
```

##### 4. 验证请求参数(Path Variables 和 Request Parameters)

一定不要忘记在**类上**加上 **@Validated** 注解了，这个参数可以告诉 Spring 去校验方法参数。

```java
@RestController
@RequestMapping("/api")
@Validated
public class PersonController {

    @GetMapping("/person/{id}")
    public ResponseEntity<Integer> getPersonByID(@Valid @PathVariable("id") @Max(value = 5,message = "超过 id 的范围了") Integer id) {
        return ResponseEntity.ok().body(id);
    }
}
```



#### 异常处理注解

全局处理 Controller 层异常是非常常见的需求。

**相关注解：**

1. **@ControllerAdvice**：注解定义**全局异常处理类**。
2. **@ExceptionHandler**：注解声明**异常处理方法**。

参数校验中可能抛出的 MethodArgumentNotValidException 异常，对这个异常进行处理。也就是声明一个全局异常处理类。然后指定需要处理的异常类型。

```java
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {

    /**
     * 请求参数异常处理
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> handleMethodArgumentNotValidException(MethodArgumentNotValidException ex, HttpServletRequest request) {
        //......
    }
}
```

更多关于 Spring Boot 异常处理的内容可参考：

1. [SpringBoot 处理异常的几种常见姿势](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485568&idx=2&sn=c5ba880fd0c5d82e39531fa42cb036ac&chksm=cea2474bf9d5ce5dcbc6a5f6580198fdce4bc92ef577579183a729cb5d1430e4994720d59b34&token=2133161636&lang=zh_CN#rd)
2. [使用枚举简单封装一个优雅的 Spring Boot 全局异常处理！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486379&idx=2&sn=48c29ae65b3ed874749f0803f0e4d90e&chksm=cea24460f9d5cd769ed53ad7e17c97a7963a89f5350e370be633db0ae8d783c3a3dbd58c70f8&token=1054498516&lang=zh_CN#rd)



#### JPA相关注解

##### 1. 创建表

**@Entity** 声明一个类对应一个数据库实体。

**@Table** 设置表名。

```java
@Entity
@Table(name = "role")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String description;
    //....
}
```

##### 2. 主键及主键生成策略

**@Id** ：声明一个字段为**主键**。

使用 @Id 声明之后还需要定义**主键的生成策略**。可以使用 **@GeneratedValue** 指定主键生成策略。

**(1) 通过 @GeneratedValue 直接使用 JPA 内置提供的四种主键生成策略来指定主键生成策略。**

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

JPA 使用枚举定义了 4 中常见的主键生成策略，如下：

```java
public enum GenerationType {
    // 使用一个特定的数据库表格来保存主键，持久化引擎通过关系数据库的一张特定的表格来生成主键
    TABLE,
    // 在某些数据库中,不支持主键自增长,比如Oracle、PostgreSQL其提供了一种叫做"序列(sequence)"的机制生成主键
    SEQUENCE,
    // 主键自增长，常用
    IDENTITY,
    // 把主键生成策略交给持久化引擎(persistence engine),持久化引擎会根据数据库在以上三种主键生成 策略中选择其中一种
    AUTO
}
```

@GeneratedValue 注解**默认**使用的策略是 GenerationType.**AUTO**。

```java
public @interface GeneratedValue {
    GenerationType strategy() default AUTO;
    String generator() default "";
}
```

一般使用 **MySQL** 数据库的话，使用 GenerationType.**IDENTITY** 策略比较普遍一点（分布式系统的话需要另外考虑**使用分布式 ID**）。

**(2) 通过 @GenericGenerator 声明一个主键策略，然后 @GeneratedValue 使用这个策略**

```java
@Id
@GeneratedValue(generator = "IdentityIdGenerator")
@GenericGenerator(name = "IdentityIdGenerator", strategy = "identity")
private Long id;
```

等价于：

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

JPA 提供的**主键生成策略**有如下几种：

```java
public class DefaultIdentifierGeneratorFactory implements 
    MutableIdentifierGeneratorFactory, Serializable, ServiceRegistryAwareService {

    @SuppressWarnings("deprecation")
    public DefaultIdentifierGeneratorFactory() {
        register( "uuid2", UUIDGenerator.class );
        // can be done with UUIDGenerator + strategy
        register( "guid", GUIDGenerator.class );	
        // "deprecated" for new use
        register( "uuid", UUIDHexGenerator.class );	
        // uuid.hex is deprecated
        register( "uuid.hex", UUIDHexGenerator.class ); 	
        register( "assigned", Assigned.class );
        register( "identity", IdentityGenerator.class );
        register( "select", SelectGenerator.class );
        register( "sequence", SequenceStyleGenerator.class );
        register( "seqhilo", SequenceHiLoGenerator.class );
        register( "increment", IncrementGenerator.class );
        register( "foreign", ForeignGenerator.class );
        register( "sequence-identity", SequenceIdentityGenerator.class );
        register( "enhanced-sequence", SequenceStyleGenerator.class );
        register( "enhanced-table", TableGenerator.class );
    }

    public void register(String strategy, Class generatorClass) {
        LOG.debugf( "Registering IdentifierGenerator strategy [%s] -> [%s]", strategy, generatorClass.getName() );
        final Class previous = generatorStrategyToClassNameMap.put( strategy, generatorClass );
        if ( previous != null ) {
            LOG.debugf( "    - overriding [%s]", previous.getName() );
        }
    }

}
```

##### 3. 设置字段及属性

**@Column** 声明字段及其**属性**。

**示例：**设置属性 userName 对应的数据库字段名为 user_name，长度为 32，非空。

```java
@Column(name = "user_name", nullable = false, length = 32)
private String userName;
```

设置字段类型并且加**默认值**，挺常用。

```java
Column(columnDefinition = "tinyint(1) default 1")
private Boolean enabled;
```

##### 4. 指定不持久化特定字段

**@Transient**：声明不需要与数据库映射的字段，在保存的时候**不需要持久化**保存进数据库 。

```java
@Entity(name="USER")
public class User {
    // 不会被持久化
    @Transient
    private String email; 
}
```

除了 @Transient 关键字声明， 还可以采用下面几种方法不持久化：

```java
static String secrect; // not persistent because of static
final String secrect = “Satish”; // not persistent because of final
transient String secrect; // not persistent because of transient
```

一般使用**注解**的方式比较多。

##### 5. 声明大字段

**@Lob**：声明某个字段为**大字段**。

```java
@Lob
private String content;
```

更详细的使用：

```java
@Lob
//指定Lob类型数据的获取策略，FetchType.EAGER表示非延迟加载，而FetchType.LAZY表示延迟加载 
@Basic(fetch = FetchType.EAGER)
// columnDefinition 属性指定数据表对应的Lob字段类型
@Column(name = "content", columnDefinition = "LONGTEXT NOT NULL")
private String content;
```

##### 6. 创建枚举类型的字段

可以使用**枚举类型**的字段，不过枚举字段要用 **@Enumerated** 注解修饰。创建一个**枚举类**。

```java
public enum Gender {
    MALE("男性"),
    FEMALE("女性");
    
    private String value;
    Gender(String str){
        value = str;
    }
}
```

在表中使用枚举类型：

```java
@Entity
@Table(name = "role")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String description;
    // 指定枚举类型
    @Enumerated(EnumType.STRING)
    private Gender gender;
    //省略getter/setter......
}
```

数据库里面对应存储的是 **MAIL/FEMAIL**。

##### 7. 增加审计功能

只要继承了 **AbstractAuditBase** 的类都会**默认加上下面四个字段**。

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@MappedSuperclass
@EntityListeners(value = AuditingEntityListener.class)
public abstract class AbstractAuditBase {

    @CreatedDate
    @Column(updatable = false)
    @JsonIgnore
    private Instant createdAt;

    @LastModifiedDate
    @JsonIgnore
    private Instant updatedAt;

    @CreatedBy
    @Column(updatable = false)
    @JsonIgnore
    private String createdBy;

    @LastModifiedBy
    @JsonIgnore
    private String updatedBy;
}
```

对应的**审计功能**的配置类可以是下面这样的（Spring Security 项目）:

```java
@Configuration
@EnableJpaAuditing
public class AuditSecurityConfiguration {
    @Bean
    AuditorAware<String> auditorAware() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

**@CreatedDate**：表示该字段为**创建时间**字段，在这个实体被 insert 的时候，会设置这个值。

**@CreatedBy**：表示该字段为**创建人**，在这个实体被 insert 的时候，会设置这个值。**@LastModifiedDate、@LastModifiedBy**同理。

**@EnableJpaAuditing**：**开启 JPA 审计**功能。

##### 8. 删除/修改数据

**@Modifying** 注解提示 JPA 该操作是**修改操作**，注意还要配合 **@Transactional 注解**使用。

```java
@Repository
public interface UserRepository extends JpaRepository<User, Integer> {
    @Modifying
    @Transactional(rollbackFor = Exception.class)
    void deleteByUserName(String userName);
}
```

##### 9. 关联关系

- **@OneToOne**：声明一对一关系
- **@OneToMany**：声明一对多关系
- **@ManyToOne**：声明多对一关系
- **MangToMang**：声明多对多关系

更多关于 Spring Boot JPA 的文章请看我的这篇文章：



#### 事务相关注解

##### 1. @Transactional

在方法上使用 **@Transactional** 注解即可开启事务!

```java
@Transactional(rollbackFor = Exception.class)
public void save() {
    //......
}
```

Exception 分为**运行时异常** RuntimeException 和**非运行时异常**。在 @Transactional 注解中如果不配置 rollbackFor 属性，那么事物只会在**遇到 RuntimeException 的时候才会回滚**，加上 rollbackFor=Exception.class，可以让事务在遇到**非运行时异常时**也回滚。

@Transactional 注解一般用在可以作用在**类或者方法**上。

- **作用于类**：当把 @Transactional 注解放在类上时，表示所有该类的 **public** 方法都配置相同的事务属性信息。
- **作用于方法**：当类配置了 @Transactional，方法也配置了 @Transactional，**方法的事务会覆盖类**的事务配置信息。

#### JSON相关注解

##### 1. 忽略JSON属性

**@JsonIgnoreProperties** 作用在**类上**用于**过滤掉特定字段不返回**或者不解析。

```java
// 生成json时将userRoles属性过滤
@JsonIgnoreProperties({"userRoles"})
public class User {
    private String userName;
    private String fullName;
    private String password;
    @JsonIgnore
    private List<UserRole> userRoles = new ArrayList<>();
}
```

**@JsonIgnore** 一般用于类的**属性**上，作用和上面的 @JsonIgnoreProperties 一样。

```java
public class User {
    private String userName;
    private String fullName;
    private String password;
    // 生成json时将userRoles属性过滤
    @JsonIgnore
    private List<UserRole> userRoles = new ArrayList<>();
}
```

##### 2. 格式化JSON数据

**@JsonFormat** 一般用来**格式化** json 数据。比如：

```java
@JsonFormat(shape = JsonFormat.Shape.STRING, pattern="yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", timezone="GMT")
private Date date;
```

##### 3. 扁平化对象

使用 **@JsonUnwrapped** 可以将对象进行**扁平化**。

```java
@Data
public class Account {
    @JsonUnwrapped
    private Location location;
    @JsonUnwrapped
    private PersonInfo personInfo;

    @Data
    public static class Location {
        private String provinceName;
        private String countyName;
    }
    @Data
    public static class PersonInfo {
        private String userName;
        private String fullName;
    }
}

```

**未扁平化**之前：

```json
{
    "location": {
        "provinceName":"111",
        "countyName":"222"
    },
    "personInfo": {
        "userName": "333",
        "fullName": "444"
    }
}
```

使用 @JsonUnwrapped 扁平对象之后：

```java
@Data
public class Account {
    @JsonUnwrapped
    private Location location;
    @JsonUnwrapped
    private PersonInfo personInfo;
    ......
}
```

```json
{
    "provinceName":"111",
    "countyName":"222",
    "userName": "333",
    "fullName": "444"
}
```

#### 测试相关注解

**@ActiveProfiles** 一般作用于**测试类**上， 用于声明生效的 Spring **配置文件**。

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
@Slf4j
public abstract class TestBase {
    //......
}
```

**@Test** 声明一个方法为**测试方法**。

**@Transactional** 被声明的测试方法的**数据会回滚**，避免污染测试数据。

**@WithMockUser** 是 Spring Security 提供的用来模拟一个真实用户的注解，并且可以赋予权限。

```java
@Test
@Transactional
@WithMockUser(username = "user-id-18163138155", authorities = "ROLE_TEACHER")
void userTest() throws Exception {
    //......
}
```







#### 参考资料

- [一文搞懂如何在 Spring Boot 正确中使用 JPA](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485689&idx=1&sn=061b32c2222869932be5631fb0bb5260&chksm=cea24732f9d5ce24a356fb3675170e7843addbfcc79ee267cfdb45c83fc7e90babf0f20d22e1&token=292197051&lang=zh_CN#rd) 。

