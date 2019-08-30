---
title: netty-"Protobuf"
date: 2019-08-02 18:06:32
categories: Netty
tags:
---

##  一、protobuf简单入门
1. 简介
> 它是一种轻便高效的数据格式,平台无关、语言无关、可扩展、可用于通讯协议和数据存储等领域

2. 优点

- 平台无关，语言无关，可扩展
- 提供了友好的动态库，使用简单；
- 解析速度快，比对应的XML快约20-100倍；
- 序列化数据非常简洁、紧凑，与XML相比，其序列化之后的数据量约为1/3到1/10。

3. 简单例子

pom内容
```xml
<dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.7.1</version>
</dependency>

```

编写Student.proto
```proto
syntax= "proto2";
//使用方式：gradle generateProto (不能再用以前那种方式，因为那个只会生成message，不会生成service)
//这个就相当于 在 右边gradle的other中的插件generateProto，使用后生成的代码放在build/generated中
//生成完代码记得拷贝到java目录中
package com.protobuf.proto;


option optimize_for = SPEED;//加快解析速度，详细可以去官网查加快解析速度，不写默认是这个
option java_package="com.tongda.protobuf";
option java_outer_classname="StudentInfo";

message Student{
    required string name=1;
    optional int32 id=2;
    optional string address=3;
}

```
执行命令

> protoc.exe --java_out

4. 测试类

```java
   
import com.google.protobuf.InvalidProtocolBufferException;

public class ProtoBufTest {
    public static void main(String[] args) throws InvalidProtocolBufferException {
        StudentInfo.Student student = StudentInfo.Student.newBuilder()
                .setName("张三")
                .setId(12121)
                .setAddress("潮州").build();

                  byte[]  studnet2ByteArray = student.toByteArray();
                  StudentInfo.Student student1 = StudentInfo.Student.parseFrom(studnet2ByteArray);
                  System.out.println(student1.getName());
                  System.out.println(student1.getAddress());

    }
}


```