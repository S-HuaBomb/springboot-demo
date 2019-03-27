# springboot-demo
    1. 它是Spring的升级版，Spring容器能做到的事情，它都能做到，而且更简便，从配置形式上来说，SpringBoot完全抛弃了繁琐的XML文件配置方式，而是替代性地用注解方式来实现，虽然本质来说，是差不多的（类似包扫描，注解扫描，类加载之类）。 
    2. SpringBoot集成的插件更多，从而使用很多服务，都只是引入一个依赖，几个注解和Java类就可以用了，具体的参考相关手册。 
    3. 在Web应用开发这一块，之前的应用一般来说是打包成war包，再发布到相关服务器容器下（例如Tomcat），虽然SpringBoot也可以这么做，但在SpringBoot下更常见的形式是将SpringBoot应用打包成可执行jar包文件。之所以这么做，源于你可以直接将SpringBoot应用看成是一个Java Application，其Web应用可以没有webapp目录（更不用说web.xml了），它推荐使用html页面，并将其作为静态资源使用。 
**下面具体记录一下，如何在IDEA下从零开始，一步步搭建SpringBoot Web应用，这里采用的是maven作依赖管理。**

需要说明的是SpringBoot依赖的JDK版本为1.8及以上。

    1. File->new,选择maven，创建一个空项目，直接next.
    2. 在pom文件中引入SpringBoot相关依赖：
    
```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.1.RELEASE</version>
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

引入本项目中所需要的相关依赖(MySQL连接驱动 以及Spring Data JPA,thymeleaf模板引擎)：

```
        <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.39</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
            <version>1.4.0.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework.data/spring-data-jpa -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
            <version>1.5.1.RELEASE</version>
        </dependency>
        ```

在application.properties中配置MySQL数据库连接信息 这里的数据库为本地数据库test,用户名和密码改成自己的：
```
#MySQL
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/test?characterEncoding=utf8
spring.datasource.username=****
spring.datasource.password=****
```
 
在application.properties中配置Spring Data JPA 这一段的意思就是说，数据库类型为MYSQL，日志信息打印具
体执行的sql语句，表更新策略以及Java类到数据库表字段的映射规则等，具体查看网络资料：
```java
#Spring Data JPA
spring.jpa.database=MYSQL
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
# Naming strategy
spring.jpa.hibernate.naming-strategy = org.hibernate.cfg.ImprovedNamingStrategy
# stripped before adding them to the entity manager)
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.MySQL5Dialect
```

编写一个实体类User @Table标签，指定数据库中对应的表名，id配置为主键，生成策略为自动生成：
```java 
/**
 * Created by Song on 2017/2/15.
 * Model 用户
 */
@Entity
@Table(name = "tbl_user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    private String password;
}
```

基于JPA，实现DAO层（即数据库数据的增删改查操作）新建 UserRepositoty.java 接口文件，源代码如下：
```java
/**
 * Created by Song on 2017/2/15.
 * User表操作接口
 */
@Repository
public interface UserRepositoty extends JpaRepository<User,Long>{

    @Query("select t from User t where t.name = :name")
    User findByUserName(@Param("name") String name);
}
```
> 需要解释的是，Spring Data JPA 提供了很多持久层接口，例如 Repository,CrudRepositoty,PagingAndSortingRepository 以及 JpaRepository 接口。
其中 Repository 为基类，JpaRepositor y继承自 PagingAndSortingRepository 接口，两个泛型参数分别代表 Java POJO 类以及主键数据类型。我们创建自己
的数据库操作接口时，只需继承上述 JPA 提供的某个接口，即可自动继承相关数据操作方法，而不需要再次实现。例如 CrudRepositoty 提供了对增删改查操作
的实现，PagingAndSortingRepository 提供了分页查询方法的实现。另外 JPA 提供了一套命名规则例如 readBy**() 等，这些方法也只需要用户申明而由 JPA 自
动实现了。如果这仍不能满足业务需求，也可以自定义SQL查询语句，例如上述代码所示，采用 @Query 标签， 其中 ：\*语法为引用下面用 @Param 标识的变量，
需要注意的是其中 User 不是表面而是 Java POJO 类名。具体使用参考 JPA 使用手册。 
-----------------------
设计 Service 层业务代码，新建 UserService 类，其源代码如下：
```java
/**
 * Created by Song on 2017/2/15.
 * User业务逻辑
 */
@Service
public class UserService {
    @Autowired
    private UserRepositoty userRepositoty;

    public User findUserByName(String name){
        User user = null;
        try{
            user = userRepositoty.findByUserName(name);
        }catch (Exception e){}
        return user;
    }
}
```
---------------------------
设计Controller层 新建UserController.java，提供两个接口，/user/index 返回页面，/user/show返回数据。其源代码如下：
```java
/**
 * Created by Song on 2017/2/15.
 * User控制层
 */
@Controller
@RequestMapping(value = "/user")
public class UserController {
    @Autowired
    private UserService userService;

    @RequestMapping(value = "/index")
    public String index(){
        return "user/index";
    }

    @RequestMapping(value = "/show")
    @ResponseBody
    public String show(@RequestParam(value = "name")String name){
        User user = userService.findUserByName(name);
        if(null != user)
            return user.getId()+"/"+user.getName()+"/"+user.getPassword();
        else return "null";
    }
}
```
------------------
在application.properties文件中配置页面引擎。这里采用SpringMVC（SpringBoot还提供thymeleaf，freemaker等）。
这里需要配置其静态资源（js、css文件、图片文件等）路径，以及html页面文件路径，参考SpringMVC在Spring下的配置：
```java
#视图层控制
spring.mvc.view.prefix=classpath:/templates/
spring.mvc.view.suffix=.html
spring.mvc.static-path-pattern=/static/**
```
--------------------
在resource目录下新建templates以及static目录，分别用于存放html文件以及（js、css文件、图片)文件。
在templates下新建user目录，在user目录下新建index.html页面，这里就不写什么了，默认页面，通过相对路径引入js文件，js文件里只做示意，弹出一个alert()。

user/index.html:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8"/>
    <script src="../static/scripts/jquery.min.js"></script>
    <script src="../static/scripts/test.js"></script>
    <title>Title</title>

</head>
    <h1>TEST PAGE</h1>
<body>

</body>
</html>
```
```javascript
static/scripts/test.js：
$(document).ready(function (){
    alert("OK TEST");
});
```
--------------------- 
# 配置JPA 
新建一个configuration包，用于存放项目配置类。类似SSM架构下，spring需要配置Java POJO类包路径以及DAO层接口路径，
以自动扫描相关注解，这里同样需要配置这两项，不同的是Spring采取的是xml配置方式，这里用Java代码+注解方式配置。新
建一个JpaConfiguration.java类，其代码如下：
```java
/**
 * Created by Song on 2017/2/15.
 * JPA 配置类
 */
@Order(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@EnableTransactionManagement(proxyTargetClass = true)
@EnableJpaRepositories(basePackages = "com.song.repository")
@EntityScan(basePackages = "com.song.entity")
public class JpaConfiguration {
    @Bean
    PersistenceExceptionTranslationPostProcessor persistenceExceptionTranslationPostProcessor(){
        return new PersistenceExceptionTranslationPostProcessor();
    }
}
```
--------------------- 
# 配置项目启动入口 
到这一步就可以删掉（5）中官方示例给出的SampleController.java了，由于我们的工程结构已经发生了改变，我们需要
告诉SpringBoot框架去扫描哪些包从而加载对应类，所以这里重新编写main函数。新建一个Entry.java类，其代码如下
（其中@SpringBootApplication是一个复合注解，就理解为自动配置吧）：
```java
/**
 * Created by Song on 2017/2/15.
 * 项目启动入口，配置包根路径
 */
@SpringBootApplication
@ComponentScan(basePackages = "com.song")
public class Entry {
    public static void main(String[] args) throws Exception {
        SpringApplication.run(Entry.class, args);
    }
}
```
--------------------- 
运行main函数，访问` http://localhost:8080/user/index)`会显示测试页面，并弹出 alert(),
访问 ` http://localhost:8080/user/show?name=**`(数据表里存在的数据)会显示user信息。
