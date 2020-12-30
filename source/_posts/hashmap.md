---
title: hashmap源码底层分析
date: 2020-11-27 08:15:09
categories: java基础
tags: java集合
---

#  前期知识铺垫

##  什么是哈希表

> 哈希表又叫散列表，他是一种基于快速存取的角度设计，也是一种典型的"空间换时间"的做法，数据结构可以理解为线性表,但是其中的元素不是紧密排列的,可能存在空隙。
>
> 哈希表是根据关键码值而直接进行访问的数据结构,通过把关键码值映射到表中一个位置来访问记录，以加快查找速度，这个映射函数就做散列函数。比如我们存储75个元素，但我们可能为这75个元素申请100元素空间，为了“快速存取“的目的。我们基于一种结果尽可能随机分布的固定函数h为每个元素存储位置,这样就可以避免遍历性质的线性搜索

#  一、HashMap核心分析(JDK1.7版)

##  1.1 什么是hashmap?

> hashmap是java集合类,以key-value键值对形式存储,在jdk1.7版本采用数组+链表

##  1.2 hashmap核心成员

~~~java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 默认容量16
static final int MAXIMUM_CAPACITY = 1 << 30;  // 最大容量
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 默认负载因子0.75
ransient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE; // Entry数组，核心

~~~

###  Entry数组（hashmap静态内部类）

~~~java
   static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v; // value值
            next = n;  // 指向下一个entry,形成链表
            key = k;   // key值
            hash = h;  // hash值
        } 
    }
~~~



##  1.3 hashmap put操作底层分析

~~~java
public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold); // 懒加载,第一次需要进行初始化
        }
        if (key == null)   // 如果key为null ,直接put值进去
            return putForNullKey(value);
        int hash = hash(key);   // 获取hash值
        int i = indexFor(hash, table.length); //计算下标
        // 遍历链表
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            // 如果has值和key都一致，进行覆盖，并把原来数据返回
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        // 如果找不到就新增一个entry
        addEntry(hash, key, value, i);
        return null;
    }
  // 获取hash值
  final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
   // 容量2的倍数-1的目的，就是取hashcode 后面几位，使得下标可以更加均匀分布，避免hash冲突
   static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
~~~

###  增加Entry对象(核心)

~~~java
 
void addEntry(int hash, K key, V value, int bucketIndex) {
     // 当entry长度大等于负载因子*容量大小 并且当前数组不为Null时候进行扩容
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }

// 扩容
    void resize(int newCapacity) {
        // new一个新的数组
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
         
        Entry[] newTable = new Entry[newCapacity];
        // 赋值操作
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

 void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
     // 遍历旧的数组
        for (Entry<K,V> e : table) {
            // 遍历链表
            while(null != e) {
                Entry<K,V> next = e.next;
                // 从initHashSeedAsNeeded基本不会进入这个地方进行重取hash
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                // 获取下标获取还是原来hash值
                int i = indexFor(e.hash, newCapacity);
                // 头插法插入链表,在线程不安全情况下会导致形成循环链表，下面会仔细分析
                e.next = newTable[i];  // 头插法关键--当前元素e的下一个元素 指向新数组的头部
                newTable[i] = e;    // 把当前元素加入新数组
                e = next;
            }
        }
    }

 void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex]; // 获取当前数组i的元素
        table[bucketIndex] = new Entry<>(hash, key, value, e);//new的entry对象，next指向头元素
        size++;
    }
~~~

###  头插法过程

![image-20201127113803957](https://jameslin23.gitee.io/2020/11/27/hashmap/image-20201127113803957.png)



> 此时e是"我",e.next是“你”
>
> 第一步：e.next = newTable[i]
>
> e.next等于newTable[i]，新数组还没插入数据，此时为null，即e.next=null
>
> 第二步: newTable[i] = e ,把当前元素插入数组中

![image-20201127114111684](https://jameslin23.gitee.io/2020/11/27/hashmap/image-20201127114111684.png)



> e = next; 执行下一个元素
>
> 此时e是"你",e.next是“他”
>
> 第一步：e.next = newTable[i]
>
> e.next = "我"
>
> newTable[i] = e

依次类推

![image-20201127114320867](https://jameslin23.gitee.io/2020/11/27/hashmap/image-20201127114320867.png)

头插法的在多线程情况下，会导致循环链表。当get()操作，没有找到对应key的时候，就会死循环遍历该链表

~~~java
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
~~~



###  总结

>  put过程中，hashmap会先判断数据是否需要初始化，然后判断key值是否为空，但key不为空时候，通过hash算法，得到hash值跟数组容量-1"与"运算得到下标，定位到数组中的位置,如果是链表，即遍历链表，判断hash,key是否一致，如果有相同的值，既覆盖并返回久的值，否则增加entry对象，在增加entry对象之前，根据当前数据容量size>=数组容量*负载因子并且当前数据 位置不为空的时候，进行扩容，采用new新的数组，采用头插法插入链表中，会造成循环链表，在get操作时候，如果没有找到对应key值，会造成死循环。所以在多线程环境下， 请使用ConcurrentHashMap。



##  问题

###  hashamp如果减少hash碰撞?

> hashmap在设计数组下标的时候，数组容量都是2的m次方，然后采用hash值和数组容量-1进行与运算。由于2的m次方减1，在二进制最低进制位都是1111的形式，与运算出来都是唯一性比较强的，比较散列分布的值。所以可以减少hash碰撞。



#  二、HashMap核心分析(JDK1.8版)

> hashmap在jdk1.8使用数组+链表/红黑树，来解决当链表过长，查找时间复杂度为o(n)的问题，并且使用尾插法。



##  Put操作底层分析

```java
 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
     // 判断数组是否初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            // 初始化和扩容代码集成一起
            n = (tab = resize()).length;
     // 当前数组位置为空
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 首节点，next为空
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // 判断是不是当前节点位置值是不是一致
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p; // 是把p赋值给e
            // 如果p节点是树
            else if (p instanceof TreeNode)
                // 添加插入二叉树
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 遍历链表
                for (int binCount = 0; ; ++binCount) {
                    // e = p.next
                    if ((e = p.next) == null) {
                        // 插入链表尾部
                        p.next = newNode(hash, key, value, null);
                        // 判断该节点是8，
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 转换成二叉树
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 找到对应位置
                    if (e.hash == hash &&
                        // k=e.key
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // 下一个元素
                    p = e;
                }
            }
            // 说明找到值了
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                // 返回旧的值
                return oldValue;
            }
        }
        ++modCount;
     // 扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

### 总结

> jdk1.8 当链表长度大于等8的时候，链表就会转换成二叉树，插入链表的时候是插入链表尾部。



#  三、 ConcurrentHashMap(JDK1.7版)

> 底层一个Segments数组，存储一个Segments对象，一个Segments中储存一个Entry数组，存储的每个Entry对象又是一个链表头结点。

![ConcurrentHashMap](https://jameslin23.gitee.io/2020/11/27/hashmap/ConcurrentHashMap.png)



> Segment数组的意义就是将一个大的table分割成多个小的table来进行加锁，也就是上面的提到的锁分离技术，而每一个Segment元素存储的是HashEntry数组+链表，这个和HashMap的数据存储结构一样

##  Put操作

~~~java
/**
从上Segment的继承体系可以看出，Segment实现了ReentrantLock,也就带有锁的功能
**/
static class  Segment<K,V> extends  ReentrantLock implements  Serializable {
}
 public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
     
        int hash = hash(key);
    // 通过hash值计算出下标
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            // 获取segment
            s = ensureSegment(j);
     // 调用put
        return s.put(key, hash, value, false);
 }

 final V put(K key, int hash, V value, boolean onlyIfAbsent) {
          // 获取锁
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value); 
            V oldValue;
         // 插入操作
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K,V> first = entryAt(tab, index);
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
~~~



###  总结

> 使用segment数组，解决多线程并发安全问题，但每次先要定位segment位置，效率会低一点，jdk1.8做出优化。



# 四、ConcurrentHashMap(JDK1.8版)

>  JDK1.8，使用CAS+Synchronized 提高并发效率

##  Put操作

~~~java
 final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode()); // 得到 hash 值
        int binCount = 0;  // 用于记录相应链表的长度
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                // 如果数组"空"，进行数组初始化
                tab = initTable();
            //  找该 hash 值对应的数组下标，得到第一个节点 f
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 用一次 CAS 操作将新new出来的 Node节点放入数组i下标位置
                // 如果 CAS 失败，那就是有并发操作，进到下一个循环
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                // 进入扩容，采用cas
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                // 锁住整个节点
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }

final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                // cas
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
~~~



###  总结

>  计算hash，算出数据的下标，判断首节点为Null,采用cas尝试插入,其次判断是否需要扩容，扩容采用cas，其他则使用synchronized锁住整个节点，避免多线程竞争。
>
> ConcurrentHashMap机制提高了并发效率。多线程环境下是最优选择。





