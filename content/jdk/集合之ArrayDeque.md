---
title: "集合之ArrayDeque"
date: 2020-10-14T01:49:55+08:00
draft: false
---




### 核心点

1. [x] 初始化
2. [ ] 关键函数
    1. [x] 头部新增
    2. [x] 尾部新增
    2. [x] 头部删除
    3. [x] 尾部删除
    4. [ ] 任意位置删除
3. [x] 扩缩容
4. [x] 迭代器
    1. [x] DeqIterator
    2. [x] DescendingIterator
5. [ ] DeqSpliterator

### 三种初始化方法

#### 1. 无参默认16个

```java
   public ArrayDeque() {
        elements = new Object[16];
   }
```
#### 2. 指定参数最少8个或者2的n次方个

1. 默认最小申请8个
2. 否则申请>=当前需要容量的 2的n次方个; 

```java
    public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }
    
    private void allocateElements(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY;
        // Find the best power of two to hold elements.
        // Tests "<=" because arrays aren't kept full.
        if (numElements >= initialCapacity) {
            initialCapacity = numElements;
            initialCapacity |= (initialCapacity >>>  1);
            initialCapacity |= (initialCapacity >>>  2);
            initialCapacity |= (initialCapacity >>>  4);
            initialCapacity |= (initialCapacity >>>  8);
            initialCapacity |= (initialCapacity >>> 16);
            initialCapacity++;

            if (initialCapacity < 0)   // Too many elements, must back off
                initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
        }
        elements = new Object[initialCapacity];
    }

    
```

#### 3. 创建并依赖集合填充数组


```java
  public ArrayDeque(Collection<? extends E> c) {
        allocateElements(c.size());
        addAll(c);
  }
```




### 关键代码函数

#### 头部新增

(head - 1) & (elements.length - 1) 
    表示下一个要存头部元素的索引;

重复子过程: 
1. 移动头部指针
    head = (head - 1) & (elements.length - 1)
2. 存放新元素: 
    elements[head] = e
3. 检测下一次尾部新增是否和当前节点碰撞;
   检测到head指针碰撞到tail指针, 则扩容;   

注意: 不支持 null 元素, 而LinkedList是没问题的;


```java
    public void addFirst(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[head = (head - 1) & (elements.length - 1)] = e;
        // 下一次尾部新增会与当前的head碰撞;
        if (head == tail)
            doubleCapacity();
    }
```

#### 尾部新增

tail 表示下一个要存尾部元素的索引; 

重复子过程: 

1. tail索引存放元素
   elements[tail] = e;
2. 移动tail指针:
    tail = (tail + 1) & (elements.length - 1)
3. 检测下一次尾部新增是否和当前节点碰撞;
   检测tail指针碰撞到head, 则触发扩容;    
   或者是下一次尾部新增将到达tail == head 即队列为空的状态, 则本次触发扩容;

```java
    public void addLast(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[tail] = e;
        // 下一次头部新增, 会与当前的tail碰撞;
        if ( (tail = (tail + 1) & (elements.length - 1)) == head)
            doubleCapacity();
    }

```


#### 头部删除



```java

    public E pollFirst() {
        int h = head;
        @SuppressWarnings("unchecked")
        E result = (E) elements[h];
        // Element is null if deque empty
        if (result == null)
            return null;
        elements[h] = null;     // Must null out slot
        head = (h + 1) & (elements.length - 1);
        return result;
    }

```

#### 尾部删除


```java
    public E pollLast() {
        int t = (tail - 1) & (elements.length - 1);
        @SuppressWarnings("unchecked")
        E result = (E) elements[t];
        if (result == null)
            return null;
        elements[t] = null;
        tail = t;
        return result;
    }

```


#### 任意位置删除


```java
    /**
     * Removes the element at the specified position in the elements array,
     * adjusting head and tail as necessary.  This can result in motion of
     * elements backwards or forwards in the array.
     *
     * <p>This method is called delete rather than remove to emphasize
     * that its semantics differ from those of {@link List#remove(int)}.
     *
     * @return true if elements moved backwards
     */
    private boolean delete(int i) {
        checkInvariants();
        final Object[] elements = this.elements;
        final int mask = elements.length - 1;
        final int h = head;
        final int t = tail;
        final int front = (i - h) & mask;
        final int back  = (t - i) & mask;

        // Invariant: head <= i < tail mod circularity
        if (front >= ((t - h) & mask))
            throw new ConcurrentModificationException();

        // Optimize for least element motion
        // 优化元素的最少移动;
        if (front < back) {  // 前面的都往后移动一个单位;
            if (h <= i) {
                System.arraycopy(elements, h, elements, h + 1, front);
            } else { // Wrap around
                System.arraycopy(elements, 0, elements, 1, i);
                elements[0] = elements[mask];
                System.arraycopy(elements, h, elements, h + 1, mask - h);
            }
            elements[h] = null;
            head = (h + 1) & mask;
            return false;
        } else {          // 后面的都往前移动一个单位;
            if (i < t) { // Copy the null tail as well
                System.arraycopy(elements, i + 1, elements, i, back);
                tail = t - 1;
            } else { // Wrap around
                System.arraycopy(elements, i + 1, elements, i, mask - i);
                elements[mask] = elements[0];
                System.arraycopy(elements, 1, elements, 0, t);
                tail = (t - 1) & mask;
            }
            return true;
        }
    }
```


### 扩缩容



#### 扩容 

扩容时机: 
    发生在头尾部新增时, 如果发生碰撞, 则触发 doubleCapacity 函数
    
扩容方法:
    1. 创建2倍长度的心数组;
    2. 将旧数组中的元素旋转数组旋转后(头部放在0这个位置), 重新放置在新数组中; 
       另外原地旋转还有个杂耍算法, 但是这里避免不了开辟新数组, 所以可以这么简答的处理; 

```java
    /**
     *
     * Doubles the capacity of this deque.  Call only when full, i.e.,
     * when head and tail have wrapped around to become equal.
     */
    private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p; // number of elements to the right of p  // 实际有数据的长度
        // 新容量为原来的2倍, 并创建新数组a
        int newCapacity = n << 1;
        if (newCapacity < 0) // 越界检测
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        // 从当前数组elements对应的头部 p 的位置， 拷贝到数组a中从0开始 r 个;
        System.arraycopy(elements, p, a, 0, r); 
        // 从当前数组elements对应的头部 0 的位置, 拷贝到数组a中从r开始 p 个;
        System.arraycopy(elements, 0, a, r, p); 
        // a升级为扩容后数组
        elements = a;
        // 从置头尾部位置; 
        head = 0;
        tail = n;
    }
```


#### 缩容

没有对应的方法, 手动的也没有... 


### 迭代器;



#### 正向迭代器
    
    头部光标初始化: 
        private int cursor = head;
    头部位移公式:     
        cursor = (cursor + 1) & (elements.length - 1);
            
            
```java

    private class DeqIterator implements Iterator<E> {
        /**
         * 当前索引默认为头部位置  -- 下一次返回的索引位置;
         * Index of element to be returned by subsequent call to next.
         */
        private int cursor = head;

        /**
         * Tail recorded at construction (also in remove), to stop
         * iterator and also to check for comodification.
         */
        private int fence = tail;

        /**
         * 上一次返回的索引位置;
         * Index of element returned by most recent call to next.
         * Reset to -1 if element is deleted by a call to remove.
         */
        private int lastRet = -1;

        public boolean hasNext() {
            return cursor != fence;
        }

        public E next() {
            if (cursor == fence)
                throw new NoSuchElementException();
            @SuppressWarnings("unchecked")
            E result = (E) elements[cursor];
            // This check doesn't catch all possible comodifications,
            // but does catch the ones that corrupt traversal
            if (tail != fence || result == null)
                throw new ConcurrentModificationException();
            lastRet = cursor;
            // 下一次光标的位置;
            cursor = (cursor + 1) & (elements.length - 1);
            return result;
        }

        // 删除上一次返回的索引位置上的元素;
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (delete(lastRet)) { // if left-shifted, undo increment in next()
                cursor = (cursor - 1) & (elements.length - 1);
                fence = tail;
            }
            lastRet = -1;
        }

        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            Object[] a = elements;
            int m = a.length - 1, f = fence, i = cursor;
            cursor = f;
            while (i != f) {
                @SuppressWarnings("unchecked") E e = (E)a[i];
                i = (i + 1) & m;
                if (e == null)
                    throw new ConcurrentModificationException();
                action.accept(e);
            }
        }
    }

```


#### 反向迭代器

光标初始化:
            private int cursor = tail;
光标位移公式: 
            cursor = (cursor - 1) & (elements.length - 1);
            

```java
    private class DescendingIterator implements Iterator<E> {
        /* 
         * 下一次从尾部开始返回;
         * This class is nearly a mirror-image of DeqIterator, using
         * tail instead of head for initial cursor, and head instead of
         * tail for fence.
         */
        private int cursor = tail;
        private int fence = head;
        private int lastRet = -1;

        public boolean hasNext() {
            return cursor != fence;
        }

        public E next() {
            if (cursor == fence)
                throw new NoSuchElementException();
            cursor = (cursor - 1) & (elements.length - 1);
            @SuppressWarnings("unchecked")
            E result = (E) elements[cursor];
            if (head != fence || result == null)
                throw new ConcurrentModificationException();
            lastRet = cursor;
            return result;
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (!delete(lastRet)) {
                cursor = (cursor + 1) & (elements.length - 1);
                fence = head;
            }
            lastRet = -1;
        }
    }
```
