[TOC]

### Spring数据访问

#### JDBC

##### 1. 概述

依赖如下

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<scope>runtime</scope>
</dependency>
```

配置文件如下

```yaml
# yaml配置数据源
spring:
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://192.168.15.22:3306/jdbc
    driver-class-name: com.mysql.jdbc.Driver
```

```java
// 测试
@RunWith(SpringRunner.class)
@SpringBootTest
@Slf4j
public class DataBaseTest {
    // 注入配置的数据源
    @Autowired
    DataSource dataSource;

    @Test
    public void testDbConnection() throws SQLException {
        log.info(dataSource.getClass() + "");
        Connection connection = dataSource.getConnection();
        log.info(connection + "");
        connection.close();
    }
}
```

默认是用 org.apache.tomcat.jdbc.pool.**DataSource** 作为数据源；数据源的相关配置都在 **DataSourceProperties** 里面。

##### 2. JdbcTemplate

配置数据源后，容器中已经自动配置好了 **JdbcTemplate 类**，可以直接使用。定义服务接口：

```java
public interface JdbcTmplUserService {
	public User getUser(Long id);
	public List<User> findUsers(String userName, String note);
	public int insertUser(User user);
	public int updateUser(User user);
	public int deleteUser(Long id);
	public User getUser2(Long id);
	public User getUser3(Long id);
}
```

实现该接口：

```java
@Service
public class JdbcTmplUserServiceImpl implements JdbcTmplUserService {

    // 自动注入
	@Autowired
	private JdbcTemplate jdbcTemplate = null;

	// 获取映射关系
	private RowMapper<User> getUserMapper() {
		// 使用Lambda表达式创建用户映射关系
		RowMapper<User> userRowMapper = (ResultSet rs, int rownum) -> {
			User user = new User();
			user.setId(rs.getLong("id"));
			user.setUserName(rs.getString("user_name"));
			int sexId = rs.getInt("sex");
			SexEnum sex = SexEnum.getEnumById(sexId);
			user.setSex(sex);
			user.setNote(rs.getString("note"));
			return user;
		};
		return userRowMapper;
	}

	// 获取对象
	@Override
	public User getUser(Long id) {
		// 执行的SQL
		String sql = " select id, user_name, sex, note from t_user where id = ?";
		// 参数
		Object[] params = new Object[] { id };
		User user = jdbcTemplate.queryForObject(sql, params, getUserMapper());
		return user;
	}

	// 查询用户列表
	@Override
	public List<User> findUsers(String userName, String note) {
		// 执行的SQL
		String sql = " select id, user_name, sex, note from t_user " + "where user_name like concat('%', ?, '%') "
				+ "and note like concat('%', ?, '%')";
		// 参数
		Object[] params = new Object[] { userName, note };
		// 使用匿名类实现
		List<User> userList = jdbcTemplate.query(sql, params, getUserMapper());
		return userList;
	}

	// 插入数据库
	@Override
	public int insertUser(User user) {
		String sql = " insert into t_user (user_name, sex, note) values( ? , ?, ?)";
		return jdbcTemplate.update(sql, user.getUserName(), user.getSex().getId(), user.getNote());
	}

	// 更新数据库
	@Override
	public int updateUser(User user) {
		// 执行的SQL
		String sql = " update t_user set user_name = ?, sex = ?, note = ?  " + " where id = ?";
		return jdbcTemplate.update(sql, user.getUserName(), user.getSex().getId(), user.getNote(), user.getId());
	}

	// 删除数据
	@Override
	public int deleteUser(Long id) {
		// 执行的SQL
		String sql = " delete from t_user where id = ?";
		return jdbcTemplate.update(sql, id);
	}
	
}
```

对 JdbcTemplate 的映射关系需要**开发者自己实现 RowMapper 接口**，以完成数据库到 POJO 对象的**映射**。JdbcTemplate 每**调用一次就会进行一次数据库连接**，两条语句就会产生两次连接，不推荐。有时需要执行**多条SQL 时**，可以使用 **StatementCallback** 接口或 ConnectionCallback 接口，如下所示。

```java
public User getUser2(Long id) {
    // 通过Lambda表达式使用 StatementCallback
    User result = this.jdbcTemplate.execute((Statement stmt) -> {
        String sql1 = "select count(*) total from t_user where id= " + id;
        ResultSet rs1 = stmt.executeQuery(sql1);
        while (rs1.next()) {
            int total = rs1.getInt("total");
            System.out.println(total);
        }
        // 执行的SQL
        String sql2 = " select id, user_name, sex, note from t_user"
            + " where id = " + id;
        ResultSet rs2 = stmt.executeQuery(sql2);
        User user = null;
        while (rs2.next()) {
            int rowNum = rs2.getRow();
            user= getUserMapper().mapRow(rs2, rowNum);
        }
        return user;
    });
    return result;
}

public User getUser3(Long id) {
    // 通过Lambda表达式使用 ConnectionCallback 接口
    return this.jdbcTemplate.execute((Connection conn) -> {
        String sql1 = " select count(*) as total from t_user"
            + " where id = ?";
        PreparedStatement ps1 = conn.prepareStatement(sql1);
        ps1.setLong(1, id);
        ResultSet rs1 = ps1.executeQuery();
        while (rs1.next()) {
            System.out.println(rs1.getInt("total"));
        }
        String sql2 = " select id, user_name, sex, note from t_user "
            + "where id = ?";
        PreparedStatement ps2 = conn.prepareStatement(sql2);
        ps2.setLong(1, id);
        ResultSet rs2 = ps2.executeQuery();
        User user = null;
        while (rs2.next()) {
            int rowNum = rs2.getRow();
            user= getUserMapper().mapRow(rs2, rowNum);
        }
        return user;
    });
}
```

####  数据源与数据库

##### 1. 配置数据库

使用 spring-boot-starter-jpa 之后默认使用**内存数据库**，如 h2, hqldb, Derby 等。

###### (1) 使用默认数据库

依赖如下

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>com.h2database</groupId>
	<artifactId>h2</artifactId>
	<scope>runtime</scope>
</dependency>
```

此时使用的是 h2 内嵌数据库。

###### (2) 自定义数据源

删除 h2 的依赖，添加 **MySQL** 的依赖。

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

在配置文件 application.properties 中修改配置即可**配置数据源**。

```properties
spring.datasource.url = jdbc:mysql://localhost:3306/chapter5
spring.datasource.username = root
spring.datasource.password = 123456
#spring.datasource.driver-class-name = com.mysql.jdbc.Driver

# 使用Tomcat 自带的数据库连接池
spring.datasource.tomcat.max-idle = 10
spring.datasource.tomcat.max-active = 50
spring.datasource.tomcat.max-wait = 10000
spring.datasource.tomcat.initial-size = 5
```

如果使用 DBCP 数据源需要加依赖

```xml
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-dbcp2</artifactId>
</dependency>
```

同时配置如下

```properties
spring.datasource.url = jdbc:mysql://localhost:3306/spring_boot_chapter5
spring.datasource.username = root
spring.datasource.password = 123456
 
spring.datasource.type = org.apache.commons.dbcp2.BasicDataSource
spring.datasource.dbcp2.max-idle = 10
spring.datasource.dbcp2.max-total = 50
spring.datasource.dbcp2.max-wait-millis = 10000
spring.datasource.dbcp2.initial-size = 5
```

##### 2. 整合Druid

配置如下：

```properties
spring:
  # 数据源基本配置
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://120.79.59.125:3305/nano?characterEncoding=utf8&useSSL=true
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource  # 使用type指定数据源 不使用默认数据源

  # 数据源其他配置 需要写个配置类配置一下 不然下面的配置不会生效 因为DataSourceProperties类中不会生效
    initialSize: 5
    minIdle: 5
    maxActive: 20
    maxWait: 60000
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: SELECT 1 FROM DUAL
    testWhileIdle: true
    testOnBorrow: false
    testOnReturn: false
    poolPreparedStatements: true
    #   配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
    #filters: stat,wall,log4j  不要配置这条不然打不开不知道为啥
    maxPoolPreparedStatementPerConnectionSize: 20
    useGlobalDataSourceStat: true
    connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500

```

yml 文件中配置了上述的参数，还需要**自定义一个配置类**来进行额外的设置，不然有的配置不能生效，一般不用。

```java
// 导入druid数据源
@Configuration
public class DruidConfig {
    // 创建自己的数据源并加入到容器中	
    @ConfigurationProperties(prefix = "spring.datasource")
    @Bean
    public DataSource druid(){
       return  new DruidDataSource();
    }

    // 配置Druid的监控
    // 配置一个管理后台的Servlet
    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean bean = new ServletRegistrationBean(new StatViewServlet(), "/druid/*");
        Map<String,String> initParams = new HashMap<>();
	    // 配置后台管理登录信息
        initParams.put("loginUsername","admin");
        initParams.put("loginPassword","123456");
        // 默认就是允许所有访问
        initParams.put("allow","");
        // 配置拒绝访问IP
        initParams.put("deny","192.168.15.21");
        bean.setInitParameters(initParams);
        return bean;
    }

    /**
     * 配置一个web监控的filter
     * @return Filter
     */
    @Bean
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());
        Map<String,String> initParams = new HashMap<>();
        // 排除拦截
        initParams.put("exclusions","*.js,*.css,/druid/*");
        bean.setInitParameters(initParams);
        bean.setUrlPatterns(Arrays.asList("/*"));
        return  bean;
    }
}
```

此时登录 localhost:8080/druid/ 即可访问**管理页面**，登录账户上面配置类有设置。

#### 整合SpringData JPA

##### 1. SpringData简介

<img src="assets/image-20200713223505148.png" alt="image-20200713223505148" style="zoom:60%;" />

##### 2. 整合SpringData JPA步骤

JPA 即 **Java Persistence API**， Java 持久化 API，是定义了**对象关系映射（ORM）以及实体对象持久化的标准接口**。

JPA 维护的核心是实体，它是通过一个持久化上下文维持。持久化上下文包括三个部分：

- **对象关系映射（ORM）**
- **实体操作 API**
- **查询语言**

(1) 编写一个**实体类（bean）和数据表进行映射，并且配置好映射关系**；

```java
// 标明是一个实体类
@Entity(name = "user")
// 定义映射的表
@Table(name = "t_user")
public class User {
    // 标明主键及生成策略，递增
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id = null;

    // 定义属性和表的映射关系 有时候数据库表名与类字段名对应不上就可以用这个
    @Column(name = "user_name")
    private String userName = null;

    private String note = null;

    // 定义转换器 将枚举类转换成数据库存储的类型
    @Convert(converter = SexConverter.class)
    private SexEnum sex = null;

    // Getters and Setters
}
```

以下是**性别转换器**，用于转换性别枚举。

```java
public class SexConverter implements AttributeConverter<SexEnum, Integer>{

    // 将枚举转换为数据库列
    @Override
    public Integer convertToDatabaseColumn(SexEnum sex) {
        return sex.getId();
    }

    // 将数据库列转换为枚举
    @Override
    public SexEnum convertToEntityAttribute(Integer id) {
        return SexEnum.getEnumById(id);
    }
}
```

(2) 编写一个 **Dao 接口**来操作实体类对应的**数据表（Repository）**。Spring 的 JPA 接口设计如下。

<img src="assets/1565179077788.png" alt="1565179077788" style="zoom:60%;" />

定义 JPA接口，**不需要任何实现类**，只需要扩展这个接口即可有默认的实现。**第一个泛型参数为 User，是 DAO 的实体类，第二个泛型参数是主键的类型，传入 Long 型。**

```java
public interface JpaUserRepository extends JpaRepository<User, Long> {
}
```

(3) 基本的配置 **JpaProperties**。

```yaml
# application.yml中
spring:  
 jpa:
    hibernate:
    # 更新或者创建数据表结构 如果没有数据表就创建 有就更新 使用这种方式都不需要创建数据库 可以自动根据实体类创建数据表
      ddl-auto: update
    # 允许控制台显示SQL
    show-sql: true
```

(4) 测试类，开发控制器。

```java
@Controller
@RequestMapping("/jpa")
public class JpaController {

    // 注入JPA接口，这里不需要使用实现类
    @Autowired
    private JpaUserRepository jpaUserRepository = null;

    @RequestMapping("/getUser")
    @ResponseBody
    public User getUser(Long id) {
        // 使用JPA接口查询对象
        User user = jpaUserRepository.findById(id).get();
        return user;
    }
}
```

配置类标注实体扫描，下列的注解是非必须的，只要加了 JPA 的 starter 就会**自动扫描**。

```java
// 定义JPA接口扫描包路径
@EnableJpaRepositories(basePackages = "com.springboot.chapter5.dao")
// 定义实体Bean扫描包路径
@EntityScan(basePackages = "com.springboot.chapter5.pojo")
```

如果满足不了需求还可以使用 **@Query 注解自定义**查询语句。

```java
@Query("from user where user_name like concat('%', ?1, '%') " 
    + "and note like concat('', ?2, '%')")
public List<User> findUsers(String userName, String note);
```

##### 3. JPA命名规则

同时满足一定的命名规则的方法还可以不写任何代码实现查询，如下：

|        关键字         |          方法命名           |       sql where字句        |
| :-------------------: | :-------------------------: | :------------------------: |
|        **And**        |      findByNameAndPwd       |  where name= ? and pwd =?  |
|        **Or**         |       findByNameOrSex       |   where name= ? or sex=?   |
|     **Is,Equals**     |   findById,findByIdEquals   |        where id= ?         |
|      **Between**      |       findByIdBetween       |  where id between ? and ?  |
|     **LessThan**      |      findByIdLessThan       |        where id < ?        |
|  **LessThanEquals**   |   findByIdLessThanEquals    |       where id <= ?        |
|    **GreaterThan**    |     findByIdGreaterThan     |        where id > ?        |
| **GreaterThanEquals** |  findByIdGreaterThanEquals  |       where id > = ?       |
|       **After**       |        findByIdAfter        |        where id > ?        |
|      **Before**       |       findByIdBefore        |        where id < ?        |
|      **IsNull**       |      findByNameIsNull       |     where name is null     |
| **isNotNull,NotNull** |      findByNameNotNull      |   where name is not null   |
|       **Like**        |       findByNameLike        |     where name like ?      |
|      **NotLike**      |      findByNameNotLike      |   where name not like ?    |
|   **StartingWith**    |   findByNameStartingWith    |    where name like '?%'    |
|    **EndingWith**     |    findByNameEndingWith     |    where name like '%?'    |
|    **Containing**     |    findByNameContaining     |   where name like '%?%'    |
|      **OrderBy**      |    findByIdOrderByXDesc     | where id=? order by x desc |
|        **Not**        |        findByNameNot        |      where name <> ?       |
|        **In**         | findByIdIn(Collection<?> c) |      where id in (?)       |
|       **NotIn**       |        findByNameNot        |      where name <> ?       |
|       **True**        |        findByAaaTue         |      where aaa = true      |
|       **False**       |       findByAaaFalse        |     where aaa = false      |
|    **IgnoreCase**     |    findByNameIgnoreCase     | where UPPER(name)=UPPER(?) |
|        **top**        |         findTop100          |  top 10/where ROWNUM <=10  |

##### 4. JPA在数据库中非持久化一个字段？

假如有下面一个类：

```java
Entity(name="USER");
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "ID")
    private Long id;

    @Column(name="USER_NAME")
    private String userName;

    @Column(name="PASSWORD")
    private String password;

    private String secrect;
}
```

如果我们想让`secrect` 这个字段不被持久化，也就是不被数据库存储怎么办？可以采用下面几种方法：

```java
static String transient1; // not persistent because of static
final String transient2 = “Satish”; // not persistent because of final
transient String transient3; // not persistent because of transient
@Transient
String transient4; // not persistent because of @Transient
```

