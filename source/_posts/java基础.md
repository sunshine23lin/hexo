---

title: 零散java知识补漏
date: 2020-11-28 14:45:45
categories: java基础
tags: java
---

###  修饰符

>private :  在同一类可见，使用对象：变量、方法。注意不能修饰类(外部类)
>
>default: 在同一包内见，不使用任何修饰符，使用对象: 类、接口、变量、方法。
>
>protected: 对同一包内的类和所有子类可见，使用对象：变量,方法。注意：不能修饰类(外部类)
>
>public: 对所有类可见，使用对象: 类、接口、方法

| 修饰符    | 当前类 | 同包 | 子类 | 其它包 |
| --------- | ------ | ---- | ---- | ------ |
| private   | √      | ×    | ×    | ×      |
| default   | √      | √    | ×    | ×      |
| protected | √      | √    | √    | ×      |
| public    | √      | √    | √    | √      |



###  final作用

- 被final修饰的类不可以继承
- 被final修饰的方法不可被重写
- 被final修饰的常量是不可能变的
- 被final修饰的对象，对象的引用时不变的

final、finally、finalize区别

- final可以修饰类、变量、方法，修饰类表示该类不能被继承、修饰方法表示该方法不能被重写、修饰变量表 
  示该变量是一个常量不能被重新赋值
- finally一般作用在try-catch代码块中，在处理异常的时候，通常我们将一定要执行的代码方法finally代码块 
  中，表示不管是否出现异常，该代码块都会执行，一般用来存放一些关闭资源的代码。
- finalize是一个方法，属于Object类的一个方法，而Object类是所有类的父类，该方法一般由垃圾回收器来调 
  用，当我们调用System.gc() 方法的时候，由垃圾回收器调用finalize()，回收垃圾，一个对象是否可回收的 
  最后判断。

###  this关键字用法

> this是自身的一个对象，代表对象本身，可以理解为:指向对象本身的一个指针

this的用法在Java中大体可以分3种

1. 普通的直接引用,this相当于是指向当前对象本身

2. 形参与成员名字重名,用来this来区分

   ~~~java
   public Person(String name, int age) {
       this.name = name;
       this.age = age;
   }
   ~~~

   

3. 引用本类的构造函数

   ~~~java
   class Person{
       private String name;
       private int age;
       
       public Person() {
       }
    
       public Person(String name) {
           this.name = name;
       }
       public Person(String name, int age) {
           this(name);
           this.age = age;
       }
   }
   ~~~

   

###  super关键字的用法

> super可以理解为是指向自己超（父）类对象的一个指针，而这个超类指的是离自己最近的一个父类。

super也有三种用法：

1. 普通的直接引用

   与this类似，super相当于是指向当前对象的父类的引用，这样就可以用super.xxx来引用父类的成员。

2. 子类中的成员变量或方法与父类中的成员变量或方法同名时，用super进行区分

   ~~~java
   class Person{
       protected String name;
    
       public Person(String name) {
           this.name = name;
       }
    
   }
    
   class Student extends Person{
       private String name;
    
       public Student(String name, String name1) {
           super(name);
           this.name = name1;
       }
    
       public void getInfo(){
           System.out.println(this.name);      //Child
           System.out.println(super.name);     //Father
       }
    
   }
   
   public class Test {
       public static void main(String[] args) {
          Student s1 = new Student("Father","Child");
          s1.getInfo();
    
       }
   }
   ~~~

   

3. 引用父类构造函数

###  this与super的区别

- super:它引用当前对象的直接父类中的成员（用来访问直接父类中被隐藏的父类中成员数据或函数，基类与派生类中有相同成员定义时如：super.变量名 super.成员函数据名（实参）
- this：它代表当前对象名（在程序中易产生二义性之处，应使用this来指明当前对象；如果函数的形参与类中的成员数据同名，这时需用this来指明成员变量名）
- super()和this()类似,区别是，super()在子类中调用父类的构造方法，this()在本类内调用本类的其它构造方法。
- super()和this()均需放在构造方法内第一行
- 尽管可以用this调用一个构造器，但却不能调用两个
- this()和super()都指的是对象，所以，均不可以在static环境中使用。包括：static变量,static方法，static语句块。



### static 存在的主要意义及特点

> static的主要意义是在于创建独立于具体对象的域变量或者方法。 **以致于即使没有创建对象，也能使用属性和调用方法** ！
>
> 1、在该类被第一次加载的时候，就会去加载被static修饰的部分，而且只在类第一次使用时加载并进行初始化，注意这是第一次用就要初始化，后面根据需要是可以再次赋值的。
>
> 2、static变量值在类加载的时候分配空间，以后创建类对象的时候不会重新分配。赋值的话，是可以任意赋值的！
>
> 3、被static修饰的变量或者方法是优先于对象存在的，也就是说当一个类加载完毕之后，即便没有创建对象，也可以去访问。



### 多态

> 所谓多态就是指程序中定义的引用变量所指向的具体类型和通过该引用变量发出来的方法调用在编程时并不确定，而在程序运行期间才确定，即一个引用变量到底会指向哪个类的实例对象，该引用变量发出来调用到底是哪个类中实现的方法，必须由程序运行期间才能决定。因为在程序运行时才能确定具体的类，这样就不用修改源代码,就可以让引用变量绑定到各个不同实现类上，从而导致该引用调用的具体方法随之改变，即不修改程序代码就可以改变程序运行时所绑定的具体代码，让程序可以选择多个运行状态，这就是多态性。

多态实现

Java实现多态有三个必要条件：继承、重写、向上转型。

- 继承: 在多态中必须存在有继承关系的子类和父类
- 重写：子类对父类中某些方法进行重新定义，在调用这些方法时就会调用子类的方法。
- 向上转型: 在多态中需要将子类的引用赋给父类对象，只有这样该引用才能够具备技能调用父类的方法和子类的方法。

>  在多态中需要将子类的引用赋给父类对象，只有这样该引用才能够具备技能调用父类的方法和子类的方法。

###  类与接口

####  抽象类与接口对比

- 抽象类是用来捕捉子类的通用特性。接口是抽象方法的集合
- 从设计层面来说,抽象类是对类的抽象,是一种模板设计,接口是行为的抽象，是一种行为规范。

####  相同点

- 接口和抽象类不能实例化
- 都位于继承的顶端,用于被其它实现或继承
- 都包含抽象方法,其子类都必须覆写这些抽象方法

#### 不同点

|  **参数**  |                          **抽象类**                          |                           **接口**                           |
| :--------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|    声明    |                   抽象类使用abstract关键字                   |                 接口使用interface关键字声明                  |
|    实现    | 子类使用extends关键字来继承抽象类,如果子类不是抽象类的话,他需要提供抽象类中所声明的方法来实现 | 子类使用implements关键类来实现接口，它需要提供接口中所有声明的方法来实现 |
|   构造器   |                      抽象类可以有构造器                      |                       接口不能有构造器                       |
| 访问修饰符 |              抽象类中的方法可以是任意访问修饰符              | 接口方法默认修饰符是public，并且不允许定义为private或者protected |
|   多继承   |                 一个类最多只能继承一个抽象类                 |                    一个类可以实现多个接口                    |
|  字段声明  |                 抽象类的字段声明可以是任意的                 |            接口的字段默认都是 static 和 final 的             |

接口和抽象类各有优缺点，在接口和抽象类的选择上，必须遵守这样一个原则：

- 行为模型应该总是通过接口而不是抽象类定义，所以通常是优先选用接口，尽量少用抽象类。
- 选择抽象类的时候通常是如下情况：需要定义子类的行为，又要为子类提供通用的功能。

###  error和exception区别

error: 程序无法处理系统错误,编译器不做检查

> Error表示系统致命的错误,程序没法处理，一般与JVM相关的问题，如系统崩溃，内存溢出，方法调用栈溢出等。
>
> 如经常遇到的StackOverflowError、OutOfMemoryError
>
> 这种类型错误，编译器不做检查，都是系统运行过程中发生的

Exception: 程序可以处理的异常,捕获后可以处理

> Exception异常是程序能够捕获的，也可以异常处理。

![image-20201128170141808](https://jameslin23.gitee.io/2020/11/28/java基础/image-20201128170141808.png)



###  hashCode()方法作用

> 返回对象的哈希代码值（就是散列码），用来支持哈希表，例如：HashMap
>
> 实现了hashCode一定要实现equals，因为底层HashMap就是通过这2个方法判断重复对象的

###  String、StringBuffer、StingBuilder的区别

- 可变性

  String类使用final关键字修饰字符数组来保存字符串,所以String对象是不可变的。

  StringBuilder与StringBuffer都继承AbstractStringBuilder,在类中没有使用final关键字修饰，所以这两种对象都是可变的

  ```Java
  abstract class AbstractStringBuilder implements Appendable, CharSequence {
      char[] value;
      int count;
      AbstractStringBuilder() {
      }
      AbstractStringBuilder(int capacity) {
          value = new char[capacity];
      }
  ```

- 安全性

  String是不可变的，所以是线程安全的

  StringBuffer使用synchironized修饰，线程安全的

  StringBuilder线程不安全的。

- 性能

  StringBuffer源码

  ```java
    @Override
      public synchronized String toString() {
          if (toStringCache == null) {
              toStringCache = Arrays.copyOfRange(value, 0, count);
          }
          return new String(toStringCache, true);
      }
  ```

​       StringBuilder源码

       ```java
 @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
       ```

 StringBuffer的缓存有数据时，就直接在缓存区，而StringBuiler每次都是直接copy，这样StringBuffer相对StingBuiler做了一个性能上的优化，所以只有当数量足够大，StringBuffer的缓存区填补不了加锁的影响的性能时，StringBuiler才能性能上展示出它的优势。