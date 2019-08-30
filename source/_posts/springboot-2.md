---
title: Springboot"配置文件"
date: 2019-08-07 14:41:03
categories: SpringBoot
tags: Java后端
---

### 一、配置文件
SpringBoot使用一个全局的配置文件,配置文件名是固定的;
- application.properties
- application.yml

配置文件的作用:
> 修改SpringBoot自动配置的默认值;SpringBoot在底层都给我们自动配置好

YAML: 以数据为中心,比json、xml等更适合做配置文件
YAML: 配置例子

> server:
>   port: 8081

XML:
```xml
   <server> 
        <port>8081</port> 
   </server>
```
1、YAML语法
YAML语法：
1. 基本语法
K: (空格) V:表示一对键值对(空格必须有)
以空格的缩进来控制层级关系;只要是左对齐的一列数据,都是同一个层级

> server:
>     port: 8081
>     path: /hello

2. 值的写法
K:v:字面直接来写
字符串默认不用加上单引号或者双引号
"":双引号；不会转义字符串里面的特殊字符;特殊字符会作为本身想表示的意思
'':单引号;会转义特殊字符,特殊字符最终只是一个普通的字符串数据

对象、Map(属性和值)(键值对)
> fridends:
>          lastName: zhangsan
>          age: 20

行内写法
> friends: {lastName: zhangsan,age: 18}

2、 配置文件值注入

```
person:
    lastName: hello
    age: 18
    boss: false
    birth: 2017/12/12
    maps: {k1: v1,k2 : 12}
    lists:
      - lisi
      - zhaoliu
    dog:
      name: 小狗
      age: 12

```

person类
```java

/**
 * 将配置文件中配置的每一个属性的值，映射到这个组件中 * @ConfigurationProperties：告诉SpringBoot将本类中的所有属性和配置文件中相关的配置进行绑定； 
 * * prefix = "person"：配置文件中哪个下面的所有属性进行一一映射 
 * ** 只有这个组件是容器中的组件，才能容器提供的@ConfigurationProperties功能；
 */
@Component
@ConfigurationProperties(prefix = "person")
public class Person {

    private String lastName;
    private Integer age;
    private Boolean boos;
    private Date birth;
    private Map<String,Object> maps;
    private List<Object> list;
    private Dog dog;

```

3、配置文件站位符
占位符获取之前配置的值,如果没有可以是用:指定默认值

```
person.last‐name=张三${random.uuid} 
person.age=${random.int} 
person.birth=2017/12/15
person.boss=false 
person.maps.k1=v1 
person.maps.k2=14 
person.lists=a,b,c 
person.dog.name=${person.hello:hello}_dog 
person.dog.age=15

```

4、Profile
1. 多Profile文件
> 我们在主配置文件编写的时候,文件名可以是application-{profile}.properties/yml
默认使用application.properties的配置
2. yml支持多文档块方式
```
spring:
  profiles:
    active: dev
---

server:
  port: 8081
spring:
  profiles: dev

---
server:
  port: 8082
spring:
  profiles: prod

---


```

5、激活指定profile
1. 在配置文件中指定spring.profile.active=dev
2. 命令行:
> java -jar spring-boot-02-config-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev；

3. 虚拟机参数
> -Dspring.profiles.active=dev

6、配置文件加载位置
springboot 启动会扫描以下位置的application.properties或者application.yml文件作为Springboot的默认配置文件
-file:./config/
–file:./
-classpath:/config/
-classpath:/
优先级由高到底,高优先级的配置会覆盖低优先级的配置
SpringBoot会从这四个位置全部加载主配置文件;互补配置

spring.config.location来改变默认的配置文件位置
项目打包好以后,我们可以使用命令行参数的形式,启动项目的时候来指定配置文件的新位置;
指定配置文件和默认加载这些配置文件共同起作用形成互补配置
> java -jar spring-boot-02-config-02-0.0.1-SNAPSHOT.jar --spring.config.location=G:/application.properties

7、自动配置原理
1. 自动配置原理:
1) SpringBoot启动的时候加载主配置类,开启了自动配置功能@EnableAutoConfiguration
2) @EnableAutoConfiguration作用:
- 利用EnableAutoConfiguationImportSelector给容器中导入一些组件
- 可以查看selectImports()方法的内容;
- List configurations = getCandidateConfigurations(annotationMetadata, attributes);获取候选的配置

> SpringFactoriesLoader.loadFactoryNames() 扫描所有jar包类路径下 META‐INF/spring.factories 把扫描到的这些文件的内容包装成properties对象 从properties中获取到EnableAutoConfiguration.class类（类名）对应的值，然后把他们添加在容器 中

精髓:
>1)、SpringBoot启动会加载大量的自动配置类
2)、我们看我们需要的功能有没有SpringBoot默认写好的自动配置类
3)、我们再来看这个自动配置类中到底配置了哪些组件；（只要我们要用的组件有，我们就不需要再来配置了）
4)、给容器中自动配置类添加组件的时候，会从properties类中获取某些属性。我们就可以在配置文件中指定这 些属性的值；

2. 细节
>@Conditional派生注解（Spring注解版原生的@Conditional作用）