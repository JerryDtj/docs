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

软引用是一种可有可无的引用,对于这部分对象,jvm会再系统发生溢出前进行回收.

当软引用没有被回收之前,是随时可以使用的.

```java
Car car = new Car();
List<Car> carList = new ArrayList();
//对于carList来说,car就是carList的软引用.
```

