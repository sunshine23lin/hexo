---
title: bean对象
date: 2019-06-06 17:17:08
categories: Spring
tags: Java后端
---

# 一、bean对象的三种创建方式
1. 通过调用构造函数来创建bean对象(常用)
> 在默认情况下,当我们在spring的配置文件中写了一个bean标签,并提供了class属性,spring会调用默认构造函数创建对象,如果没有默认构造函数，则对象创建失败

```java
public class HelloService {
       public void hello(){
           System.out.println("hello spring");
       }
}

```
 
2. 通过静态工厂创建bean对象
> 工厂类中提供了一个静态方法,可以返回要用到bean对象

```java
ublic class StaticBeanFactory {

    /**
     * 模拟一个获取bean对象的方法
     */
    public static HelloService getBean(){
        return new HelloService();
    }
}
```

3.通过实例创建bean对象
```java
public class InstanceBeanFactory {

    public HelloService getBean(){
        return  new HelloService();
    }
}
```
- spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--
    xmlns 即 xml namespace xml 使用的命名空间
    xmlns:xsi 即 xml schema instance xml 遵守的具体规范
    xsi:schemaLocation 本文档 xml 遵守的规范 官方指定
    -->
    <!--
         属性：
              id:对象的唯一标识
              class:要创建对象的全限定类名
              factory-method:指定创建bean对象的方法,该方法可以是静态的,也可以部署、
              factory-bean : 指定创建bean对象的工厂bean的id

         bean对象的三种创建方式
         第一种:通过调用构造函数来创建bean对象 常用
                在默认情况下,当我们在spring的配置文件中写了一个Bean标签,并提供了class属性,spring就会调用默认构造函数创建对象
                如果没有默认构造函数,则对象创建失败

         第二种：通过静态工厂创建bean对象,工厂类中提供一个静态方法,可以返回要用到bean对象


    -->
     <!-- 默认构造函数 -->
    <bean id="helloService" class="com.myweb.spring.HelloService"></bean>

    <!--静态工厂创建 -->
    <bean id="staticBeanFactory" class="com.myweb.spring.StaticBeanFactory" factory-method="getBean"></bean>

    <!-- 实例工厂创建-->
    <bean id = "instanceBeanFactory" class="com.myweb.spring.InstanceBeanFactory"></bean>
    <bean id="instanceHelloBeanFactory" factory-bean="instanceBeanFactory" factory-method="getBean"  ></bean>

</beans>

```
# 二、bean对象的作用范围
> 配置的属性:bean标签的scope属性
属性的取值:
     singleton:单例的。 默认
     prototype:多例
     request:请求范围
     session:会话范围
     global-session:全局会话 范围


# 三、bean对象的生命周期

