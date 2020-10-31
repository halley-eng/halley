---
title: "集合之TreeSet"
date: 2020-10-31T14:35:59+08:00
draft: false
---


![TreeSet](http://assets.processon.com/chart_image/5f9cdd8b6376891d3200c941.png)



### 核心点分解


### 插入

以参数e为key, value固定为常量 PRESENT 插入到map.
1. 通过map的key的唯一性保证集合的唯一性. 
2. 通过map的有序性保证集合的有序性. 

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

ClassCastException: 当输入的key不能与字段中存量的key比较时;

NullPointerException: 当输入的key为null, 并且比较器不兼容null值时，或者没有比较器时;

### 其他

利用key在底层字典中的有序性, 所以TreeSet也能提供相关导航接口. 



