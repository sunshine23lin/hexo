---
title: ArrayList源码分析
date: 2020-12-26 14:07:16
categories: java基础
tags: java集合
---

##  属性

```java
// 序列化版本UID
private static final long
        serialVersionUID = 8683452581122892189L;

/**
 * 默认的初始容量
 */
private static final int
        DEFAULT_CAPACITY = 10;

/**
 * 用于空实例的共享空数组实例
 * new ArrayList(0);
 */
private static final Object[]
        EMPTY_ELEMENTDATA = {};

/**
 * 用于提供默认大小的实例的共享空数组实例
 * new ArrayList();
 */
private static final Object[]
        DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 存储ArrayList元素的数组缓冲区
 * ArrayList的容量，是数组的长度
 * 
 * non-private to simplify nested class access
 */
transient Object[] elementData;

/**
 * ArrayList中元素的数量
 */
private int size;
```

##  构造方法

- 无参构造方法

  ```java
  /**
   * 无参构造方法 将elementData 赋值为
   *   DEFAULTCAPACITY_EMPTY_ELEMENTDATA
   */
  public ArrayList() {
      this.elementData =
              DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
  }
  ```

- 带初始容量构造方法

  ```java
  /**
   * 带一个初始容量参数的构造方法
   *
   * @param  initialCapacity  初始容量
   * @throws  如果初始容量非法就抛出
   *          IllegalArgumentException
   */
  public ArrayList(int initialCapacity) {
      if (initialCapacity > 0) {
          this.elementData =
                  new Object[initialCapacity];
      } else if (initialCapacity == 0) {
          this.elementData = EMPTY_ELEMENTDATA;
      } else {
          throw new IllegalArgumentException(
                  "Illegal Capacity: "+ initialCapacity);
      }
  }
  ```

- 带一个集合参数的构造方法

  ```java
  /**
   * 带一个集合参数的构造方法
   *
   * @param c 集合，代表集合中的元素会被放到list中
   * @throws 如果集合为空，抛出NullPointerException
   */
  public ArrayList(Collection<? extends E> c) {
      elementData = c.toArray();
      // 如果 size != 0
      if ((size = elementData.length) != 0) {
          // c.toArray 可能不正确的，不返回 Object[]
          // https://bugs.openjdk.java.net/browse/JDK-6260652
          if (elementData.getClass() != Object[].class)
              elementData = Arrays.copyOf(
                      elementData, size, Object[].class);
      } else {
          // size == 0
          // 将EMPTY_ELEMENTDATA 赋值给 elementData
          this.elementData = EMPTY_ELEMENTDATA;
      }
  }
  ```

  

##  扩容分析

这里以无参构造函数创建的 ArrayList 为例分析：

**add方法**

```java
/**
     * 将指定的元素追加到此列表的末尾。 
     */
   public boolean add(E e) {
    //添加元素之前，先调用ensureCapacityInternal方法
         ensureCapacityInternal(size + 1);  // Increments modCount!!
        //这里看到ArrayList添加元素的实质就相当于为数组赋值
        elementData[size++] = e;
        return true;
 }
```

**ensureCapacityInternal方法**

```java
 private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

   private static int calculateCapacity(Object[] elementData, int minCapacity) {
       // DEFAULT_CAPACITY=10,
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
```

由代码可知道，当add进第一个元素时，mincapacity是1，Math.max方法比较后，返回最大值是10

**ensureExplicitCapacity方法**

```java
 
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // 判断是否扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

我们来仔细分析一下：

- 当add进第一个元素到ArrayList时，elementData.length为0(因为第一个空的list),因为执行了ensureCapacityInternal()方法，所以minCapacity此时为10，此时minCapacity - elementData.length > 0 成立，所以会进入 grow(minCapacity) 方法。
- 当add第2个元素时，minCapacity 为2，此时e lementData.length(容量)在添加第一个元素后扩容成 10 了。此时，minCapacity - elementData.length > 0 不成立，所以不会进入 （执行）grow(minCapacity) 方法。
- 添加第3、4···到第10个元素时，依然不会执行grow方法，数组容量都为10。
- 直到添加第11个元素，minCapacity(为11)比elementData.length（为10）要大。进入grow方法进行扩容。

grow()方法

```java
    private void grow(int minCapacity) {
        // overflow-conscious code
        // oldCapacity为旧容量，newCapacity为新容量
        int oldCapacity = elementData.length;
        // 将oldCapacity 右移一位，其效果相当于oldCapacity /2
        // 我们知道位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的1.5倍，
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量，
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }  
    // 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) `hugeCapacity()` 方法来比较 minCapacity 和 MAX_ARRAY_SIZE，19        //如果minCapacity大于最大容量，则新容量则为`Integer.MAX_VALUE`，否则，新容量大小则为 MAX_ARRAY_SIZE                   MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8。
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

##  总结

ArrayList是属于动态扩容，当你未指定容量大小时，默认容量是10，每次add时候会判断当前长度+1-容量大小是否大于0。
如果是，就会进行扩容，扩容新容量=旧容量+旧容量右移1位，也就是原来的1.5倍左右，此时会进行2个大小判断，一个判断是否小于最小容量，如果小于就取最小容量为新的容量，再次判断新容量是否>inter.max.value-8,大于新的容量取integer.max.value。

**本质就是计算出新的容量实例化数组，将原有的数组内容复制到新的数组去。**