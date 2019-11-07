# 深入理解jvm——3.2.3 再谈引用

## java的引用类型:

**强引用、软引用、弱引用、虚引用**.

引用强度依次递减

## 强引用

用户直接new出来的引用.如下面的例子

```java
Map<String> m = new hashMap();
```

#### 是否会被回收

不会,当内存不足时,宁愿抛出oom异常,也不会回收强引用

## 软引用

> 软引用是用来描述一些还有用但并非必须的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围进行**第二次回收**。如果这次回收还没有足够的内存，才会抛出内存溢出异常。

```java
import java.lang.ref.SoftReference;

/**
 * 软引用何时被收集
 * 运行参数 -Xmx200m -XX:+PrintGC
 * @author Jerry
 * @Date 2019/11/3 12:07 下午
 */
public class SoftReferenceDemo {

    public static void main(String[] args) throws InterruptedException {
        //100M的缓存数据
        byte[] cacheData = new byte[100 * 1024 * 1024];
        //将缓存数据用软引用持有
        SoftReference<byte[]> cacheRef = new SoftReference(cacheData);
        //将缓存数据的强引用去除
        cacheData = null;
        System.out.println("第一次GC前" + cacheData);
        System.out.println("第一次GC前" + cacheRef.get());
        //进行一次GC后查看对象的回收情况
        System.gc();
        //等待GC
        Thread.sleep(500);
        System.out.println("第一次GC后" + cacheData);
        System.out.println("第一次GC后" + cacheRef.get());

        //在分配一个120M的对象，看看缓存对象的回收情况
        byte[] newData = new byte[120 * 1024 * 1024];
        System.out.println("分配后" + cacheData);
        System.out.println("分配后" + cacheRef.get());
    }

}
```

对应的日志如下:

```tex
第一次GC前null
第一次GC前[B@1d44bcfa
[GC (System.gc())  106537K->103096K(196608K), 0.0013308 secs]
[Full GC (System.gc())  103096K->102935K(196608K), 0.0150606 secs]
第一次GC后null
第一次GC后[B@1d44bcfa
[GC (Allocation Failure)  103969K->102967K(196608K), 0.0019377 secs]
[GC (Allocation Failure)  102967K->102967K(196608K), 0.0026988 secs]
[Full GC (Allocation Failure)  102967K->102922K(196608K), 0.0070996 secs]
[GC (Allocation Failure)  102922K->102922K(196608K), 0.0018367 secs]
[Full GC (Allocation Failure)  102922K->504K(152576K), 0.0094763 secs]
分配后null
分配后null
```

从上面代码看出,第一次手动调用gc 是不会直接删除弱引用

**只要堆上的对象仅仅只被软引用所指向，并且当内存空间不足时，GC才会触发full gc回收对象的内存空间。**

注意:**Full GC 会导致STW 程序暂停.所以软引用回收比弱引用回收更加费时间**

## 弱引用

> 强度比软引用更低的一种软引用.被引用关联的对象只能生存到下一次垃圾收集发生之前.当垃圾收集器工作时,无论当前内存是否足够,都会回收掉只被弱引用关联的对象

```java
/**
 * 弱引用何时被收集
 * 运行参数 -Xmx200m -XX:+PrintGC
 * @author Jerry
 * @Date 2019/11/3 12:07 下午
 */
public class WeakReferenceDemo   {

    public static void main(String[] args) throws InterruptedException {
        //100M的缓存数据
        byte[] cacheData = new byte[100 * 1024 * 1024];
        //将缓存数据用软引用持有
        WeakReference<byte[]> cacheRef = new WeakReference(cacheData);
        System.out.println("第一次GC前" + cacheData);
        System.out.println("第一次GC前" + cacheRef.get());
        //进行一次GC后查看对象的回收情况
        System.gc();
        //等待GC
        Thread.sleep(500);
        System.out.println("第一次GC后" + cacheData);
        System.out.println("第一次GC后" + cacheRef.get());

        //将缓存数据的强引用去除
        cacheData = null;
        System.gc();
        //等待GC
        Thread.sleep(500);
        System.out.println("第二次GC后" + cacheData);
        System.out.println("第二次GC后" + cacheRef.get());
    }
}
```

日志如下:

```tex
第一次GC前[B@1d44bcfa
第一次GC前[B@1d44bcfa
第一次GC后[B@1d44bcfa
第一次GC后[B@1d44bcfa
第二次GC后null
第二次GC后null
```

从日志可以看出:弱引用比软引用的强度更低,而且弱引用是否被回收取决于这个对象有没有其他强引用指向它.(第一次gc 没有回收,强引用去除后,第二次gc 触发了回收).换句话来说就是**只要堆上的对象仅仅只被弱引用所指向，不管当前内存空间是否足够，下次GC都会回收对象的内存空间**

## 虚引用

> 也被称为幽林引用或者幻影引用.它是最弱的一种引用关系.一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来获取一个对象的实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。
>
> 虚引用和弱引用对关联对象的回收都不会产生影响，如果只有虚引用活着弱引用关联着对象，那么这个对象就会被回收。它们的不同之处在于弱引用的`get`方法，虚引用的`get`方法始终返回`null`,弱引用可以使用`ReferenceQueue`,虚引用必须配合`ReferenceQueue`使用。

`jdk`中直接内存的回收就用到虚引用，由于`jvm`自动内存管理的范围是堆内存，而直接内存是在堆内存之外（其实是内存映射文件，自行去理解虚拟内存空间的相关概念），所以直接内存的分配和回收都是有`Unsafe`类去操作，`java`在申请一块直接内存之后，会在堆内存分配一个对象保存这个堆外内存的引用，这个对象被垃圾收集器管理，一旦这个对象被回收，相应的用户线程会收到通知并对直接内存进行清理工作。