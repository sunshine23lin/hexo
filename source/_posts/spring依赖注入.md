---
title: spring依赖注入
date: 2019-06-10 08:58:30
categories: Spring
tags: Java后端
---

# 一、Spring的依赖注入概述
1. 注入的方式:只有3种
> 第一种方式:通过构造函数
> 第二种方法:通过set方法

2. 注入内容:
> 第一类:基本类型和String类型
> 第二类:其它的bean
> 第三类:复杂类型(集合类型)

# 二、依赖注入方式-xml
1. 构造函数
```xml
  <!-- 
   涉及的标签:
   constructor-arg
   该标签是写在bean标签内部发子标
   标签的属性:
       type:指定要注入的参数在构造函数中类型
       index:指定要注入的参数在构造函数的索引位置
       name: 指定参数在构造函数的中的名称
       value:指定注入的数据内容,他只能指定基本类型数据和String类型数据
       ref:指定其他bean类型数据。写的其它bean的id
 -->

  <bean id="accountService" class="com.mydata.service.impl.AccountServiceImpl">
  <constructor-arg name="name" value="莫斯特"></constructor-arg>              
  <constructor-arg name="age" value="18"></constructor-arg>
  <constructor-arg name="birthday" value="now"></constructor-arg>
  </bean>

<bean id="now" class="java.util.Data"></bean>                                                                                                                                                                                                 

```

2. Set方法注入
```xml
   <!--
      涉及的标签:property
      标签的属性:
           name:指定的是set方法的名称。匹配的是类中set后面的部分
           value:指定注入的数据内容
           ref: 其它bean类型
    
    <bean id="accountService2" class="com.mydata.service.impl.AccountServiceImpl">
  <property name="name" value="莫斯特"></property>              
  <property name="age" value="18"></property>
  <property name="birthday" value="now"></property>
  </bean>

   -->

```

# 三、依赖注入方法-注解

```xml
<!--
用于创建对象
@Componet:
      作用:就相当于spring的xml配置文件中写了一个bean标签
      属性:
           value:用于指定bean的id。当不写时,默认值当前类名,首字母改小写
  由此注解衍生的三个注解:
     @Controller:一般用于表现层
     @Service：一般用于业务层
     @Repository:一般用于持久层
  他们的作用以及属性和@Compoent的作用是一模一一样。spring框架为我们提供更明确的语义来指定不同层
     @Autowired
     作用:自动按照类型注入。只要容器中有唯一的类型匹配,则可以直接注入成功。
     细节:当使用此注解注入时,set方法就可以省略了
     @Qualifier
     作用:在自动按照类型注入的基础上,再按照bean的id注入。在给类成员注入时,他不能够独立使用。
     属性:
         value:用于指定bean的id

     @Resource
     作用:直接按照bean的id注入
     属性:
         name:用于指定bean的id
     
     以上3个注解,都只能用于注入其他bean类型,而不能注入基本类型各String

     @value
         作用:用于注入基本类型和String类型的数据。
         属性:
              value:用于指定要注入的数据。它支持使用Spring的el表达式。
              spring的el表达式写法:
                         ${表达式}
     
     <context:property-placeholder location="" />
     
      
-->
```