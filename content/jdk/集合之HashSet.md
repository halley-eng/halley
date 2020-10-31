---
title: "集合之HashSet"
date: 2020-10-31T14:58:37+08:00
draft: false
---

![uml](http://assets.processon.com/chart_image/5f9d0674f346fb11c3cc0c75.png)

### 插入

以参数e为key, value固定为常量 PRESENT 插入到map.
1. 通过map的key的唯一性保证集合的唯一性; 
2. 因为底层是HashMap, 所以并不能保证有序; 

```java
    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }
```

### 删除

如果字典能成功删除对象o, 并且其值为PRESENT时. 

```java
    public boolean remove(Object o) {
        return m.remove(o)==PRESENT;
    }
```

### 异常处理

因为TreeSet需要依赖key的有序性来处理问题, 所以会产生下面的异常:

ClassCastException: 当输入的key不能与字段中存量的key比较时;
NullPointerException: 当输入的key为null, 并且比较器不兼容null值时，或者没有比较器时;

但是HashSet并不需要有序性保证，而且key支持null, 所以甚至没有npe异常;


### 其他

利用key在底层字典中只保证唯一性 不保证有序性, 所以TreeSet没有提供导航接口。 



