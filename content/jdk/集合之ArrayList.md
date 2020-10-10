---
title: "集合之ArrayList"
date: 2020-10-10T18:28:19+08:00
draft: false
---


![](http://assets.processon.com/chart_image/5f816448f346fb06e1c92ed3.png)

1. [x] 三种不同的初始化方法   
2. [x] 扩缩容机制
3. [x] null元素的特殊处理
4. [x] removeIf 基于BitSet的实现;
4. [x] 序列化和反序列化
8. [x] 迭代器之内部类 ListItr,Itr
9. [x] 视图之内部类 SubList
6. [x] 底层数组修饰符transient有什么用? 
5. [ ] ArrayListSpliterator



   transient Object[] elementData;



### 三种不同的初始化方法   

1. 指定容量初始化

* 容量为0时底层数组指向空数组; 
* 容量有效则按需申请;
* 负数容量抛出异常

```java
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

```
2. 默认空数组初始化
```java
   public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
当第一个元素添加的时候, 
会触发申请数组大小为 DEFAULT_CAPACITY == 10

```java
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    // 触发申请数组大小DEFAULT_CAPACITY
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```

所以, 这里其实是有一个延迟效果, 可以有效避免申请数组然后并没有add
的情况, 但是代价是每次add 或者 ensureCapacityInternal的时候都要检测底层数组引用是否是 DEFAULTCAPACITY_EMPTY_ELEMENTDATA


3. 依赖其他集合实现

这里就可以看到Collection接口的toArray方法作用了, 
我们直接用其返回的数组作为ArrayList的底层数组; 


特殊情况是
* 非 Object[] 转换之;
  * 假如我们有 1 个Object[]数组，并不代表着我们可以将Object对象存进去，这取决于数组中元素实际的类型
  * 所以这种情况集合的类型其实是Object;
  * 解决方法是把目标数组在复制一遍, 并且新的类型为Object
* 空数组还是引用到自身的空数组实现(注意不是默认数组) 

```java
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

### 扩缩容机制

#### 扩容

还是以add方法为例

1. 游标移动前通过ensureCapacityInternal函数实现
   目标元素有效或者说容量足够;
```java
    public boolean add(E e) {

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

```

2. 如果目前底层函数是无参构造函数构建出来的则设置即将扩容为默认容量
```java
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

```

3. 更新修改标记, 如果确实不够才触发扩容;

```java
   private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

4. 实际扩容

实际扩容1.5倍, 当然最大不能超过Integer.MAX, 
并且优先设置为MAX_ARRAY_SIZE, 后者会距离最大整数8个距离
为了避免 OutOfMemoryError


```java
    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

#### 缩容

将底层数组容量缩小为实际占用的容量, 需要程序主动调用. 
目标数组可能为:
1. 空数组
2. 容量恰当正好的新数组; 
3. 容量恰当正好的原数组,  当size == element.length

```java
    /**
     * Trims the capacity of this <tt>ArrayList</tt> instance to be the
     * list's current size.  An application can use this operation to minimize
     * the storage of an <tt>ArrayList</tt> instance.
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```



### NULL元素的特殊处理

1. 新增或者设置的时候不用关心, 不去检测报异常即可

```java
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

2. 删除的时候需要特殊处理;

如下第一个分支代码就是删除null元素, 会扫描底层数组,找到第一个null, 并使用fastRemove(index)删除;

注意: 
1. 该函数只是删除首个匹配对象, 但是list本身是允许重复的
2. 所以 **删除某个对象后, 并不代表里面不存在该对象了**  
3. 可以使用removeIf函数, 确保其被删除; 


```java
    public boolean remove(Object o) {
        // 删除null元素
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```

### removeIf实现

1. 使用输入函数检测底层数组的每一个元素, 
   并将检测结果存储在BitSet中;
2. 双指针分别指向下一个写入的位置和待搬迁的位置
   并迭代搬迁
3. 清理数组尾巴;   
   
   
```java

    @Override
    public boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        // figure out which elements are to be removed
        // any exception thrown from the filter predicate at this stage
        // will leave the collection unmodified
        int removeCount = 0;
        final BitSet removeSet = new BitSet(size);
        final int expectedModCount = modCount;
        final int size = this.size;
        // 收集需要修改的元素;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            @SuppressWarnings("unchecked")
            final E element = (E) elementData[i];
            if (filter.test(element)) {
                removeSet.set(i);
                removeCount++;
            }
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

        // shift surviving elements left over the spaces left by removed elements
        final boolean anyToRemove = removeCount > 0;
        if (anyToRemove) {
            // 1. 搬迁旧数组;
            final int newSize = size - removeCount;
            for (int i=0, j=0; (i < size) && (j < newSize); i++, j++) {
                i = removeSet.nextClearBit(i);
                elementData[j] = elementData[i];
            }
            // 2. 清扫尾部空间;
            for (int k=newSize; k < size; k++) {
                elementData[k] = null;  // Let gc do its work
            }
            // 3. 更新列表容量;
            this.size = newSize;
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            modCount++;
        }

        return anyToRemove;
    }
    
    
```
### 序列化

#### 反序列化

1. 读对象标记位
2. 读数组容量;
3. 申请数组空间;
4. 依次读数组元素;

```java
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```

#### 序列化

1. 写数组标记位
2. 写输入大小
3. 依次写入数组元素对象


```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```


### 迭代器之内部类 ListItr, Itr

 和vector实现原理一致, vector使用syschorize实现并发安全, 
 这里使用修改标记保证不存在并发更新; 

### 视图之内部类SubList

和AbstractList逻辑一致, 这里会同步原列表的modCount
并同步更新之;


### 底层数组修饰符transient有什么用? 
 
```java
    transient Object[] elementData;
```
   
transient 装饰字段 elementData 标示其不会被自动序列化

原因如下: 

    ArrayList在序列化的时候会调用writeObject，直接将size和element写入ObjectOutputStream；反序列化时调用readObject，从ObjectInputStream获取size和element，再恢复到elementData。
       
为什么不直接用elementData来序列化，而采用上诉的方式来实现序列化呢？
    
    原因在于elementData是一个缓存数组，它通常会预留一些容量，等容量不足时再扩充容量，那么有些空间可能就没有实际存储元素，采用上诉的方式来实现序列化时，就可以保证只序列化实际存储的那些元素，而不是整个数组，从而节省空间和时间。
       
       



### 参考链接

1. [c.toArray bug 问题](https://blog.csdn.net/GuLu_GuLu_jp/article/details/51457492)
2. [序列化](https://blog.csdn.net/zero__007/article/details/52166306)
