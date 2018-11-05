## 背景

[TOC]

最近看了些关于`ThreadLocal`的文章，有一点思考，结合源码写了一些自己的见解.

### 1. ThreadLocal是什么?


可以通过`ThreadLocal`来维护在不同线程中同一个类的不同实例

- 作用域为线程
- 同一个类不同实例

> 一般来说，当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用 ThreadLocal。-《开发艺术探索》

### 2. 具体的实现

#### 2.1 以Looper为例


1. 创建ThreadLocal实例，泛型为Looper

   ```java
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
   ```

2. 设置默认值：

   ```java
   private static void prepare(boolean quitAllowed) {
       if (sThreadLocal.get() != null) {
           throw new RuntimeException("Only one Looper may be created per thread");
       }
       sThreadLocal.set(new Looper(quitAllowed));
   }
   ```

3. 取值

   ```java
   public static @Nullable Looper myLooper() {
       return sThreadLocal.get();
   }
   ```

**根据上面三个步骤，就实现了，每个线程能持有同一个类的不同实例**

#### 2.2 抽象整个过程


1. 设置泛型为目标类的ThreadLocal
2. 设置值的方法
3. 取出值的方法

### 3. ThreadLocal为什么能实现这样的功能?


在阐述这块功能之前我们先思考几个问题

- ThreadLocal存取是一个什么样过程？

- 既然ThreadLocal能使每个线程拥有拥有一个实例，那ThreadLocal和Thread之间有什么联系？
- ThreadLocal存储数据使用什么样的数据结构？

下面我们带着这仨个问题来完成这部分内容

#### 3.1 ThreadLocal的存取过程

##### 3.1.1 存

```java
//ThreadLocal.java中：
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            //1.已当前ThreadLocal为key将对应Value存储
            map.set(this, value);
        else
            //2.创建
            createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
}

//Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;

```

我们可以看到在ThreadLocal存储值的时候，使用ThreadLocalMap，那么我们先看看ThreadLocalMap是用来干什么的。

```java
/**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {
        ...
    }

```

**注释大致意思：ThreadLocalMap是一个定制的哈希映射，只适合维护线程本地值。在ThreadLocal类之外不可以进行任何操作。类是包私有的，允许在Thread中声明字段。为了帮助处理非常大且使用时间很长的用法，哈希表条目对键使用了WeakReferences。但是，由于没有使用引用队列，因此只有当表开始耗尽空间时才保证删除陈旧的条目。**

从这段信息中我们可以获取到：

1. 存储本地线程中的值
2. 使用Hash表结构来存储
3. 包私有，在Thread中声明使用
4. 对存储的值使用WeakReference

##### 3.1.2 取

```java
ThreadLocal.java中：

public T get() {
        Thread t = Thread.currentThread();
        //1.根据当前所在线程，将该线程的所有数据取出
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //3.通过以当前ThreadLocal为key将数据取出
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
    	//2.如果该线程存储数据，则设置默认数据
        return setInitialValue();
    }
    
private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
 protected T initialValue() {
        return null;
 }
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

通过上述分析我们可以看到，在取值时，若之前没有通过`set()`方法赋值，那么系统会默认赋值为`null`，那事实是不是这样呢？我们继续以`Looper`为例：

![](https://user-gold-cdn.xitu.io/2018/8/20/165570ccd0e077ff?w=2106&h=892&f=png&s=296775)

可以从测试代码中看到，当`Looper`，没有调用`prepare()`方法就使用`Looper.myLooper()`获取的值确实为`null`

#### 3.2 Thread和ThreadLocal之间的关系

**在结构上：**

![](https://user-gold-cdn.xitu.io/2018/8/20/165570bd3361704a?w=550&h=378&f=png&s=73812)


他们同处于`java.lang`包下，所有针对默认限制符的方法和属性，二者均可以调用，我们在上文阐述`ThreadLocalMap`中提到过，该类的的限定符是`default`所以只有在同包、子类、该类中调用，这就限制了`ThreadLocalMap`的使用。

**使用上：**

在ThreadLocal中通过getMap方法，获取了在Thread中的类型为`ThreadLocal.ThreadLocalMap`的`threadLocals`,

在Thread中创建`ThreadLocal.ThreadLocalMap`，开发者可以通过该变量获取存储在线程中的值。

 #### 3.3 ThreadLocalMap中使用的数据结构

使用的数据结构为哈希表；

> **散列表**（**Hash table**，也叫**哈希表**），是根据[键](https://zh.wikipedia.org/wiki/%E9%8D%B5)（Key）而直接访问在内存存储位置的[数据结构](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)。也就是说，它通过计算一个关于键值的函数，将所需查询的数据[映射](https://zh.wikipedia.org/wiki/%E6%98%A0%E5%B0%84)到表中一个位置来访问记录，这加快了查找速度。这个映射函数称做[散列函数](https://zh.wikipedia.org/wiki/%E6%95%A3%E5%88%97%E5%87%BD%E6%95%B0)，存放记录的数组称做**散列表**。 -《维基百科》

元素特征转变为数组下标的方法就是散列法；

哈希表的查找效率受冲突影响，冲突小，则效率会高，反之则低，那么我们知道影响产生冲突的有三个因素；

- 散列函数是否均匀
- 处理冲突的方法
- 散列表的载荷因子（英语：load factor = 填入表中的元素个数 / 散列表的长度）

##### 3.3.1 类结构

```java
    	//ThreadLocalMap是ThreadLocal的内部类
		ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            //1. 创建Entry(条目)数组，并且设置初始长度
            table = new Entry[INITIAL_CAPACITY];
            //2. 获取数组索引，通过key的hash值和数组长度做与运算，获取数组索引，这样的目的就是
            //存取方便，取值的时候，直接使用key的hash和数组做与运算，
            //与运算的结果就是一个0 <= index <= (INITIAL_CAPACITY - 1)的结果
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            //3. 设置数组的阀值，当超过这个阀值的时候进行扩容，这个我们稍后阐述
            setThreshold(INITIAL_CAPACITY);
		}

		//ThreadLocalMap中存放值的实体，针对key的应用采用的了弱引用
		static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

```java
	   // 初始化大小
        private static final int INITIAL_CAPACITY = 16;
       //存放数据的数组
        private Entry[] table;
        //table的大小
        private int size = 0;
        //动态扩容的阀值
        private int threshold; // Default to 0

		//设置阀值，这里的载荷因子(loadfactor 设置为2/3)
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
		//获取下一个索引
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
		//获取上一个索引
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
```

通过分析类结构我们可以得到ThreadLocalMap是如何避免冲突的：

- 散列函数尽可能均匀

  ```java
  注意到：int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  
  private static final int HASH_INCREMENT = 0x61c88647;
  private final int threadLocalHashCode = nextHashCode();
  private static AtomicInteger nextHashCode = new AtomicInteger();
  private static int nextHashCode() {
      return nextHashCode.getAndAdd(HASH_INCREMENT);
  }
  ```

  这是`ThreadLocalMap`获取hash值的方式，因为`nextHashCode`是静态变量，所以在内存中只存在一份,而ThreacLocal实例化的时候，这个code码已经生成了，生成的机制是在上一个`nextHashCode`基础上再加`HASH_INCREMENT`，这样尽可能的使数据均分在散列表上；

- 处理冲突的方式：线性探测法，当冲突发生的时候，往后探测i 个值，直到找到这个值

- 载荷因子 2/3,(标准为0.7-0.8) 较低，所以冲突的可能性较低

##### 3.3.2 取值

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    //1. 根据指定索引找到对应条目，如果这个条目不为空，且key相同，那就直接返回
    //这样做的目的是，因为hash值有可能冲突，特别是和table.length进行与操作之后
    //这样做的目的增大获取到对应的值效率
    if (e != null && e.get() == key)
        return e;
    else
        //2. 发现entry不为空，但是key值不相同，这就发生了hash冲突
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            //4. 删除这个item，已经被GC回收了
            expungeStaleEntry(i);
        else
            //3. 继续向下，寻找
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}

private static int nextIndex(int i, int len) {
	return ((i + 1 < len) ? i + 1 : 0);
}
```

> 这里我在考虑一个问题：在getEntryAfterMiss()方法中，如果e一直不为null，那这岂不是一个死循环？

其实这个是没必要担心的，我也是一点点理解的，因为散列表的载荷因子是2/3，是不可能装满的。

##### 3.3.3 存值

```java
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.
    
			//我们不像get()那样使用快速路径，因为使用set()创建新值与修改同样常见，在这种				  		  情况下，快速路径常常失败。
    		//这里快速路径就是直接使用 tab[i] = new Entry(key, value);
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

    		//for循环的作用就是避免重复添加
            for (Entry e = tab[i]; e != null;e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
				//如果当前存在，直接返回
                if (k == key) {
                    e.value = value;
                    return;
                }
				//之前存储过，但是被GC回收了，重新复制
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
			//添加信息的实体
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                //扩容
                rehash();
}
```

##### 3.3.4 扩容

```java
private void rehash() {
    //删除表中被GC被回收的数据，这部分内容比较简单，不贴代码了
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}

/**
 * 扩大到原来的两倍容量
 */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    //将原来的数据放到新数组中
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```
