---
author: CC-2018
layout: post
title: spring boot: 相关基础知识
date: 2017-10-13 10:18:01 +0800
categories: java
tag: java
---

* content
{:toc}

#### 可执行jar

要创建可执行的jar，我们需要将spring-boot-maven-plugin添加到我们的pom.xml中，spring-boot-maven-plugin。

当pom.xml里添加了这个插件后，运行mvn package可生成一个可执行的jar文件(用idea的mvn插件也可调用package)，如果pom继承spring-boot-starter-parent工程，mvn会寻找含main的类，否则，要添加以下配置：
```
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <mainClass>${start-class}</mainClass>
    <layout>ZIP</layout>
  </configuration>
  <executions>
    <execution>
      <goals>
        <goal>repackage</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

starter-parent工程包含以下配置：
```
Java 1.6作为默认编译器级别。
源代码UTF-8编码。
依赖关系管理，允许您省略常见依赖的标签，其默认版本继承自spring-boot-dependencies POM。
更合理的资源过滤。
更合理的插件配置（exec plugin，surefire，Git commit ID，shade）。
针对application.properties和application.yml的更合理的资源过滤，包括特定的文件（例如application-foo.properties和application-foo.yml）
最后一点：由于默认的配置文件接受Spring样式占位符（${…}），Maven过滤更改为使用 @..@ 占位符（您可以使用Maven属性resource.delimiter覆盖它）。
```

各个spring-boot版本会配置好对应的依赖版本：如[1.5.7 dependencies](https://github.com/spring-projects/spring-boot/blob/v1.5.7.RELEASE/spring-boot-dependencies/pom.xml)

Spring Boot采用一个不同的方法这样可以直接对jar进行嵌套。

查看生成jar的内容，可使用命令你想查看里面，可以使用`jar tvf target/xxx.jar`.您还应该在target目录中看到一个名为xxx.jar.original的较小文件。 这是Maven在Spring Boot重新打包之前创建的原始jar文件。

#### 启动器(Starters)

启动器是一组方便的依赖关系描述符

Spring Boot提供了一些“启动器(Starters)”，可以方便地将jar添加到类路径中。spring-boot-starter-parent就是一个特殊启动器，提供一些Maven的默认值。

可使用`mvn dependency:tree`查看各个jar之间的依赖关系。

#### Spring Beans 和 依赖注入

使用@ComponentScan搜索bean，结合@Autowired构造函数(constructor)注入效果很好。将应用程序类放在根包(root package)中构建代码，则可以使用@ComponentScan而不使用任何参数。 所有应用程序组件（@Component，@Service，@Repository，@Controller等）将自动注册为Spring Bean。

#### 常用注解
```
@RestController: 告诉Spring将生成的字符串(如json)直接返回给调用者，同修饰方法的@ResponseBody。
@RequestMapping: 提供“路由”信息。
@EnableAutoConfiguration: 根据添加的jar依赖关系来“猜(guess)”你将如何配置Spring。
    @EnableAutoConfiguration注解通常放置在您的主类上，它隐式定义了某些项目的基本“搜索包”。
@ComponentScan: 使用根包(root package)时，可不需指定basePackage属性，结合@Component，@Service，@Repository，@Controller等使用
@SpringBootApplication：相当于使用@Configuration，@EnableAutoConfiguration和@ComponentScan和他们的默认属性
@value：可用来注入外部配置
```

#### 运行

可在更目录执行 `mvn spring-boot:run`, 或使用 `java -jar target/xxx.jar`方式


#### SpringApplication

SpringApplication可用来解析应用程序参数，web环境，注册app的生命周期监听，banner的订制

SpringApplication将尝试代表您创建正确类型的ApplicationContext。 默认情况下，将使用AnnotationConfigApplicationContext或AnnotationConfigEmbeddedWebApplicationContext，具体取决于您是否正在开发Web应用程序。

用于确定“Web环境”的算法是相当简单的（基于几个类的存在）。 如果需要覆盖默认值，可以使用setWebEnvironment（boolean webEnvironment）。

也可以通过调用setApplicationContextClass() 对ApplicationContext完全控制。

####  Spring MVC

**外部配置**

Spring Boot允许您外部化您的配置，以便您可以在不同的环境中使用相同的应用程序代码。可以再resource下添加application.properties

或者使用命令行：`java -jar target/xxx.jar --server.port=8083`

RandomValuePropertySource可以用来注入随机值

**配置文件(Profiles)**
Spring 配置文件提供了将应用程序配置隔离的方法，使其仅在某些环境中可用。 任何@Component或@Configuration都可以使用@Profile进行标记，以限制其在什么时候加载：

```
@Configuration
@Profile("production")
public class ProductionConfiguration {
    // ...
}
```

#### 日志格式

依次字段为
```
日期和时间 - 毫秒精度并且容易排序。
日志级别 - ERROR, WARN, INFO, DEBUG, TRACE.
进程ID。
—分隔符来区分实际日志消息的开始。
线程名称 - 括在方括号中（可能会截断控制台输出）。
记录器名称 - 这通常是源类名（通常缩写）。
日志消息。
```

如果要将控制台上的日志输出到日志文件，则需要设置logging.file或logging.path属性（例如在application.properties中）。

日志级别有TRACE，DEBUG，INFO，WARN，ERROR，FATAL, OFF之一
```
logging.level.root=WARN
logging.level.org.springframework.web=DEBUG
logging.level.org.hibernate=ERROR
```

#### spring mvc

模板引擎

Spring Boot包括对以下模板引擎的自动配置支持：

    FreeMarker
    Groovy
    Thymeleaf
    Mustache

当您使用默认配置的模板引擎之一时，您的模板将从 src/main/resources/templates 自动获取。


CORS 支持
在Spring Boot应用程序中的controller方法使用@CrossOrigin注解的CORS配置不需要任何特定的配置。

**定制嵌入式servlet容器**

可以使用Spring Environment属性配置常见的servlet容器设置。 通常您可以在application.properties文件中定义属性。

常用服务器设置包括：
```
网络设置：侦听端口的HTTP请求（server.port），接口地址绑定到server.address等。
会话设置：会话是否持久化（server.session.persistence），会话超时（server.session.timeout），会话数据的位置（server.session.store-dir）和session-cookie配置（server.session.cookie.*）。
错误管理：错误页面的位置（server.error.path）等
SSL
HTTP压缩
```
