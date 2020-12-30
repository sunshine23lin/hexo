---
title: OpenFeign
date: 2020-12-25 09:42:40
categories: springCloud
tags: 服务调用
---

##  概述

###  Feign

Feign是一个声明式的web服务客户端，让编写web服务客户端变得非常容易，只需要创建一个接口并在接口上添加注解即可

Feign使得Java Http客户端变得更容易。Ribbon+RestTemplate时，利用RestTemplate对http请求的封装处理，形成了一套模板化的调用方法。但实际开发的调用可能不止一处。往往一个接口会被多处调用，所以通常都会针对每个微服务自行封装一些客户端类来包装。Feign在基础上做了进一步封装。由他来帮助我们定义和实现依赖服务接口的定义。在Feign的实现下，我们只需要**创建一个接口并使用注解的方式来配置它**，即可完成对服务提供方 的接口绑定，简化了使用Ribbon,自动封装服务调用客户端的开发量。

###  Feign & OpenFeign

- Feign是Springcloud组件中的一个轻量级Restful的HTTP服务客户端，Feign内置了Ribbon，用来做客户端负载均衡，去调用服务注册中心的服务。Feign的使用方式是：使用Feign的注解定义接口，调用这个接口，就可以调用服务注册中心的服务

  ```xml
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-feign</artifactId>
          </dependency>
  ```

- OpenFeign是springcloud在Feign的基础上支持了SpringMVC的注解，如@RequestMapping等等。OpenFeign的@FeignClient可以解析SpringMVC的@RequestMapping注解下的接口，并通过动态代理的方式产生实现类，实现类中做负载均衡并调用其他服务。

  ```xml
           <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-openfeign</artifactId>
          </dependency>
  ```

  

###  配置

Feign接口实现

```java
@FeignClient(value="content") // 服务名
@RequestMapping("/content")
public interface ContentFeign {
    /***
     * 根据分类ID查询所有广告
     */

    @GetMapping(value = "/list/category/{id}")
    Result<List<Content>> findByCategory(@PathVariable(name = "id") Long id);
}
```

启动类扫描对应接口包

```Java
@EnableFeignClients(basePackages= {"com.haigou.content.feign"})
```

###  超时控制

OpenFeign默认等待一秒钟，超时后报错

可以在YML文件开启OpenFeign客户端超时控制

```yaml
ribbon:
	ReadTimeout:  5000
	ConnectTimeout: 5000
```

###  日志打印

Feign提供了日志打印功能，对Feign接口调用情况进行控制和输出。

**配置Bean**

```Java
import feign.Logger;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignConfig {

@Bean
Logger.Level feignLoggerLevel(){
return Logger.Level.FULL;
    }
}
```

**yml文件开启**(xxx对应feign接口路径)

```yaml
logging:
  level:
    xxx.xxx: debug
```



