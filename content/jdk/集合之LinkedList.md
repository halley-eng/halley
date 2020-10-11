---
title: "集合之LinkedList"
date: 2020-10-11T20:58:34+08:00
draft: false
---



![graph](http://assets.processon.com/chart_image/5f82b87307912906db1e6589.png)



### 核心点

1. [x] 链表数据结构;
2. [x] 链表基本操作; 
3. [x] 迭代器
4. [x] 位置定位


### 链表数据结构;

双链表包含 prev 和 next 两个指针; 

```java

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }

```


### 双链表基本操作

#### 头部新增

1. 缓存旧头节点: f
2. 创建新节点并升级为头节点 first
    1. 新节点next指针指向旧的头节点
3. 处理旧头节点: f
    1. null : 新节点同时更新到last
    2. 否则其pre节点指向新节点; 

```java

    /**
     * Links e as first element.
     */
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```

#### 头部删除

1. 缓存next节点
2. 清理被删除节点相关属性为null 
3. 处理next节点
    1. null : last 同步也需要改为null 保证一致性
    2. next.pre = null; // 头节点当然没有前缀;

```java
    /**
     * Unlinks non-null first node f.
     */
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
    
```

#### 尾部新增

1. 缓存旧的尾巴到 l
2. 创建新节点并升级为尾巴last
3. 处理旧的尾巴
    1. null: 本次新增到空链表, 则同时也为firt
    2. 否则旧尾巴有了新后缀 l.next = newNode;

```java
    /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

```

#### 尾部删除


1. 保存前缀到prev
2. 清理被删除的节点l
3. 处理前缀
    1. 升级为尾巴last
    2. 前缀为null
        1. 是: 没有前缀那么first = null
        2. 否: prev.next = null;


```java
   /**
     * Unlinks non-null last node l.
     */
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```

#### 相对前置插入


1. 缓存 前缀 pre
2. 创建新节点 newNode
    1. 后缀prev指针指向newNode: 之前prev指向的是pred节点;
3. 修正前缀节点 succ
    1. null: 前缀为null, 则更新first
    2. 否则指向newNode

```java
    /**
     * Inserts element e before non-null Node succ.
     */
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }

```
#### 随机删除


1. 本地化x节点的三个属性
2. 修正前缀节点 prev
    1. null : 说明当前 prev 为first节点, 则需要更新first
    2. 否则 : 更新前缀节点后缀 , 重置x节点前缀;
3. 更新后缀节点 next
    1. null : 说明当前 next 为last节点, 则需要更新last
    2. 否则 : 更新后缀节点前缀 , 重置x节点后缀;

```java
    /**
     * Unlinks non-null node x.
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }

```

### 迭代器


#### 正向迭代

相对于vector可以直接通过index查到节点, 
这里需要通过next指针来找


```java
    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;
        private Node<E> next;
        private int nextIndex;
        private int expectedModCount = modCount;

        ListItr(int index) {
            // assert isPositionIndex(index);
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }

        public boolean hasNext() {
            return nextIndex < size;
        }

        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }

        public boolean hasPrevious() {
            return nextIndex > 0;
        }

        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }

        public int nextIndex() {
            return nextIndex;
        }

        public int previousIndex() {
            return nextIndex - 1;
        }

        public void remove() {
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned)
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;
        }

        public void set(E e) {
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }

        public void add(E e) {
            checkForComodification();
            lastReturned = null;
            if (next == null)
                linkLast(e);
            else
                linkBefore(e, next);
            nextIndex++;
            expectedModCount++;
        }

        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            while (modCount == expectedModCount && nextIndex < size) {
                action.accept(next.item);
                lastReturned = next;
                next = next.next;
                nextIndex++;
            }
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```
#### 反向迭代

反向索引的next函数适配到正向索引的previous函数, 很简洁的解决了反向索引的问题

```java
    /**
     * Adapter to provide descending iterators via ListItr.previous
     */
    private class DescendingIterator implements Iterator<E> {
        private final ListItr itr = new ListItr(size());
        public boolean hasNext() {
            return itr.hasPrevious();
        }
        public E next() {
            return itr.previous();
        }
        public void remove() {
            itr.remove();
        }
    }

```
### 位置定位

因为我们知道链表两端的指针first和last
所以找期间的任意一个元素都可以选择从任意一个方向出发;

但是如果因为知道链表总长度为size, 所以还是能够算出来
从哪一端会更近一些;


```java
    /**
     * Returns the (non-null) Node at the specified element index.
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
