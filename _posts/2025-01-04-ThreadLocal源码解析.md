---
title: ThreadLocal源码解析
date: 2025-01-04 18:32:00 +0800
categories: [JUC, ThreadLocal]
tags: [ThreadLocal]

---





本文基于**JDK17**版本进行源码剖析



## 1、ThreadLocal简介

```
This class provides thread-local variables.
These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. 
ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e. g., a user ID or Transaction ID).

--- 译文：
此类提供线程局部变量。
这些变量与它们的普通对应项不同，因为访问一个变量（通过其 get 或 set 方法）的每个线程都有其自己的、独立初始化的变量副本。
ThreadLocal 实例通常是类中的私有静态字段，希望将状态与线程（例如，用户 ID 或事务 ID）相关联。
```

上文是源码中对ThreadLocal类作出的描述，其关键词为提供 **线程局部变量**，所谓线程局部变量，即可理解为每个线程内部专属变量。
线程局部变量通常在线程并发的情景中使用，即ThreadLocal的应用场景为多线程并发环境。





## 2、与常见并发方案的不同（Synchronized）

synchronized与ThreadLocal都是用于并发场景下的资源控制，二者的区别如下

- ThreadLocal采用 **以空间换时间** 的方式，为每一个线程提供了各自单独的变量；因此需要多开辟属于变量的内存，但却可以并行执行。
- synchronized采用 **以时间换空间** 的方式，多个线程之间共享同一个变量；因此需要加锁以避免访问冲突问题。

下面以两个不同案例，展示二者的区别和应用场景：

### 2.1、synchronized实现

> 三位销售共同售卖一定数量的产品，售罄截至。

```java
public class Sale {

    private static Integer totalSales = 10;

    public static void main(String[] args) {

            Runnable runnable = () -> {
                while (true) {
                    // 只要还存在剩余产品，则一直售卖
                    synchronized (Sale.class) {
                        if (totalSales > 0) {
                            System.out.println(Thread.currentThread().getName() + "正在售卖倒数第" + totalSales + "张票");
                            --totalSales;
                        } else {
                            // 产品售罄后停止循环（售卖）
                            break;
                        }

                        // 模拟售卖的时间间隔
                        try {
                            Thread.sleep(50);
                        } catch (InterruptedException e) {
                            throw new RuntimeException(e);
                        }
                    }
                }
            };

            // 启动三个销售
        Thread thread1 = new Thread(runnable, "销售1号");
        Thread thread2 = new Thread(runnable, "销售2号");
        Thread thread3 = new Thread(runnable, "销售3号");

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

可以看到，在这种多个线程共同操作（共享）一个资源的前提下，使用synchronized关键词为关键的代码流程添加锁，使得同一时刻只有一个线程持有该资源，即可避免冲突问题。





### 2.2、ThreadLocal实现

> 统计三位销售各自售出的产品数量

```java
public class SaleCount {

    private static final ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {

        Runnable runnable = () -> {
            // 每位销售售卖三次，每次售卖10件以内产品
            for (int i = 0; i < 3; i++) {
                int sales = (int) (Math.random() * 10 + 1);
                threadLocal.set(sales + threadLocal.get());
                System.out.println(Thread.currentThread().getName() + "卖出了" + sales + "件产品");

                try {
                    // 模拟售卖的时间间隔
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            // 统计三次共售出多少件产品
            System.out.println(Thread.currentThread().getName() + "总共卖出了" + threadLocal.get() + "件产品");
        };

        // 启动三个销售
        Thread thread1 = new Thread(runnable, "销售1号");
        Thread thread2 = new Thread(runnable, "销售2号");
        Thread thread3 = new Thread(runnable, "销售3号");

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

相对synchronized，threadlocal则更适合处理每个线程都有独属的变量的场景。





## 3、源码结构

ThreadLocal源码结构如下图，在当前的设计方案中，存在两个重点的**静态内部类**， 即ThreadLocalMap和Entry类，这里简单提下内部类的作用：

- 实现多重继承
- 方便函数回调
- 隐藏类中细节

普通的内部类即存在以上三种作用，静态内部类则稍特殊，其是完全独立于外部类的存在，因此作用仅限于**隐藏类中细节**，
在这里，ThreadLocalMap和Entry都是ThreadLocal的功能的实现方式，因此完全没有必要对外显示，所以设计为静态内部类比较合适。

![image-20250105115551480](https://s2.loli.net/2025/01/05/1vR4fsXdDw7LKoW.png) 



由上图及案例代码可知，ThreadLocal源码设计有以下特点

- 每个Thread（线程）实例中都维护了独立的ThreadLocalMap，用于存储当前线程的局部变量，一定程度上避免了资源访问冲突问题。
- ThreadLocalMap是一个自定义的Hash表，提供了一系列API来实现功能，内部使用Entry实例（键值对）进行存储。
- ThreadLocal在提供功能入口的同时，作为键存入entry实例中。



### 3.1、Entry静态内部类

在源码结构中，需要重点分析以下**Entry** 这个静态内部类，其源码如下

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

可以看到，该类继承了WeakReference&lt;ThreadLocal&gt;并指定了泛型, 那么在Entry实例中，传入的key即ThreadLocal实例就具有弱引用的特性，这里简单说一下强引用和弱引用的区别

- 强引用是最为普遍的引用。如果一个实例具有强引用，垃圾回收器就**不会回收**该实例， 除非手动显式地设置强引用为**null**。

  ```java
  // obj 是一个强引用
  Object obj = new Object(); 
  ```

- 弱引用则多用于缓存、监听器等内存敏感的场景，因为**只要垃圾回收器发现**具有弱引用的实例，就会回收其内存。

  ```java
  WeakReference<Object> weakRef = new WeakReference<>(new Object());
  // 获取弱引用指向的对象
  Object obj = weakRef.get(); 
  ```

在ThreadLocal源码中，当我们传入ThreadLocal对象进行Entry实例的实例化时，Entry实例就会根据传入的ThreadLocal实例创建并持有对应的**弱引用**（作为key），这个弱引用即指向存储于堆中的ThreadLocal实例。
需要注意的是：**即使每次传入的ThreadLocal实例相同，最终创建并持有的弱引用也是不同的**。

与之相对的，Entry实例进行实例化后，就会持有一个**强引用**作为value，该强引用指向了value在内存中的地址。





## 4、ThreadLocal源码分析（功能入口）

ThreadLocal中的关键代码为以下三个函数

- set(T): void
- get(): T
- remove(): void

### 4.1、set(T)函数分析

```java
    public void set(T value) {
        // 获取当前线程实例
        Thread t = Thread.currentThread();
        // 根据线程实例获取其内部threadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 存储键值对，this即为threadlocal实例
            map.set(this, value);
        } else {
            // 实例化threadLocalMap，并存储键值对
            createMap(t, value);
        }
    }
```



### 4.2、get()函数分析

```java
    public T get() {
        // 获取当前线程实例
        Thread t = Thread.currentThread();
        // 根据线程实例获取其内部threadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 传入key，即threadlocal实例，取出对应的entry实例
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 如果threadLocalMap尚未实例化，则创建map后返回默认值（默认为null）
        return setInitialValue();
    }
```



### 4.3、remove()函数分析

```java
     public void remove() {
         // 根据当前线程的实例，取出对应的threadlocalMap
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null) {
             m.remove(this);
         }
     }
```







## 5、ThreadLocalMap源码分析（功能实现）

ThreadLocal中的函数只作为功能入口，具体代码的实现则是通过ThreadLocalMap实现，即通过ThreadLocalMap的实例调用对应的函数。由以上代码同样可以看到其三个关键函数

- map.set(this, value);
- ThreadLocalMap.Entry e = map.getEntry(this);
- m.remove(this);

### 5.1、set()函数分析

由set()函数的代码流程看，需要认识到threadLocal源码设计的重要思想：

- entry[]数组作为存储多个键值对的容器
- 通过固定的算法获取键值对的存储位置
- 如果计算出的存储位置已有键值对存在，则依次向后遍历并判断是否符合存储条件
- **由计算出的存储位置开始 到不断向后遍历的位置，之间不允许存在null值，出现null值则一定会填充当前键值对，保证hash表的连续性**

接下来在查看其他源码的过程中，需要谨记以上思想。

```java
// 以ThreadLocal实例作为key，用户传入的值为value        
private void set(ThreadLocal<?> key, Object value) {
			// table，为ThreadLocalMap内部定义的Entry[],用于存储用户创建的多个entry实例（键值对）
            Entry[] tab = table;
    		// 获取容器的长度
            int len = tab.length;
    		// 重点！计算当前键值对存储的索引位置
            int i = key.threadLocalHashCode & (len-1);
			
    		// 从索引i开始，依次向后遍历entry实例，直到遇到null值
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                // 如果entry实例持有的引用指向key，即当前threadlocal实例，则进行赋值操作
                if (e.refersTo(key)) {
                    e.value = value;
                    return;
                }
                // 如果entry实例持有的引用指向null，代表threadlocal实例已被回收内存，则执行清理无效槽位的操作，同时进行键值对的存储，具体看下文中对replaceStaleEntry()函数的分析。
                if (e.refersTo(null)) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
    		
    		// 当前索引位置暂无数据（null），则直接存储
            tab[i] = new Entry(key, value);
    		// size，为ThreadLocalMap中定义的变量，用于存储变量table，即entry[]中的entry数量
            int sz = ++size;
    
    		// 清理无效槽位失败并且entry实例的数量 >= 阈值（threshold），则进行扩容操作
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

#### 5.1.1、内部replaceStaleEntry()分析

```java
// 传入键值对，以进行存储操作；同时传入遍历到的无效槽位        
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    		// table，为ThreadLocalMap内部定义的Entry[],用于存储用户创建的多个entry实例（键值对）
            Entry[] tab = table;
    		// 获取容器的长度
            int len = tab.length;
            Entry e;

            // 单独记录要删除的槽位
            int slotToExpunge = staleSlot;
    		// 从传入的无效槽位之前开始，依次向前遍历entry实例，直到遇到null值
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                // 如果当前entry实例持有的引用指向null，则代表threadlocal已被回收内存，更新要删除的槽位信息
                if (e.refersTo(null))
                    slotToExpunge = i;

            // 从传入的无效槽位之后开始，依次向后遍历entry实例，直到遇到null值
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                // 如果entry实例持有的引用指向key，即当前threadlocal实例，则进行赋值操作。注意！此时数据位于向后遍历到的索引i的位置上
                if (e.refersTo(key)) {
                    e.value = value;
					// 完成赋值操作后，将当前槽位与传入的无效槽位中的值进行替换操作。即将值向前移动，保证有效的值永远位于较前的位置
                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // 如果单独记录的要删除的槽位信息和传入的槽位信息相同，也就是说之前依次向前遍历并没有发现被回收的无效引用
                    if (slotToExpunge == staleSlot)
                        // 更新单独记录的要删除的槽位信息
                        slotToExpunge = i;
                    // 清理无效槽位
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    // 结束当前程序
                    return;
                }

                // 如果entry实例持有的引用指向null，代表threadlocal实例已被回收内存
                // 单独记录的要删除的槽位信息和传入的槽位信息相同，也就是说之前依次向前遍历并没有发现被回收的无效引用
                // 在以上二个条件都满足的情况下，更新单独记录的要删除的槽位信息
                if (e.refersTo(null) && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // 如果向后遍历的entry实例为null，则代表之后的空间仍未使用，将传入的键值对覆盖当前槽位
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // 如果单独记录的要删除的槽位信息与传入的槽位信息不同，则代表进入过之前向前或向后的遍历操作流程，则一定是有无效槽位存在
            if (slotToExpunge != staleSlot)
                // 清理无效槽位
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```



### 5.2、getEntry()分析

getEntry()函数并没有特别复杂的流程，只是需要注意可能会返回null值，具体见getEntryAfterMiss()函数的分析

```java
// 传入key，以获取entry实例。key为当前threadlocal实例
private Entry getEntry(ThreadLocal<?> key) {
            // 通过固定算法获取索引位置
            int i = key.threadLocalHashCode & (table.length - 1);
            // 取值
            Entry e = table[i];
            // 当entry实例不为null，且entry实例持有的引用指向当前的threadlocal实例，则返回entry实例
            if (e != null && e.refersTo(key))
                return e;
            else
                // 如果不符合条件，则进行额外操作
                return getEntryAfterMiss(key, i, e);
        }
```

#### 5.2.1、内部getEntryAfterMiss()分析

```java
// 传入key，索引值，对应的entry实例       
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            // table，为ThreadLocalMap内部定义的Entry[],用于存储用户创建的多个entry实例（键值对）
            Entry[] tab = table;
            // 获取容器的长度
            int len = tab.length;
			
            while (e != null) {
                // 当entry实例持有的引用指向当前的threadlocal实例，返回entry
                if (e.refersTo(key))
                    return e;
                // 当entry实例持有的引用指向null，则代表threadlocal实例已被回收内存，则清除无效槽位
                if (e.refersTo(null))
                    expungeStaleEntry(i);
                else
                    // 获取下一个entry实例，直到获取到null值结束
                    i = nextIndex(i, len);
                e = tab[i];
            }
    		// 如果没有符合条件的entry，则最后会返回null值
            return null;
        }
```





### 5.3、remove()分析

```java
// 传入key，即当前threadlocal实例        
private void remove(ThreadLocal<?> key) {
    		// table，为ThreadLocalMap内部定义的Entry[],用于存储用户创建的多个entry实例（键值对）
            Entry[] tab = table;
    		// 获取容器的长度
            int len = tab.length;
    		// 通过固定算法获取索引值
            int i = key.threadLocalHashCode & (len-1);
    		// 从当前索引值开始，依次向后遍历entry实例，直到遇到null值
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                // 如果当前entry持有的引用指向key，即当前threadlocal实例，则清除数据。同时清理无效槽位
                if (e.refersTo(key)) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```



### 5.4、expungeStaleEntry()分析

我们可以看到，在set()、getEntry()、remove()三种存、取、删的函数中，

都存在一个关键函数expungeStaleEntry()，用于清理entry[]中的无效槽位，下面简单分析一下其基本实现方式。

```java
// 传入无效槽位的索引信息        
private int expungeStaleEntry(int staleSlot) {
    		// table，为ThreadLocalMap内部定义的Entry[],用于存储用户创建的多个entry实例（键值对）
            Entry[] tab = table;
    		// 获取容器长度
            int len = tab.length;

            // 将当前槽位的key、value值置为null，以进行垃圾回收
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
    		// 容器中存储的实例数量减一
            size--;

            Entry e;
            int i;
    		// 从下一个槽位开始，依次向后遍历entry实例，直到遇到null值
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                // 获取key值，即threadlocal实例
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    // 当key值不为null，则通过固定算法获取其应该存储的索引位置
                    int h = k.threadLocalHashCode & (len - 1);
                    // 如果应该存储的位置（h）和实际存储的位置(i)不同，则必定在存储时计算错了位置，因此直接将当前位置（i）的数据置为null即可
                    if (h != i) {
                        tab[i] = null;

                        // 当前entry实例本应该存储在索引h，但如果索引h中不为null，则依次向后遍历，直到遇到null
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        // 将entry实例存入新的位置（null值所在的位置）
                        tab[h] = e;
                    }
                }
            }
    		// 返回无效槽位后下一个null值的索引，也可以理解为清除了staleslot 至 i之间的无效槽位信息
            return i;
        }
```





## 6、内存泄漏场景

在使用ThreadLocal的过程中，一定要使用remove()函数，将hash表中的数据清除。否则容易发生内存泄漏的情况，原因分析如下：

### 6.1、内存引用

通常情况下，内存引用如下图所示

![image-20250105192720320](https://s2.loli.net/2025/01/05/yiWBZGpXndmJD9k.png) 



我们首先从Key的指向，即ThreadLocal实例进行分析

- 假定ThreadLocal变量在函数执行完毕后被垃圾回收器将**内存回收**， 强引用失效。
- 此时ThreadLocal实例仅被**弱引用**（Entry实例中的key）指向，在垃圾回收器扫描到后，即会将ThreadLocal实例进行回收。
- 因此不会发生内存泄漏情况。

接着分析Value的指向，即内存中的Value值

- 由于Entry实例持有了**强引用**，指向了内存中的Vlaue值；因此若想回收Value值所占的内存，则必须回收Entry实例。
- 由于ThreadLocalMap底层是通过Entry实现的，因此ThreadLocalMap实例持有了Entry实例的引用，则必须先回收ThreadLocalMap实例。
- 由于Thread实例内部持有了ThreadLocalMap的引用，则必须先回收Thread实例。
- 如果Thread实例不能被内存回收，则整条链路的内存都不会被回收，就可能发生内存泄漏问题。

但是在实际开发中，我们经常使用线程池技术，线程池中的线程在使用后总是会重新投放到线程池中，并不会被回收，

因此就容易发生内存泄漏问题



### 6.2、解决方案

针对这种情况，有两种解决方案

1. 在代码执行完毕后，**手动销毁**当前Thread实例（不常见）
2. 手动调用ThreadLocal中的**remove()** 函数，进行内存回收操作

