---
layout:     post
title:      "SpringCloud-Feign继承特性"
subtitle:   "从零开始"
date:       2018-03-24 
header-img: "img/springcloud-fegin-extends-bg.jpg"
catalog:    true
author:     "Neil"
tags:
    - SpringCloud
    - Java
---  

## 写在前面的话  
　　一直在用Dubbo最近开始接触SpringCloud项目，从零开始入手SpringCloud，SpringCloud利用Feign采用REST Api方式调用。今天主要是想说一下Feign的继承特性 伪RPC模式。在学习过程也遇到过各种坑，感谢Github，随时可以Fock一个项目来学习，在看文章之前假装你已经了解了Eureka。

## 直接开整
#### 构建基础项目
项目目录如下：
![img](/img/blogarticles/springcloud_feign/project-maven.png)  
> `springcloud-parent`package为pom负责统一管理项目 pom.xml核心部分如下  

```
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.10.RELEASE</version>
	</parent>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Edgware.BUILD-SNAPSHOT</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<repositories>
		<repository>
			<id>spring-snapshots</id>
			<name>Spring Snapshots</name>
			<url>https://repo.spring.io/libs-snapshot</url>
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</repository>
	</repositories>
	<modules>
		<module>springcloud-eureka</module>
		<module>springcloud-api</module>
		<module>springcloud-provider</module>
		<module>springcloud-consumer</module>
	</modules>
```

> `springcloud-api`为接口，pom.xml核心如下    

```
	<parent>
		<groupId>me.neil</groupId>
		<artifactId>springcloud-parent</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>
	<artifactId>springcloud-api</artifactId>
	<dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
		</dependency>
	</dependencies>
```

> `springcloud-consumer`为消费者也就是客户端，pom.xml核心如下  

```
	<parent>
		<groupId>me.neil</groupId>
		<artifactId>springcloud-parent</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>
	<artifactId>springcloud-consumer</artifactId>
	<build />
	<dependencies>
		<dependency>
			<groupId>me.neil</groupId>
			<artifactId>springcloud-api</artifactId>
			<version>0.0.1-SNAPSHOT</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-feign</artifactId>
		</dependency>
	</dependencies>
```

配置文件application.yml为 

```
spring:
  application:
    name: consumer
server:
  port: 9002
eureka:
  instance: 
    preferIpAddress: true
    instance-id: ${spring.cloud.client.ipAddress}:${server.port} 
  client:
    service-url:
      defaultZone: http://localhost:9000/eureka 
```

> `springcloud-provider`为服务提供者也就是Service，pom.xml核心如下   

```
	<parent>
		<groupId>me.neil</groupId>
		<artifactId>springcloud-parent</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>
	<artifactId>springcloud-provider</artifactId>
	<build />
	<dependencies>
		<dependency>
			<groupId>me.neil</groupId>
			<artifactId>springcloud-api</artifactId>
			<version>0.0.1-SNAPSHOT</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
	</dependencies>
```
配置文件application.yml为   

```
spring:
  application:
    name: provider
server:
  port: 9001
eureka:
  instance: 
    preferIpAddress: true
    instance-id: ${spring.cloud.client.ipAddress}:${server.port} 
  client:
    service-url:
      defaultZone: http://localhost:9000/eureka 
```

> `springcloud-eureka`为Eureka注册中心，pom.xml核心如下   

```
  <parent>
    <groupId>me.neil</groupId>
    <artifactId>springcloud-parent</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </parent>
  <artifactId>springcloud-eureka</artifactId>
  <build/>
  <dependencies>
  	<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
  	</dependency>
  </dependencies>
``` 
配置文件application.yml为 

```
spring:
  application:
    name: eureka-server
server:
  port: 9000
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```
#### Eureka 
接下来我们启动Eureka 

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerStart {
	
	public static void main(String[] args) {
		SpringApplication.run(EurekaServerStart.class, args);
	}
}
```
![img](/img/blogarticles/springcloud_feign/eureka-server-img.png)
#### API 
api模块代码如下 

```
public interface BusinessApi {
	@GetMapping("/info/{id}")
	String getInfo(@PathVariable("id")Integer id);
}
```

#### Consumer 
项目结构如下
![img](/img/blogarticles/springcloud_feign/consumer-tree.png) 

```
@FeignClient("provider")
public interface ApiHandle extends BusinessApi{}
```

```
@RestController
public class BusinessController {
	@Autowired
	private ApiHandle apiHandle;
	@GetMapping("/info/{id}")
	public String getInfo(@PathVariable("id")Integer id){
		return this.apiHandle.getInfo(id);
	}
}
```

#### Provider
项目结构如下
![img](/img/blogarticles/springcloud_feign/provider-tree.png)
```
@RestController
public class BusinessService implements BusinessApi{
	@Override
	public String getInfo(@PathVariable("id")Integer id) {
		return "查询的id为："+id;
	}
}
```

启动之后访问http://localhost:9002  可以正常返回数据
![img](/img/blogarticles/springcloud_feign/url-action.png)
#### Exception
这里扩展个全局异常处理，代码如下 
```
@RestControllerAdvice
public class GlobleException {
	
	@ExceptionHandler(NumberFormatException.class)
	@ResponseStatus(HttpStatus.NOT_FOUND)
	@ResponseBody
	public String numberFormatException(NumberFormatException e){
		return "发生异常:"+e.getMessage();
	}
}
```
因为我们接受参数是integer类型的，但是如果入参是String类型的怎么办了上面代码可以拦截指定类型的异常并按需求进行相应
![img](/img/blogarticles/springcloud_feign/exception.png)

--- 
学无止境，愿今年可以打好基础并迈入门大数据的天坑  
--Neil 后记于2018.03