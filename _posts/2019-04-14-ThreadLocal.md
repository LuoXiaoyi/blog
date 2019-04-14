---
layout:    post
title:     JDK ThreadLocal 源码剖析
category:  JDK 源码
tags: ThreadLocal ThreadLocalMap
---


## 概述
---
```ThreadLocal``` 顾名思义，就是“线程局部”的意思，换句话说就是属于某个线程的局部对象，其他线程是没法访问到的，亦即该对象不存在线程安全的问题，因为不可能被多线程访问到，这个是理解 ThreadLocal 的大前提。这篇文章将结合源码，从以下几方面做些分享
* 使用场景举例
* 源码剖析
* 可能存在的坑
  
## 使用场景举例
---
大家在使用数据库进行 java 代码编写的应用的时候，始终绕不开一个话题，那就是```事务```。比如说咱们在往数据库里面插入一条数据或者修改一条数据的某个字段时，我们在操作成功之后，必须要先提交事务，才能确保该数据真正被更新到数据库，这样应用程序的其他线程就能看到当前的最新数据了。那要做到这一点，我们怎么办到呢？为了更好的说明场景，我们举一个更为具体的例子。如用户注册，大概的流程如下：<br>
* 用户在前端提交注册请求 
* 后端 service 层代码进行业务逻辑处理 
* service 调用 DAO 层将数据插入到数据库
* service 通过判断 DAO 的插入结果给前端返回注册结果

### 上述过程的一个可能实现的伪代码如下
```
public class UserController{

    public Result<Object> userRegister(User user){
        ...
        UserService service = new UserService();
        return service.doRegister(user);
    }
}

public class UserService{

    public Result<Object> doRegister(User user){
        // 可能需要进行一些相关的验证
        ...
        UserBo bo = transfer(user);
        TransactionManager.begin(); // 开启事务
        UserDao dao = new UserDao();
        try{
            dao.save(bo);
            TransactionManager.commit(); // save 成功，提交事务
        }catch(Exception e){
            TransactionManager.rollback(); // save 异常，回滚事务
        }
        ...
        // 根据结果返回相应的注册结果
    }
}

// 数据库事务管理器
public class TransactionManager{

    private static ThreadLocal<Session> sessionLocal = new ThreadLocal<>();

    public static Session begin(){
        Session s = sessionLocal.get();
        // 如果当前线程中不存在事务，则 new 一个，否则使用之前的事务（当然这种策略是可以改的，Spring 的事务管理中就支持可配置事务的传播特性）
        if(s == null){
            s = new Session();
            sessionLocal.set(s);
        }
        
        s.open();
        return s;
    }

    public static void commit(){
        Session s = sessionLocal.get();
        if(s != null){
            s.commit();
            s.close();
            sessionLocal.remove();
        }

        throw new TransactionException("Transaction state error.");
    }

    public static void rollback(){
        Session s = sessionLocal.get();
        if(s != null){
            s.rollback();
            s.close();
            sessionLocal.remove();
        }
        
        throw new TransactionException("Transaction state error.");
    }
}
```
代码特别简单易懂，就不逐一说明了。从上面的代码看的出来，我们在事务管理器里面实际上就用到了 ThreadLocal 来保存我们的数据库 session，用于事务的开始、提交或回滚。如果不了解 ThreadLocal 的同学可能会问了，你这个是个静态变量，如果在多线的环境下运行，不是多个线程同时访问了同一个 ThreadLocal 对象了么？那是不是有可能，在线程 A 中开启的事务，被线程 B 给 commit 或 rollback 了？答案当然是否定的。这就是 ThreadLocal 的设计巧妙之处了。带着这样的疑问，我们通过源码一窥究竟。

## 源码剖析
---
### 关键类和以及各类之间的关系
一说到 ThreadLocal 就不得不提 Thread、ThreadLocalMap，因为这三者是紧密联系在一起的，他们之间的关系如下图
![system](/blog/images/threadlocal/thread-threadlocal-threadlocalmap.png)
<br>
从上图中可以看到，Thread 中有个 ThreadLocalMap 的变量 threadLocals，用于保存当前线程中几乎所有的 ThreadLocal 对象（之所有不是全部，是因为实际上线程还有可能从父线程继承一些 ThreadLocal 对象过来，并且保存在ThreadLocal.ThreadLocalMap inheritableThreadLocals这个变量中），而 ThreadLocalMap 是 ThreadLocal 的一个内部类，并且 ThreadLocal 对象与 ThreadLocal 对象所代表值实际上都是以<ThreadLocal, Object>这样的键值对的形式保存在 ThreadLocalMap 中的，并不是把 ThreadLocal 指向的值直接保存在 ThreadLocal 中，在 ThreadLocal 中只有一个 threadLocalHashCode 字段，用于保存当前对象的 hash 值。这样我们对这三者在关系上就有了一个宏观上的了解了。接下来我们重点分析 ThreadLocal 的3个 public 方法，set(T obj), get(), remove() 方法。

### ThreadLocal
---
#### ThreadLocal.set(T value)
表示往 ThreadLocal 对象中存入某个值，该值并不是保存在 ThreadLocal 对象中的，而是保存在当前线程的 ThreadLocalMap 中。
源码如下
```
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程的ThreadLocalMap，即 threadLocals
    ThreadLocalMap map = getMap(t);
    // 不为空，则将 ThreadLocal 和 value 添加到 map 中
    if (map != null)
        map.set(this, value);
    else
        // 用 threadlocal 和 value 为 Thread 的第一个 ThreadLocal 对象，并创建一个 ThreadLocalMap
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals; // 返回 Thread 中的 ThreadLocalMap 私有成员
}

void createMap(Thread t, T firstValue) {
    // 创建一个 ThreadLocalMap 实例，并赋值给 Thread 的 threadLocals
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

#### T ThreadLocal.get()
该方法表示，根据 ThreadLocal 对象，从各个 Thread 中获取出它所代表的值，如果还未进行 set，则会调用 setInitialValue() 方法来进行初始值的设定，并返回给调用线程。这里需要注意的是，虽然不同线程拿到的 ThreadLocal 对象是同一个，但是，在不同的线程中，该对象所代表的值是不一样的，因为值被保存在不同线程的 ThreadLocalMap 中了，理解这点非常非常重要。下面我们看源码
```
public T get() {
    Thread t = Thread.currentThread();
    // 获取当前线程的 threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 若 map 不为空，则尝试从 map 中获取当前 ThreadLocal 对象所对应的 Entry 对象，这个 Entry 对象后面会细讲
        ThreadLocalMap.Entry e = map.getEntry(this);
        // 如果不为空，则返回 entry 的 value，接 set 时插入的 value 对象
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }

    // 如果上面的查找过程中没有找到 ThreadLocal 所对应的值，则尝试调用 setInitialValue 方法来获取值，因为 ThreadLocal 对象很有可能覆写了 initialValue 方法来获取值
    return setInitialValue();
}

private T setInitialValue() {
    // 调用 initialValue 获取初始化值
    T value = initialValue();
    // 进行前面已经说明过的 set 方法同样的逻辑将初始值插入到线程的 ThreadLocalMap 中
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    // 返回获取的值
    return value;
}
```

#### ThreadLocal.remove()
从 Thread 的 ThreadLocalMap 中把 ThreadLocal 锁代表的值删除掉，即类似平常我们使用 Map 时调用 remove(key) 的思路一样，只是在 ThreadLocalMap 中的实现逻辑不一样。后面在 ThreadLocalMap 中进行分析的时候会详细说明，我们这里先不展开。remove 的源码如下
```
public void remove() {
    // 获取当前线程的 ThreadLocalMap
    ThreadLocalMap m = getMap(Thread.currentThread());
    // 如果不为空，则尝试从 ThreadLocalMap 删除 ThreadLocal 锁代表的值
    if (m != null)
        m.remove(this); // 调用 ThreadLocalMap 的 remove 方法进行删除
}
```
### ThreadLocal总结
以上对 ThreadLocal 的几个关键方法进行了详细的说明，逻辑上还是很简单的，因为其核心逻辑全部被实现在 ThreadLocalMap 中。从 ThreadLocal 的实现我们不难发现，当我们使用同一个 ThreadLocal 对象来在不同的线程中保存值的时候，实际上只是用它做了 key，而对应的 value 是不同的，所保存的位置也是不同线程的 ThreadLocalMap 中，这就是为什么在前面的例子中，即使多个线程同时用同一个 ThreadLocal 对象来保存 session 时，不会出现 A 线程开启的事务，被 B 线程 rollback 或 commit 这种情况了，因为他们的 session 实际上是两个不同对象，且完全被保留在不同的 ThreadLocalMap 中。<br>
接下来我们在来重点分析一下 ThreadLocalMap 中的核心实现。

### ThreadLocalMap
---
底层实际上是一个数组，和 HashMap 是一样的结构。
#### 成员变量说明
```
// 就是一个数组对象，每个数组元素是一个 Entry 对象
private Entry[] table;

// map 中元素的个数
private int size = 0;

// 用于扩容时做阈值判断，默认情况下和 HashMap 的装载因子效果一样
private int threshold; // Default to 0
```
#### 静态成员说明
```
// 这就是那个核心的数组元素对象，该对象是 WeakReference 的子类
static class Entry extends WeakReference<ThreadLocal<?>> {
    // ThreadLocal 的 value 会被保存在这个字段上
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        // ThreadLocal 作为 key，被保存在 WeakReference 的 referent 属性上，这个很重要，因为弱引用在 GC 的时候会被自动回收
        super(k);
        value = v;
    }
}

// 初始 map 的大小，为2的 n 次方
private static final int INITIAL_CAPACITY = 16;
```
#### 核心方法说明
##### ThreadLocalMap 构造函数
---
构造函数有好2个，源码如下
```
// 根据给定的 threadLocal 和 value 来构造一个新的 ThreadLocalMap
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 初始化 table 数组
    table = new Entry[INITIAL_CAPACITY];
    // ThreadLocal 中的 hash code 计算数组中的位置
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    // 设置 threshold，方便扩容
    setThreshold(INITIAL_CAPACITY);
}

// 该构造函数用于拷贝一个 ThreadLocalMap，类似 C++ 中的拷贝构造函数，该函数主要用于线程在创建子线程时，
// 将当前线程中的 ThreadLocal 变量都拷贝到子线程，如果有必要的话，这个可以参考 Thread 的构造函数中的 inheritThreadLocals，这个默认情况下是 true
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```
##### ThreadLocalMap.set(ThreadLocal<?> key, Object value)
---
```
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 先判断当前位置上是否有元素已经存在，如果没有则直接结束 for 循环，调到后面直接 new 一个对象插入 i 的位置
    // 否则继续往下查找下一个元素，直到找到为 null 的 slot 为止
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        // 如果不为空，则比较两个 key 是否相等，如果相等，则直接覆盖原来的值，并结束 set 方法
        if (k == key) {
            e.value = value;
            return;
        }

        // 如果 key 为 null，即说明该 key 已经被 gc 了，则用当前的最新值替换掉老值，
        // 注意，这里之所以不能直接将 value 替换成 e.value 是因为，当前的 key 已经是 null 了，无法确定 key 是否和当前的key 一样，
        // 因为仅仅只是 hash 值计算出来的 slot 一样而已，所以需要使用 replaceStaleEntry 算法遍历整个数组去找对应的 key
        // 在根据实际情况来做赋值，具体我们在 replaceStaleEntry 中会详述
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 创建一个新的 Entry 插入表中
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 做一次过期值的清理，如果没有元素被清理了并且元素值超过了扩容阈值，则执行 rehash 扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
##### set 方法总结如下
1. 根据插入的 key 的 hash 值计算在 table 中的 slot 位置
2. 如果该位置 slot 上有元素，则循环执行如下判断，直到找到没有元素的 slot 或者 满足如下任意条件为止
    * 若元素的 key 是否和当前 key相等，若相等则直接用新值替换旧值，set 方法结束
    * 若元素的 key 为 null（被 GC 掉了），则无法判断之前此处的元素key 是否和 当前 key 相等，所以需要执行 replaceStaleEntry 算法，进行元素值的替换，set 方法结束
3. 如果找到 slot 为空的表位置，则将key、value 插入到当前 slot 位置
4. 尝试执行一次过期元素清理，如果清理了元素则不需要扩容（因为清理的数目必然大于等于1），而当前也只插入了一个元素，故无需扩容，如果没有清理掉任何元素，且当前表的 size 大于等于扩容阈值，则执行扩容操作

##### ThreadLocalMap.remove(ThreadLocal<?> key)
---
```
// 从 map 中将 key 删除
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // for 用于解决 hash 冲突，如果不为空，则一直往下遍历，直到表的 slot 为空为止
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        // 只有在 key 相等的情况下才执行删除操作
        if (e.get() == key) {
            // referent 置 null
            e.clear();
            // 执行 expungeStaleEntry 删除算法
            expungeStaleEntry(i);
            return;
        }
    }
}
```
##### remove 方法总结如下
1. 根据 key 的 hash 值计算在 table 中的 slot 位置
2. 若 slot 上的元素 e 为 null，则什么也不做，返回，否则执行3
3. 若 e 不为 null，在循环比较接下来的每一个不为 null 的 table 元素，直到以下任意条件为止
   * e 的 key 和 被删除 key 相等，则执行删除操作，使用 expungeStaleEntry 进行删除；
   * 否则继续查找表的下一个元素，直到下一个元素为 null 为止，表示没有匹配到任意要删除的元素

##### ThreadLocalMap.replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot)
---
该方法包含的3个参数，第一、二两个参数是 map 中 KV 价值对，第三个参数为过期的 slot 位置。主要作用是，用新的 key、value 去替换可能在 staleSlot 上的值，由于可能存在 hash 冲突，所以需要被替换的值，不一定就在 staleSlot 上，这个时候就需要向 staleSlot 的后面进行巡检查找该元素；
1. 先对 staleSlot 之前的 slot 进行巡检，一直找到表中最后一个元素不为空，且 key 为空的元素，并记录其位置为 slotToExpunge ；
2. 然后再从 staleSlot 往后巡检，查找与当前 key 相等的元素，如果找到，则用 value 替换掉原来的 value，同时执行清理动作，方法返回；
3. 如果没有找到，则可以将 staleSlot 替换成新的 key，value。最后如果slotToExpunge!=staleSlot, 则说明有需要被删除的元素，从而执行删除算法 cleanSomeSlots。

<br>
源码如下
```
// 使用 key、value， 替换掉 staleSlot 上的或该位置之后元素不为空且key等于入参 key 的元素
private void replaceStaleEntry(ThreadLocal<?> key, Object value,int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 往前巡检，直到table[i] == null，并且记录下最后一个 table[i]!=null 且 key 为 null 的元素位置，以便后续的删除
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // 从 staleSlot 的下个位置开始查找，如果table[i]不为空，且 table[i].get() == key, 则用 value 替换掉原来的值，否则继续往下巡检，直到 table[i] == null
    for (int i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        
        // 找到需要被替换掉的过期元素
        if (k == key) {
            // 替换 value
            e.value = value;

            // 交换 table[i] 和 table[staleSlot]
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // 重置 slotToExpunge 为 i
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;

            // 执行清除逻辑
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 若我们在前面没有找到过期的元素，则 slotToExpunge 的位置应该从 i 开始
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 若没有找到绝对匹配的元素，则直接替换 staleSlot 的值为新值
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 如果存在需要被清理的元素，则执行清理算法 cleanSomeSlots
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```
##### ThreadLocalMap.expungeStaleEntry(int staleSlot)
---
该方法的主要作用是
1. 删除 staleSlot 的值
2. 从 staleSlot 位置开始，直到下一个table[i]为 null 的元素之间，删除 table[i].key 为 null 的元素，同时调整该区间内的元素位置到更正确的位置上。

<br> 
源码如下
```
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 删除过期元素
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // 删除或者重新 rehash 元素，从 staleSlot 开始直到为空结束
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            // 删除过期元素
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // rehash 元素到更恰当的位置，主要前期是由于 hash 冲突引起的
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

##### ThreadLocalMap.cleanSomeSlots(int i, int n)
---
该方法是删除从 i 开始，并进行 log2(n) 次启发式删除。其中 i 所代表的位置是不存在任何元素的位置，然后每次从它的下一个位置开始进行删除，每次删除都执行 expungeStaleEntry 算法，总共进行 log2(n) 次。若成功删除至少一个元素，则返回 true，否则返回 false，表示此次没有删除任何元素
```
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 如果 e 存在，且 key 已经过期，则执行删除
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0); // n \= 2
    return removed;
}
```
### ThreadLocalMap 总结
至此，ThreadLocalMap中所有核心逻辑和方法都已经介绍完，做个简单总结
1. ThreadLocalMap 的 key 是 WeekReference，这样就决定了，在 GC 时，只要没有任何强引用指向 Map 中的key，那么该 Key 就会在下一次 GC 时被回收，从而 key.get() 就会返回 null，而在 set、get 等方法时，会对这些 key.get() 为 null KV 键值对进行清理操作，从而释放内存，避免内存泄漏；但是和 WeekHashMap 一样，即使 key 被 GC 掉了，变为 null，但是如果从此之后不再对 map 做任何操作，那么这些空 key 对应的 value 所占用的内存也是不可能被 GC 的，在这种情况下，有可能造成内存泄漏；
2. ThreadLocalMap 中处理 hash 冲突时采用的并不像传统的 hash 表那样，用链表的方式来进行，而是一旦产生 hash 冲突，假设元素计算出来的数组下标为 i，那么该元素将会被保存在从 i+1 开始的第一个为 null 的元素所在的slot，假设是 k，这就有可能造成一种问题，原本应该被保存在 k 上的元素，有可能再次被迫执行并不是 hash 冲突时的hash 冲突算法（即同样往后移动，直到后续的 table 元素为空来保存该元素）；
3. 虽然说 ThreadLocalMap 中的 key 是弱引用，但是，一旦外边还有其他强引用指向该 key，意味着该 key 永远都不可能被 gc 掉，那么其 WeakReference 的能力就基本失去了作用，这就势必导致内存无法回收的后果，所以每次使用完，如果确定不再需要该 key，就需要调用 ThreadLocal 的 remove 方法或者 set 方法（set(null)）来帮助 GC 回收相关内存，这将是一个很好的编程习惯。

### 可能存在的坑
在上面总结 ThreadLocalMap 的时候，已经大概总结出来了一些，这里就把我碰到过的坑都列一下
1. 使用线程池处理事务时，如果同一个线程上一次处理过的 ThreadLocal 对象，在用完之后没有及时清理，很有可能导致该线程在处理下一个事务 B 时，由于在某些情况下没有重新设置该 ThreadLocal 对象，而导致处理事务 B 的时候，读取到了上一次处理过程中 ThreadLocal 的值，从而产生脏读到现象；
2. 内存泄漏问题，因为一般情况下，ThreadLocal 对象可能是静态对象，意味着在 Thread 的 ThreadLocalMap 中该 ThreadLocal 的弱引用失效（因为静态对象的引用是强引用），从而无法达到 key 被 GC 掉，并且在后续操作 Thread 的 ThreadLocalMap 时，无法将不再需要的对象清除掉而造成内存泄漏的问题；
3. 在字节码增强项目中碰到了，由于使用 ThreadLocal 而导致类加载器无法被卸载的问题。具体情况是，使用 ClassLoaderA 加载了 ClassA，并且将 A 作为 ThreadLocal 保存到了目标 JVM 线程（比如 thread1）的 ThreadLocalMap 中，并且在使用完之后，没有及时从 thread1 的 ThreadLocalMap 中 remove 掉，而 thread1 会一直存在，不会死，从而导致 ClassA 的这个对象一直从 thread1 GC 可达，从而造成 ClassA 一直有 live 的对象，而导致 ClassA、ClassLoaderA 以及由 ClassLoaderA 加载的所有的类、静态对象等无法被 gc 和从 JVM 中卸载掉。