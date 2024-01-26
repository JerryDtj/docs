# 多线程之threadlocal

## 作用

主要用于多线程之间的数据隔离,防止数据被多线程污染

## 代码示例

```java
package threadlocal;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author Jerry
 * @Date 2024/1/26 07:13
 */
public class ThreadlocalLearn {
    public static void main(String[] args) throws InterruptedException {
        ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        ExecutorService pool = Executors.newFixedThreadPool(3);
        for (int i=0;i<6;i++){
            int finalI = i;
            if (i<3) {
                pool.execute(() -> {
                    threadLocal.set(finalI);
                    System.out.println(Thread.currentThread().getId()+" set "+threadLocal.get());
                });
            }else {
                pool.execute(() -> {
                    System.out.println(Thread.currentThread().getId()+" get "+threadLocal.get());
                });
            }
        }
        pool.shutdown();
    }
}
```

## 输出结果

```tex
11 set 0
13 set 2
12 set 1
11 get 0
12 get 1
13 get 2

进程已结束，退出代码为 0
```

## 原理

### 准备动作

在看原理之前,先回顾下java的4种引用关系:

1. 强引用: 直接通过new 方法得到的对象.只要强引用存在,那么gc就不回收这个对象
2. 软引用: 使用**SoftReference**修饰的对象.该对象只在即将发生内存溢出时回收
3. 弱引用:使用**WeakReference**修饰的对象,该对象只要发生gc就会被回收
4. 虚引用:使用**PhantomReference**修饰的对象.虚引用唯一的作用就是用队列接收对象即将死亡的通知

threadlocal为弱引用.如果在项目运行时触发gc,导致一些执行完成的线程被回收.而这些线程又存在threadlocalMap中.那么就会出现**key为null,但是有value值**的数据.这种数据由于entry[]的key已经被回收了,导致gc无法回收从而引发数据泄露.

那么threadlocal有什么改进措施呢,请往下看.

### ThreadLoacalMap声明

1. 内部类声明,防止外部直接调用
2. 采用Entry[]的形式来存放不同的线程数据.
3. 内部采用主动扫描(cleanSomeSlots)+数据清理(expungeStaleEntry)的方式来防止数据泄露

```java
/**
 * ThreadLocalMap是一个自定义的哈希映射，仅适用于维护线程本地值。不将任何操作导出到ThreadLocal类的外部。该类是包私有的，以允许在   * 类Thread中声明字段。为了帮助处理非常大和长期的用法，哈希表条目使用weakreference作为键。但是，由于不使用引用队列，因此只有在表	* 开始空间不足时，才能确保删除陈旧条目
 */
static class ThreadLocalMap {
  /**
  * 此哈希映射中的条目扩展WeakReference，使用其主ref字段作为键 (始终是ThreadLocal对象)。请注意，空键 
  *	(即entry.get() = = null) 意味着该键不再被引用，因此该条目可以从表中删除。这样的条目在下面的代码中被称为 “陈旧条目”。
  **/
  static class Entry extends WeakReference<ThreadLocal<?>> {
      /** The value associated with this ThreadLocal. */
      Object value;

      Entry(ThreadLocal<?> k, Object v) {
          super(k);
          value = v;
      }
  }
  
 /**
 * 表，根据需要调整大小。table.length必须始终为2的幂。
 */
  private Entry[] table;
  
  /**
  * 初始容量-必须是2的幂。
  **/
  private static final int INITIAL_CAPACITY = 16;
  
  /**
  *	要调整大小的下一个size值。
  **/
  private int threshold;
  
  /**
  *	设置调整大小阈值以在最坏的情况下保持2/3的负载系数
  **/
  private void setThreshold(int len) {
      threshold = len * 2 / 3;
  }
	
  /**
  *	构造一个最初包含 (firstKey，firstValue) 的新映射。ThreadLocalMaps是懒惰地构造的，
  *	所以我们只在至少有一个条目要放入它时创建一个
  **/
  ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
      table = new Entry[INITIAL_CAPACITY];
      int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
      table[i] = new Entry(firstKey, firstValue);
      size = 1;
      setThreshold(INITIAL_CAPACITY);
  }

  /**
  *	获取与key关联的条目。此方法本身仅处理快速路径: 直接命中现有密钥。否则会中继到getEntryAfterMiss。
  * 这被设计成使直接命中的性能最大化，部分地通过使该方法容易地不可用。
	* 形参:
	*		key -线程本地对象
	* 返回值:
	*		与key关联的条目，如果没有，则为null
  **/
  private Entry getEntry(ThreadLocal<?> key) {
    	//i = 7
      int i = key.threadLocalHashCode & (table.length - 1);
      Entry e = table[i];
      if (e != null && e.get() == key)
          return e;
      else
          return getEntryAfterMiss(key, i, e);
  }
  
  /**
  *	在其直接哈希槽中找不到key时使用的getEntry方法的版本。
  * 形参:
  * 	key -线程本地对象 
  *		i -key的哈希码 
  *		e 的表索引-表 [i] 中的条目
  * 返回值:
  * 	与key关联的条目，如果没有，则为null
  **/
  private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
      Entry[] tab = table;
      int len = tab.length;

      while (e != null) {
          ThreadLocal<?> k = e.get();
          if (k == key)
              return e;
          if (k == null)
              expungeStaleEntry(i);
          else
              i = nextIndex(i, len);
          e = tab[i];
      }
      return null;
  }
  
  /**
  *	将在设置操作期间遇到的过时条目替换为指定键的条目。在value参数中传递的值存储在条目中，无论指定键的条目是否已经存在。
  *	作为副作用，此方法删除包含陈旧条目的 “运行” 中的所有陈旧条目。(运行是两个空槽之间的条目的序列。)
	*	形参:
	*		key the key 
	*		value 和key关联的值 
	*		stalelot 查找key时遇到的第一陈旧的entry数据
  **/
  private void replaceStaleEntry(ThreadLocal<?> key, Object value,int staleSlot) {
      Entry[] tab = table;
      int len = tab.length;
      Entry e;

      // 备份以检查当前运行中以前的陈旧条目。我们一次清理整个运行，
    	// 以避免由于垃圾收集器释放成束的引用 (即，每当收集器运行时) 而导致的连续增量重新哈希
      int slotToExpunge = staleSlot;
      for (int i = prevIndex(staleSlot, len);
           (e = tab[i]) != null;
           i = prevIndex(i, len))
          if (e.get() == null)
              slotToExpunge = i;

      // 查找运行的键或尾随空槽，以先发生者为准
      for (int i = nextIndex(staleSlot, len);
           (e = tab[i]) != null;
           i = nextIndex(i, len)) {
          ThreadLocal<?> k = e.get();

          // 如果我们找到key，那么我们需要将其与陈旧的条目交换以维护哈希表顺序。
        	// 然后，可以将新的陈旧插槽或在其上方遇到的任何其他陈旧插槽发送到expungeStaleEntry，
        	// 以删除或重新散列run中的所有其他条目。
          if (k == key) {
              e.value = value;

              tab[i] = tab[staleSlot];
              tab[staleSlot] = e;

              // Start expunge at preceding stale entry if it exists
              if (slotToExpunge == staleSlot)
                  slotToExpunge = i;
              cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
              return;
          }

          // 如果我们在向后扫描时未找到陈旧条目，则扫描密钥时看到的第一个陈旧条目是运行中仍然存在的第一个。
          if (k == null && slotToExpunge == staleSlot)
              slotToExpunge = i;
      }

      // If key not found, put new entry in st	ale slot
      tab[staleSlot].value = null;
      tab[staleSlot] = new Entry(key, value);

      // If there are any other stale entries in run, expunge them
      if (slotToExpunge != staleSlot)
          cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
  }
  
  /**
  *	重新包装和/或重新调整表的大小。首先扫描整个表，删除陈旧的条目。如果这不能充分缩小表的大小，则将表大小加倍
  **/
  private void rehash() {
    	//删除key值为null的数据
      expungeStaleEntries();

      // Use lower threshold for doubling to avoid hysteresis
      if (size >= threshold - threshold / 4)
          resize();
  }
  
  /**
  *	当entry[]数组大小有变动时,会调用此方法调整数组大小并清理一些无用数据
  *	通过重新散列位于staleSlot和下一个空槽之间的任何可能冲突的条目来删除陈旧的条目。这还会删除在尾随null之前遇到的任何其他陈旧条目。	*	参见Knuth，第6.4节
	*	形参:
	*		staleSlot -插槽的索引已知具有空键
	*	返回值:
	*		staleSlot之后的下一个空插槽的索引 (将检查staleSlot和此插槽之间的所有值以进行删除)。
  **/
  private int expungeStaleEntry(int staleSlot) {
      Entry[] tab = table;
      int len = tab.length;

      // expunge entry at staleSlot
      tab[staleSlot].value = null;
      tab[staleSlot] = null;
      size--;

      // Rehash until we encounter null
      Entry e;
      int i;
      for (i = nextIndex(staleSlot, len);
           (e = tab[i]) != null;
           i = nextIndex(i, len)) {
          ThreadLocal<?> k = e.get();
          if (k == null) {
              e.value = null;
              tab[i] = null;
              size--;
          } else {
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
  
  /**
  *	启发式地扫描一些单元格，寻找陈旧的条目。当添加新元素或删除另一个陈旧元素时，将调用此方法。它执行对数扫描，
  *	作为无扫描 (快速但保留垃圾) 和与元素数量成比例的扫描数量之间的平衡，这将找到所有垃圾，但会导致一些插入需要O(n) 时间。
	*	形参:
	*		i 已知没有陈旧条目的位置。扫描从i之后的元素开始。 
	*		n 计算出的entry数组的大小
	*			扫描控制:log2(n)扫描单元格，除非找到陈旧的条目，在这种情况下log2(table.length)-1扫描其他单元格。
	*			当从插入调用时，此参数是元素的数量，但是当从replaceStaleEntry调用时，
	*			它是表长度。(注意: 所有这一切都可以通过加权n而不是仅仅使用直日志n来改变，但这个版本简单，快速，似乎工作得很好。)
	*	返回值:
	*		如果已删除任何陈旧条目，则为true
  **/
  private boolean cleanSomeSlots(int i, int n) {
      boolean removed = false;
      Entry[] tab = table;
      int len = tab.length;
      do {
          i = nextIndex(i, len);
          Entry e = tab[i];
          if (e != null && e.get() == null) {
            	//如果发现key不为空但是key引用的对象(线程)已经被gc回收,那么触发数据清理
              n = len;
              removed = true;
              i = expungeStaleEntry(i);
          }
      } while ( (n >>>= 1) != 0);
      return removed;
  }
  
  private static int nextIndex(int i, int len) {
      return ((i + 1 < len) ? i + 1 : 0);
  }
  
  /**
  *	返回此引用对象的referent。如果此引用对象已被程序或垃圾回收器清除，则此方法返回null。
	*	返回值:
	*		此引用所引用的对象，如果此引用对象已被清除，则为null
  **/
  public T get() {
      return this.referent;
  }
  
  //......
}
```

### 获取数据:get()

获取时会根据entry数组的实际大小和默认值大小取余去做定位,如果定位失败,那么就会执行数据清理

```java
/**
*	返回此线程局部变量的当前线程副本中的值。如果该变量没有当前线程的值，则首先将其初始化为通过调用initialValue方法返回的值。
* 返回值:
*		此线程本地的当前线程值
**/
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
  	//第一次直接调用get(),ThreadLocalMap默认值为null
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

/**
*	set() 的变体来建立initialValue。在用户重写set() 方法的情况下使用而不是set()。
*	返回值:
*		初始值
**/
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

/**
*	创建与ThreadLocal关联的映射。在InheritableThreadLocal中重写。
*	形参:
*		t -当前线程 
*		firstValue -映射初始条目的值
**/
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

/**
* 获取与ThreadLocal关联的地图。在InheritableThreadLocal中重写。
*	形参:
*		t -当前线程
*	返回值:
*		the map
**/
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

ThreadLocal.ThreadLocalMap threadLocals = null;

//连续生成的哈希码之间的差异-将隐式顺序线程本地id转换为两次幂大小的表的近乎最佳扩展的乘法哈希值。
private static final int HASH_INCREMENT = 0x61c88647;

/**
*	ThreadLocals依赖于附加到每个线程的每线程线性探针哈希映射 (thread.threadLocals和inheritableThreadLocals)。
*	ThreadLocal对象充当键，通过threadLocalHashCode搜索。这是一个自定义哈希代码 (仅在ThreadLocalMaps中有用)，
*	它消除了在相同线程使用连续构造的ThreadLocals的常见情况下的冲突，同时在不太常见的情况下保持良好的行为
**/
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

### 放置数据:set(T value)

放置时把当前线程和对应的指放入到ThreadLocalMap中,该操作会触发**数据扫描+数据清理**

```java
/**
*	设置与键关联的值。
*	形参:
*		key -线程本地对象 
*		value -要设置的值
**/
private void set(ThreadLocal<?> key, Object value) {

    // 我们不像get() 那样使用快速路径，因为使用set() 来创建新条目至少和替换现有条目一样常见，
  	// 在这种情况下，快速路径会经常失败。
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
          //如果这个值在threadLocalMap中已经存在,那么直接返回
            e.value = value;
            return;
        }

        if (k == null) {
          	//如果发现线程key已经被gc回收了,那么就触发数据清理
            replaceStaleEntry(key, value, i);
            return;
        }
    }
		
  	//开始放value
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
      	//如果entry需要扩容
        rehash();
}
```

### 移除数据:remove()

```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```
