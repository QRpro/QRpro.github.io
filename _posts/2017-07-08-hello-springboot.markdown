---
layout:     post
title:      " Spring Boot 初探 "
subtitle:   "Spring Boot 打开新世界大门..."
date:       2017-07-08 
header-img: "img/hell-springboot-bg.jpg"
catalog:    true
author:     "Neil"
tags:
    - Spring
    - Java
---  

## 写在前面的话
　　最近在公众号推送以及很多地方总能看到许多人说公司在大力推Spring Boot，作为一个对技术敏感的骚年怎么能不心动呢，所以就花了一周时间简单了解了下，本篇内容作为spring Boot的初探，仅仅做对spring Boot不了解的人做一个科普，然后做一下简单的Demo，更深入的知识博主正在努力挖掘中…   
如果你对Spring有一定了解的话那我们直接跳过简史直接[进入正题](#springboot)

## Spring 简史
### Spring 的三个阶段
**第一阶段：xml配置**  
spring1.x时代，使用spring开发都是xml配置的bean 随着项目的扩展，需要将xml配置文件放在不同的配置文件中，需要p频繁的在java代码和xml之间切换  
**第二阶段：注解配置**  
spring2.x时代，JDK1.5支持注解，spring提供声明bean注解（@Controller，@Service等），大大减少了配置量，最终妥协为基本配置（如数据库配置）用xml，业务配置使用注解  
**第三阶段：java配置**  
从spring3.x到现在，spring 提供了java配置能力，并且spring4.0和spring Boot都推荐使用java配置。  
**举个栗子**  
java配置是通过`@Configuration`和`@Bean`来实现的。  
@Configuration声明当前类是一个配置类，相当于Spring配置的xml文件  
@Bean 注解在方法升，声明当前方法的返回值作为一个Bean  
下图就是分页插件PageHelper的Java配置方式
![img](/img/springboot/java-configuration.png)  
这就相当于在`mybatis-config.xml`中进行配置,并在Spring配置中配置`sqlSessionFactory`的时候引入配置文件`mybatis-config.xml`, 是不是感觉java配置非常便捷


## Spring Boot
　　随着现在动态语言的流行，java的开发显得格外笨重，超多配置，开发效率也得不到提升，复杂的部署流程以及三方技术集成难度大，面对上面种种问题，spirng Boot就应运而生，简单的说spring Boot就是使用默认开发配置来实现快速开发，你只需要很少的spring配置(也可以不用)，在学习任何技术过程中我们都需要了解其短板和长处。
### Spring Boot特性
#### Spring Boot优点
- 创建独立spring应用
-	嵌入tomcat，无需部署war文件
-	简化Maven配置
-	自动配置spring
-	提供生产就绪型功能，如指标，健康检查和外部配置
-	开箱即用，没有代码生产，无需配置xml

#### Spring Boot短板
-	依赖太多，随便一个Spring Boot应用都有好几十M
-	缺少服务的注册和发现等解决方案
-	缺少监控继承方案，安全管理方案
-	中文文档和资料太少

下面为Spring Boot 实例程序 这里用到的技术都是spring Boot 的starter pom(maven坐标都为spring-boot-starter-*)，当这些技术被选中后与这项技术相关的Sping的bean会被自动配置。
<span id='springboot'>
## Hello World

#### 创建项目
- 创建普通maven项目spring-boot-helloword    

#### 添加依赖  
- 添加父依赖  

```
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>1.5.4.RELEASE</version>
</parent>
```
- 添加坐标  

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

- 将编译器版本改为JDK1.8（最低支持1.6）    

```
<properties>
	<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	<java.version>1.8</java.version>
</properties>
```

　　spring Boot父级依赖，`spring-boot-starter-parent`是一个特殊的starter，用来提供Maven默认依赖，使用后可以省去version标签(福音,再也不用担心版本问题导致的各种难以排查的bug) 可以查看 maven仓库下`\org\springframework\boot\spring-boot-dependencies\1.X.X.RELEASE\ spring-boot-dependencies-1.X.X.RELEASE.pom`来查看`spring-boot-starter-web`依赖spring web项目常用jar包包含spring、springmvc、tomcat等相关jar。

#### 撰写代码  
创建两个类Controller和App，具体代码如下  
```java
//@RestController = @Controller + @ResponseBody
@RestController
public class Controller {
	
	@RequestMapping("/index")
	public String index(){
		return "Hello Spring Boot";
	}
}
```

```java
@SpringBootApplication
public class App {
	
	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}
}
```

右键Run As  -JavaApplication然后就能看到如下启动信息说明启动成功
![img](/img/springboot/start-banner.png)  
接着访问`localhost:8080/index`就能看到相应文字“Hello Spring Boot”

####  @SpringBootApplication分析  
Spring boot项目的核心注解，主要目的是开启自动配置，这个注解是 一个组合注解  
![img](/img/springboot/springbootapplication-annotation-desc.png)  
源码是这个样子的↓  

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
public @interface SpringBootApplication {}
```  
包括了`@SpringBootConfiguration`、`@EnableAutoConfiguration`(让spring Boot根据类路径中的jar包依赖为当前项目进行自动配置，如：添加了spring-boot-starter-web依赖，会自动添加Tomcat和SpringMvc等相关依赖，那么会对Tomcat、 SpringMvc等自动配置)、`@ComponentScan` 若不使用用`@SpringBootApplication` 可在入口类上直接使用上述三个注解

#### 关于启动Banner
**修改Banner**  
启动Banner如上所示、可以对启动Banner自定义可以去[这里](http://patorjk.com/software/taag/ )成特定字符串或自定义字符进行修改在`src/main/resource`下面建立`banner.txt`并写入字符即可自定义页面  
![img](/img/springboot/neilqin-banner.png)  
**关闭Banner**  
`main`方法中修改为  
```java
SpringApplication app = new SpringApplication(App.class);
app.setShowBanner(false);
app.run(args);
```
或者使用 `fluent API`   

```java
new SpringApplicationBuilder(App.class).bannerMode(Banner.Mode.OFF).run(args);
```  
 关于建造者模式可以猛戳这篇文章--[如何更优雅的撸码](http://neilqin.me/2017/06/25/how-to-write-code-gracefully/#builder_pattern)  

##Spring Boot的配置文件  
`application.properties`放置在`src/main/resources`或者类路径下的`/config`下如修改Tomcat默认端口可加入` server.port=80`, 修改默认访问路径`”/”` `server.context-path=/demo`虽不提倡使用配置，但如有特殊要求可使用`@ImportResource("classpath:xxx.xml")` 来引入配置，常规属性可使用`@Value`来取值也可以在类上使用  
**Profile配置**  
如引用不同配置以方便在测试环境和生产环境中切换可新建文件`application-dev.properties`和`application-prd.properties`在核心配置文件`application.properties`如想在测试环境使用则引入`spring.profiles.active=dev`即可，测试效果如下：
![img](/img/springboot/profile.png) 
通过切换不同的值可以打到不同的效果  
![img](/img/springboot/profile-result.png)

## 集成MyBatis
#### 测试程序
**添加依赖**  

```
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>1.1.1</version>
</dependency>
```  
**添加配置信息**  
在application.properties中添加如下配置信息   

```
spring.datasource.name=MYSQL
spring.datasource.url=jdbc:mysql:///test?useSSL=true
spring.datasource.username=root
spring.datasource.password=qinrui
spring.datasource.max-active=20  
spring.datasource.max-idle=8  
spring.datasource.min-idle=8  
spring.datasource.initial-size=10  
```

**撰写代码**  
- 编写POJO    

```java
public class Demo {
	private Long id;
	private String name;
	//get 和 set 方法
}
```
- 编写mapper  

```java
@Mapper
public interface DemoMapper {

	@Select("SELECT * FROM demo WHERE name = #{name}")
	public List<Demo> likeName(String name);
	
	@Select("SELECT * FROM demo WHERE id = #{id}")
	public Demo getDemoById(Long id);
}
```  
- 编写Service  

```java
@Service
public class DemoService {
	
	@Resource
	private DemoMapper demoMapper;
	
	public List<Demo> likeName(String name){
		return demoMapper.likeName(name);
	}
	
	public Demo getDemo(Long id){
		return demoMapper.getDemoById(id);
	}
}
```  
- 编写Controller   

```java
@RestController
public class DemoController {
	
	@Resource
	private DemoService demoService;
	
	@RequestMapping("/list")
	public List<Demo> likeName(String name){
		return demoService.likeName(name);
	}
}
``` 
启动后在地址栏输入`localhost:8080/list?name=Mary`就能看到返回的json  

#### 加入分页插件PageHelper
**添加依赖**  

```
<dependency>
	<groupId>com.github.pagehelper</groupId>
	<artifactId>pagehelper</artifactId>
	<version>4.1.0</version>
</dependency>
```
**编写配置信息**  

```java
@Configuration
public class MyBatisCongifuration {
	@Bean
	public PageHelper pageHelper(){
		PageHelper pageHelper = new PageHelper();
		Properties properties = new Properties();
		properties.setProperty("dialect", "mysql");
		properties.setProperty("offsetAsPageNum", "true");
		properties.setProperty("rowBoundsWithCont", "true");
		properties.setProperty("reasonable", "true");
		pageHelper.setProperties(properties);
		return pageHelper;
	}
}
```
将Collec中需要分页的内容进行更改  

```java
@RequestMapping("/list")
public List<Demo> likeName(String name,int pageNum,int pageSize){
	PageHelper.startPage(pageNum, pageSize);
	return demoService.likeName(name);
}
```
地址栏中输入如下即可看到分页后的信息  `http://localhost:8080/list?name=Mary&pageNum=1&pageSize=2` 

## JSP支持 
在pom.xml中加入如下依赖  

```
<!-- jstl -->
<dependency>
	<groupId>javax.servlet</groupId>
	<artifactId>jstl</artifactId>
</dependency>

<!-- tomcat支持 -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-tomcat</artifactId>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-jasper</artifactId>
	<scope>provided</scope>
</dependency>
```  
配置application.properties支持jsp   


```
spring.mvc.view.prefix=/WEB-INF/jsp/
spring.mvc.view.suffix=jsp
```
配置之后就可以正常使用并返回自己的页面了

## 运行原理 

　　前文说道`@SpringBootApplication`注解，这个注解是一个组合注解，它的核心功能是由`@EnableAutoConfiguration `提供，在`@EnableAutoConfiguration`注解的关键导入配置功能`@Import`，这里博主使用的是最新1.5.4版本，EnableAutoConfigurationImportSelector类使用了Spring Core包的SpringFactoriesLoader类的loadFactoryNamesof()方法。 SpringFactoriesLoader会查询`META-INF/spring.factories`(在spring-boot-autoconfigure-1.x.x.RELEASE.jar下)文件中包含的JAR文件。 当找到spring.factories文件后，SpringFactoriesLoader将查询配置文件命名的属性  
在这里分析一个简单的Sprong Boot内置的自动配置功能：http编码配置  
通常我们会在web.xml文件中进行如下配置  
![img](/img/springboot/web-xml-encoding.png)  
自动配置需要满足两个条件：  
-  配置CharacterEncodingFilter这个Bean;  
- 能配置encoding和forceEncoding这两个参数。  

**配置参数**  
在这里使用类型安全配置  
![img](/img/springboot/httpencoding-properties.png)  
配置Bean  
![img](/img/springboot/http-encode-autoconfiguration.png)   

跟上述mybatis分页插件配置一样，这样就实现了spring Boot的自动配置。  

## 后记
整理笔记是一件又辛苦又兴奋的事情，可以加深印象并且会发现之前很多当时不太清楚的知识点，当然知识分享是一件很有意义的事情，那关于Sping Boot的入门就暂告一段落，在深入学习之后将继续更新Spring Boot 深究。

---
-- Neil 后记于 2017.07
