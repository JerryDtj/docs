# ThreadLocal原理

## 是什么

在多线程场景下，为每一个线程创建一个独立的静态私有变量。

画外音：这句话里面有三层含义

- ThreadLocal是一种变量类型，我们称之为“线程局部变量”
- 每个线程访问这种变量的时候都会创建该变量的副本，这个变量副本为线程私有
- ThreadLocal类型的变量一般用private static加以修饰

## 使用示例

```java
public class QLExpressTimer {

    private static ThreadLocal<Boolean> NEED_TIMER = new ThreadLocal<Boolean>(){
        @Override
        protected Boolean initialValue() {
            return false;
        }
    };
    private static ThreadLocal<Long> TIME_OUT_MILLIS = new ThreadLocal<Long>(){};
    private static ThreadLocal<Long> START_TIME = new ThreadLocal<Long>(){};
    private static ThreadLocal<Long> END_TIME = new ThreadLocal<Long>(){};



    /**
     * 设置计时器
     * @param timeoutMillis 超时时间
     */
    public static void setTimer(long timeoutMillis)
    {
        NEED_TIMER.set(true);
        TIME_OUT_MILLIS.set(timeoutMillis);
    }

    /**
     * 开始计时
     */
    public static void startTimer()
    {
        if(NEED_TIMER.get()) {
            long t = System.currentTimeMillis();
            START_TIME.set(t);
            END_TIME.set(t+TIME_OUT_MILLIS.get());
        }
    }


    /**
     * 断言是否超时
     * @throws QLTimeOutException
     */
    public static void assertTimeOut() throws QLTimeOutException {

        if(NEED_TIMER.get() && System.currentTimeMillis()>END_TIME.get()){
            throw new QLTimeOutException("运行QLExpress脚本的下一条指令将超过了限定时间:" + TIME_OUT_MILLIS.get() + "ms");
        }
    }

    public static boolean hasExpired()
    {
        if(NEED_TIMER.get() && System.currentTimeMillis()>END_TIME.get()){
            return true;
        }
        return false;
    }

    public static void reset()
    {
        if(NEED_TIMER.get()) {
            START_TIME.remove();
            END_TIME.remove();
            NEED_TIMER.remove();
            TIME_OUT_MILLIS.remove();
        }
    }
}
```

## 主要操作

```java
/**
 * 返回此线程局部变量的当前线程的“初始值”。 该方法将在线程第一次使用get方法访问变量时调用，除非线程之前调用了set方法，在这种情况下，
 * 将不会为线程调用 initialValue方法。 通常，每个线程最多调用此方法一次，但在后续调用remove后跟get情况下，它可能会再次调用。
 * 此实现仅返回null ； 如果程序员希望线程局部变量具有除null以外的初始值，则必须对ThreadLocal进行子类化，并覆盖此方法。 通常，将使用匿名内部类。
 * 返回：此线程本地的初始值
 */
protected T initialValue() {
    return null;
}

/**
  * 将此线程局部变量的当前线程副本设置为指定值。 
  * 大多数子类将不需要覆盖此方法，仅依赖于initialValue方法来设置线程initialValue的值。
  * 参数：value -- 要存储在此线程本地的当前线程副本中的值。
  */
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

/**
  * 返回此线程局部变量的当前线程副本中的值。 如果该变量对于当前线程没有值，则首先将其初始化为调用initialValue方法返回的值。
  * 返回：此线程本地的当前线程的值
  */
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
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
  * 删除此线程局部变量的当前线程值。 如果此线程局部变量随后读取当前线程，它的价值将通过调用其重新初始化initialValue方法，除非它的值设置在临时当前线程。
  * 这可能会导致在当前线程中多次调用initialValue方法。
  * 自从：1.5
  */
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

```

## 上源码

### 所有方法

![image-20211019190123234](https://i.loli.net/2021/10/19/J3o7IGvQuVD2d64.png)

### set方法

![img](https://i.loli.net/2021/10/19/Jv2RkCiO3nd9cGH.png)

![img](https://i.loli.net/2021/10/19/yZnUkqgJdtirGY8.png)

![img](https://i.loli.net/2021/10/19/yZnUkqgJdtirGY8.png)

![img](https://i.loli.net/2021/10/19/nUBM3c1eNiCLp47.png)

可以看到，ThreadLocalMap底层是一个数组，数组中元素类型是Entry类型
set操作是向当前线程的ThreadLocal.ThreadLocalMap类型的成员变量threadLocals中设置值，key是this，value是我们指定的值
注意，这里传的this代表的是那个ThreadLocal类型的变量（或者说叫对象）
也就是说，每个线程都维护了一个ThreadLocal.ThreadLocalMap类型的对象，而set操作其实就是以ThreadLocal变量为key，以我们指定的值为value，最后将这个键值对封装成Entry对象放到该线程的ThreadLocal.ThreadLocalMap对象中。每个ThreadLocal变量在该线程中都是ThreadLocal.ThreadLocalMap对象中的一个Entry。既然每个ThreadLocal变量都对应ThreadLocal.ThreadLocalMap中的一个元素，那么就可以对这些元素进行读写删除操作。

### get方法

get()方法就是从当前线程的ThreadLocal.ThreadLocalMap对象中取出对应的ThreadLocal变量所对应的值
同理，remove()方法就是清除这个值
用图形表示的话，大概是这样的：

![img](https://i.loli.net/2021/10/19/xgGWeAKtafH9UZ6.png)

或者是这样

![img](https://i.loli.net/2021/10/19/fv7YXSIjJRidcgx.png)