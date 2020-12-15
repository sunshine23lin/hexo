---
title: java设计模式
date: 2020-11-27 17:21:43
categories: java基础
tags: java设计模式
---

#  一、单例模式

> 手写单例步骤
>
> 1、私有构造函数,外部不可new对象
>
> 2、实例化对象
>
> 3、提供外部访问接口

##  单例模式类型

###  1. 饿汉式

~~~java
public class Hungry {

    private Hungry(){}
    // 类初始化new对象
    private final static Hungry HUNGRY = new Hungry();
    public static Hungry getInstance(){
        return HUNGRY;
    }
}
~~~



###  2.懒汉式

1. 基本写法(线程不安全)

   ~~~java
   // 线程不安全
   public class LazyDemo1 {
       private LazyDemo1(){ }
       private static LazyDemo1 lazyDemo1;
       public static LazyDemo1 getInstance(){
           if (lazyDemo1==null){
               lazyDemo1 = new LazyDemo1();
           }
           return lazyDemo1;
       }
   }
   ~~~

   

2. 使用Synchronized同步（效率低）

   ~~~java
   public class LazyDemo2 {
       private static LazyDemo2 lazyDemo2;
       private LazyDemo2(){
       }
   
       public static synchronized LazyDemo2 getInstance(){
           if (lazyDemo2==null){
               lazyDemo2 = new LazyDemo2();
           }
           return lazyDemo2;
       }
   }
   ~~~

   

3. 双重检测

   ```java
   // 会被反射和反序列化攻击破坏
   public class LazyDemo3 {
       private static volatile LazyDemo3 lazyDemo3;
       private LazyDemo3(){}
       public static LazyDemo3 getInstance(){
           if (lazyDemo3==null){
               synchronized (LazyDemo3.class){
                   if (lazyDemo3==null){
                       lazyDemo3 = new LazyDemo3();
                   }
               }
           }
           return lazyDemo3;
       }
   }
   ```

   

4. 静态内部类

   ~~~java
   // 静态内部类方式在 Singleton 类被装载时并不会立即实例化，而是在需要实例化时，调用 getInstance 方法，才会装载 SingletonInstance 类，从而完成 Singleton 的实例化
   // 类的静态属性只会在第一次加载类的时候初始化，所以在这里，JVM 帮助我们保证了线程的安全性，在类进行初始化时，别的线程是无法进入的
   public class LazyDemo4 {
       private static  LazyDemo4 lazyDemo4;
       private LazyDemo4(){}
       private static class SingletonInstance{
           private static final LazyDemo4 INSTANCE = new LazyDemo4();
       }
       public static LazyDemo4 getInstance(){
           return SingletonInstance.INSTANCE;
       }
   }
   ~~~

   

5. 枚举

   ```java
   // 被《Effective Java》称为最佳的单例实现模式，因为最简单，而且不会被反射和反序列化攻击破坏。
   enum  EnumDemo {
       INSTANCE;
   
       public void doSomething() {
           System.out.println("doSomething");
   
       }
   
       public static void main(String[] args) {
           EnumDemo.INSTANCE.doSomething();
       }
   }
   ```

   