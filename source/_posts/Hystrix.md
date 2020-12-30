---
title: Hystrix
date: 2020-12-25 10:43:36
categories: springCloud
tags: 服务降级
---

##   概述

Hystrix是一个用于分布式系统的延迟和容错的开源库，在分布式系统里，许多依赖不可避免的会调用失败，比如超时，异常等。Hystrix能够保证在一个依赖出问题的情况下，不会导致整体服务失败，避免级联故障。以提高分布式系统的弹性。

“断路器”本身是在一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控(类似熔断保险丝)，像调用方返回一个符合预期的，可处理的备选响应（FallBack）,而不是长时间的等待或者抛出调用方无法处理异常。避免服务调用方的线程不会被长时间，不必要的占用，从而避免故障在分布式系统中蔓延，乃至雪崩。

## 服务降级

服务器忙，请稍后再试，不让客户等待并立刻返回一个友好提示，fallback。

以下情况会触发降级

- **程序运行异常**
- **超时**
- **服务熔断触发服务降级**
- **线程池/信号量打满**

**Demo**

服务A--当有很多个客户端请求时，该服务接口被困死，因为tomcat线程里面的工作线程已经被挤占完毕

服务B--调用A服务客户端访问响应缓慢，转圈圈

A服务限流配置`@HystrixCommand`

```Java
  // 失败
    @HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler",commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "3000") })
    public String paymentInfo_TimeOut(Integer id){
        int  timeNumber = 3;
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "线程池："+Thread.currentThread().getName()+"   paymentInfo_TimeOut,id：  "+id+"\t"+"呜呜呜"+" 耗时(秒)"+timeNumber;
    }

    //兜底方法
    public String paymentInfo_TimeOutHandler(Integer id){
        return "线程池："+Thread.currentThread().getName()+"   系统繁忙, 请稍候再试  ,id：  "+id+"\t"+"哭了哇呜";
    }
```

启动配置增加注解`@EnableCircuitBreaker`

```Java
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker
public class PaymentHystrixMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentHystrixMain8001.class,args);
    }
}
```

B服务设置服务降级

```Java
@GetMapping("/consumer/payment/hystrix/timeout/{id}")
@HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod",commandProperties = {
@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "1500")  //3秒钟以内就是正常的业务逻辑
})
public String paymentInfo_TimeOut(@PathVariable("id") Integer id){
String result = paymentHystrixService.paymentInfo_TimeOut(id);
return result;
}

//兜底方法
public String paymentTimeOutFallbackMethod(@PathVariable("id") Integer id){
return "我是消费者80，对付支付系统繁忙请10秒钟后再试或者自己运行出错请检查自己,(┬＿┬)";
}

```

主启动类

```java
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
public class PaymentHystrixMain80 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentHystrixMain80.class,args);
    }
}
```

**存在问题**

- 每个controller配置一个
- 和业务逻辑混合在一起

**解决方案**

重新新建一个类（PaymentFallbackService）实现该接口，统一为feign接口里面的方法进行异常处理

```java
@Service
public class PaymentFallbackService implements PaymentHystrixService {
    @Override
    public String paymentInfo_OK(Integer id) {
        return "-----PaymentFallbackService fall back-paymentInfo_OK , (┬＿┬)";
    }

    @Override
    public String paymentInfo_TimeOut(Integer id) {
        return "-----PaymentFallbackService fall back-paymentInfo_TimeOut , (┬＿┬)";
    }
}
```

feign接口

```java
@Service//,fallback = PaymentFallbackService.class
@FeignClient(value = "CLOUD-PROVIDER-HYSTRIX-PAYMENT",fallback = PaymentFallbackService.class)
public interface PaymentHystrixService {
    @GetMapping("/payment/hystrix/ok/{id}")
    public String paymentInfo_OK(@PathVariable("id") Integer id);

    @GetMapping("/payment/hystrix/timeout/{id}")
    public String paymentInfo_TimeOut(@PathVariable("id") Integer id);
}

```

yml配置

```yaml
feign:
  hystrix:
    enabled: true #如果处理自身的容错就开启。开启方式与生产端不一样。
```



## 服务熔断

类比保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示

**熔断机制**

熔断机制是应对雪崩的一种微服务链路保护机制。当某个服务出错不可用或者响应时间太长了时，会进行服务的降级，进而熔断该节点微服务的调用，快速返回错误的响应信息。

当检测到该节点微服务调用响应正常后，恢复链路调用

在spring cloud框架中，熔断机制通过Hystrix实现，Hystrix会监控微服务调用的状况，当失败的调用到一定阈值，缺省5秒内20调用失败，就会启动熔断机制。

**服务层配置**

```java
//服务熔断
@HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
@HystrixProperty(name = "circuitBreaker.enabled",value = "true"),  //是否开启断路器
@HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),   //请求次数
@HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"),  //时间范围
@HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60"), //失败率达到多少后跳闸
})
public String paymentCircuitBreaker(@PathVariable("id") Integer id){
if (id 0){
throw new RuntimeException("*****id 不能负数");
    }
String serialNumber = IdUtil.simpleUUID();

return Thread.currentThread().getName()+"\t"+"调用成功,流水号："+serialNumber;
}
public String paymentCircuitBreaker_fallback(@PathVariable("id") Integer id){
return "id 不能负数，请稍候再试,(┬＿┬)/~~     id: " +id;
}
```

**熔断状态**

- 熔断打开
- 熔断关闭
- 熔断半开



## 服务限流

秒杀高并发等操作，严禁一窝蜂的过来拥挤，大家排队，一秒钟N个，有序进行



##  服务监控hystrixDashboard

