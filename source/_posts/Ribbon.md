---
title: Ribbon
date: 2020-12-25 08:30:48
categories: springCloud
tags: 服务调用
---

##  概述

###  是什么？

SpringCloud Ribbon是基于Netflix Ribbon实现的一套**客户端负载均衡**的工具。是Netflix发布开源项目，主要功能提供客户端的软件**负载均衡和服务调用**。Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。就是在配置文件中列出Load Balancer(Lb)后面的所有机器，Ribbon会自动的帮助你基于某种规则(如简单轮询，随机连接等)去连接这些机器。我们很容易使用Ribbon实现自定义的负载均衡算法。

###  LB（负载均衡）

简单的来说就是将用户的请求平摊的分配到多个服务上,从而达到系统的HA(高可用)。常见的负载均衡有Nginx,LVS,硬件F5等。

**Ribbon本地负载均衡客户端 VS Nginx服务端负载均衡区别**

Nginx是服务器负载均衡，客户端所有请求都会交给Nginx,然后由nginx实现转发请求。负载均衡由服务实现的。

Ribbon本地负载均衡，在调用微服务接口时，会在注册中心上获取信息服务列表之后缓存到JVM,从而在本地实现RPC远程服务调用。

总结：**负载均衡+RestTemplate调用**

![image-20201225090457343](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201225090457343.png)

Ribbon在工作时分两步

- 先选择EurekaServer,他优先选择在同一个区域内负载比较少的server
- 再根据用户指定的策略，在从server取到服务注册列表中选择一个地址。其中Ribbon提供了多种策略: 比如轮询、随机和根据响应时间加权。

**简单demo**

核心代码

```java
@Configuration
public class ApplicationContextConfig {

    @Bean
    @LoadBalanced  //赋予RestTemplate负载均衡的能力
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }
}
```

```Java
@RestController
@Slf4j
public class OrderController {

    public static final String PAYMENT_URL = "http://CLOUD-PAYMENT-SERVICE";
    @Resource
    private RestTemplate restTemplate;
    

    @GetMapping("/consumer/payment/get/{id}")
    public CommonResult<Payment> getPayment(@PathVariable("id") Long id){
        return restTemplate.getForObject(PAYMENT_URL+"/payment/get/"+id,CommonResult.class);
    }
}
```

##  核心组件IRule

###  IRule

根据特定算法从服务列表中选取要访问的服务

- **RoundRobinRule**  ：轮询
- **RandomRule** ：随机
- **RetryRule** ：先轮询策略获取服务，如果获取指定失败则在指定时间内进行重试
- **weightedResponseTimeRule**： 对RoundRobinRule的扩展，响应速度越快实例选择权越大，越容易被选择
- **BestAvailableRule** ：先会过滤掉由于多次访问故障而处于断路器跳闸状态的服务然后选择一个并大量最小的服务
- **AvailabilityFilteringRule**：先过滤掉故障实例，在选择并发最小的实例
- **ZoneAvoidanceRule**：默认规则，复合判断server所在区域的性能和server的可用性选择器

###  使用

**配置细节**

自定义配置类不能放在@componentScan所扫描的当前包下以及子包下，否则我们自定义的配置类就会被所有Ribbon客户端所共享，达不到特殊化定制目的。

![image-20201225093627726](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201225093627726.png)

```Java
@Configuration
public class MySelfRule {
    @Bean
    public IRule myRule(){
        return new RandomRule();// 随机
    }
}
```

启动类配置

```java
@EnableEurekaClient
@SpringBootApplication
@RibbonClient(name = "CLOUD-PAYMENT-SERVICE",configuration = MySelfRule.class)
public class OrderMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class,args);
    }

}
```

**由于Ribbon目前也进入维护模式，但是生产还是有大规模的使用，还是需要学习一下。**