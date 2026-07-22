---
title: "ConcurrentHashMap源码分析和理解"
description: about concurrentHashMap code analyse
date: 2026-06-23T8:34:53+08:00
categories:
    - Java集合
    - 并发
tags:
    - Map
    - Java
    - JUC
image: screen.jpg
math: 
license: 
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

#### 第一部分：引论 - 为什么要用 `ConcurrentHashMap`？

- **问题引入**：从 `HashMap` 在多线程环境下可能导致的**死循环、数据丢失**等问题切入。
- **对比分析**：比较 `HashMap`（非线程安全）、`Hashtable`（全表锁，并发度低）和 `Collections.synchronizedMap`（同样低效）的优劣，点明 `ConcurrentHashMap` 在并发场景下的必要性。

#### 第二部分：JDK 1.7 的实现 - 分段锁机制（理解演进）

- **核心数据结构**：理解 **`Segment` 数组 + `HashEntry` 数组 + 链表**的结构。`Segment` 继承自 `ReentrantLock`，扮演锁的角色。
- **并发控制**：**写操作**时锁定对应的 `Segment`；**读操作**则通过 `volatile` 保证可见性，无需加锁。
- **关键源码**：
  - **初始化**：重点关注 `concurrencyLevel` 参数如何决定 `Segment` 数组的大小（默认16）。
  - **`put` 方法**：观察如何定位 `Segment` 并上锁。
  - **`get` 方法**：分析其**无锁化**的实现。
- **总结与思考**：分析分段锁的优点（支持默认16个线程并发写）与缺点（结构复杂， `Segment` 数量固定，扩容粒度小）。

#### 第三部分：JDK 1.8 的实现 - CAS + Synchronized（掌握精髓）

- **数据结构革新**：废弃 `Segment`，采用 **`Node` 数组 + 链表 + 红黑树** 的结构。
- **并发控制**：采用更细粒度的 **CAS（Compare-And-Swap）+ `synchronized`** 锁住桶的头节点。
- **关键源码（重点）**：
  1. **核心成员变量**：理解 `table`、`nextTable`、`baseCount`、`sizeCtl` 等关键字段的作用。
  2. **初始化**：分析 `sizeCtl` 如何控制 **延迟初始化**（首次 `put` 时才创建数组）。
  3. **`put` 方法**：这是最核心的方法。跟踪其完整流程：
     - **空桶**：通过 **`CAS`** 直接插入新节点。
     - **扩容中**：如果桶被标记为 `MOVED`，当前线程**协助扩容**（`helpTransfer`）。
     - **非空桶**：使用 **`synchronized`** 锁住桶的头节点，然后在链表或红黑树中执行插入或更新操作。
     - **链表树化**：当链表长度超过阈值（8）时，调用 `treeifyBin` 将链表转换为红黑树。
     - **扩容检查**：最后调用 `addCount` 检查是否需要扩容。
  4. **`get` 方法**：分析其**全程无锁**的高效实现。
  5. **扩容机制**：
     - **多线程协同**：多个线程可以同时参与扩容，提高效率。
     - **`transfer` 方法**：理解数据如何从旧表迁移到新表。
     - **`ForwardingNode`**：掌握扩容期间，旧桶上的这个占位标记节点的作用。
  6. **`size()` 方法**：
     - **`baseCount` + `CounterCell`**：理解其类似于 `LongAdder` 的设计思想。
     - **`sumCount()`**：分析它如何累加 `baseCount` 和所有 `CounterCell` 的值来获取总数。

#### 第四部分：总结与对比

- **JDK 1.7 vs JDK 1.8**：从**数据结构、锁的实现、锁粒度、扩容机制**等多个维度进行对比总结。
- **实战建议**：给出在不同场景下的使用建议。





## 引言

### HashMap的并发问题

单线程视角下，调用HashMap.put()元素时，放入元素后，若元素总数>=阈值时，会触发扩容，扩容则进行：创建新数组和计算新阈值、rehash(将旧数组元素重新分配到新数组)、用新数组替代旧数组。 等待下一次调用HashMap.put()方法。

单线程视角下的put和扩容都没有问题，那多线程视角呢？

在多线程视角下，put和扩容时存在并发问题。

#### 数据丢失问题

##### 桶位为空的快速插入

若线程A判断当前桶位为空，正准备插入时，线程B同样判断同一个桶位为空，抢先插入该位置。而导致线程A覆盖了线程B的数据。后执行的线程会覆盖先执行线程的节点，导致一个键值对直接丢失。

##### 插入链尾和树尾

- 两个线程在同一个桶的链表中遍历到尾部，各自创建节点并尝试接到末尾，但 `next` 指针的赋值不是原子的，可能导致其中一个节点丢失。

```java
if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
}
```

##### 大小计数（size）丢失

HashMap 的 `size` 字段维护了键值对总数，每次 `put` 成功后会执行 `++size`。该操作 **不是原子的**：

比如：线程 A 和 B 同时读取 `size = 10`，各自计算后写入 `size = 11`（实际应变为 12）。

##### 扩容时的数据丢失问题

情况：线程A检测到HashMap需要扩容，并创建新的数组，线程B在线程A扩容完成前，向HashMap的旧数组添加了一个新元素。

原因：由于线程A扩容时是按哈希桶一个个进行遍历的，线程B的数据可能添加在已遍历的哈希桶中。

结果： 从而导致扩容完成后，线程B添加的数据丢失。

### HashTable

HashTable是一个线程安全的类，它使用synchronized来锁住整张Hash表来实现线程安全，即每次锁住整张表让线程独占，相当于所有线程进行读写时都去竞争一把锁，导致效率非常低下。

另外则是若HashTable的存储的数据量较多，则进行扩容时，由于重新分配到新数组中，持有锁的时间会很长。

写（put）

```java
public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
    	// 相同Key则覆盖value
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
		// 创建新Entry加入
        addEntry(hash, key, value, index);
        return null;
    }
```

读（read）

```java
public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
```

### Collections.synchronizedMap

通过 `Collections.synchronizedMap(map)`将 `map`为入参返回一个 `SynchronizedMap`

```java
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
        return new SynchronizedMap<>(m);
    }
```

SynchronizedMap是Collections的内部静态类，其底层就相当于对传入的 map包装了一层 synchronized(mutex)

```java
private static class SynchronizedMap<K,V> implements Map<K,V>, Serializable {
        private final Map<K,V> m;
        final Object      mutex;   // 用作synchronize的互斥锁
        
        public V put(K key, V value) {
            synchronized (mutex) {return m.put(key, value);}
        }
    	public V remove(Object key) {
            synchronized (mutex) {return m.remove(key);}
        }
    	public V get(Object key) {
            synchronized (mutex) {return m.get(key);}
        }
}
```

这样导致同一时刻只能有一个线程能够读写 `map`。感觉与 `HashTable`实现的效果无异。两者都是**粗粒度锁**的全局锁，同一时刻只允许一个线程操作整个 Map。

----

## ConcurrentHashMap1.7 

数据结构







----

## ConcurrentHashMap1.8详解

数据结构

```java
transient volatile Node<K,V>[] table;
```





----

## 常见问题

### 1.7和1.8的区别？







## 参考文档

[【集合框架ConcurrentHashMap进阶】-腾讯云开发者社区-腾讯云](https://cloud.tencent.com.cn/developer/article/2561469?from=15425&frompage=seopage)

[ConcurrentHashMap详解：原理、实现与并发控制-CSDN博客](https://blog.csdn.net/Dcein/article/details/148591286)

[(8 封私信 / 80 条消息) 一文读懂Java ConcurrentHashMap原理与实现 - 知乎](https://zhuanlan.zhihu.com/p/104515829)

[ConcurrentHashMap源码解析6.TreeBin类_concurrentmap treebin waiter-CSDN博客](https://blog.csdn.net/qq_46312987/article/details/121568509)
