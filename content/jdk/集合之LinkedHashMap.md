---
title: "集合之LinkedHashMap"
date: 2020-10-27T21:53:21+08:00
draft: false
---



![LinkedHashMap](http://assets.processon.com/chart_image/5f98178407912906db43dc2a.png)


### 核心点分解

1. [x] 数据结构
2. [x] 构造方法
3. [x] 链表特殊操作
3. [x] 查询
4. [x] 迭代器
4. [x] 视图




### 数据结构


```java

// 链表头指针
transient LinkedHashMap.Entry<K,V> head;
// 链表尾指针
transient LinkedHashMap.Entry<K,V> tail;
// 是否维持访问序
final boolean accessOrder;

```

### 构造方法

在HashMap的所有构造方法上, 都加上了 accessOrder 属性

```java
      /**
     * Constructs an empty insertion-ordered <tt>LinkedHashMap</tt> instance
     * with the specified initial capacity and a default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity
     * @throws IllegalArgumentException if the initial capacity is negative
     */
    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }

    /**
     * Constructs an empty insertion-ordered <tt>LinkedHashMap</tt> instance
     * with the default initial capacity (16) and load factor (0.75).
     */
    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

    /**
     * Constructs an insertion-ordered <tt>LinkedHashMap</tt> instance with
     * the same mappings as the specified map.  The <tt>LinkedHashMap</tt>
     * instance is created with a default load factor (0.75) and an initial
     * capacity sufficient to hold the mappings in the specified map.
     *
     * @param  m the map whose mappings are to be placed in this map
     * @throws NullPointerException if the specified map is null
     */
    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }

    /**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }

```
### 链表特殊操作

```java

    /**
     * 新建立的节点 通过这里同步添加到 LinkedHashMap
     */
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
        LinkedHashMap.Entry<K,V> t =
            new LinkedHashMap.Entry<K,V>(q.hash, q.key, q.value, next);
        transferLinks(q, t);
        return t;
    }

    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p);
        return p;
    }

    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
        TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
        transferLinks(q, t);
        return t;
    }

    /**
     * 删除节点钩子
     * @param e 被删除的节点
     */
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        //
        p.before = p.after = null;
        // 删除一个没有前缀的节点, 说明是head节点
        if (b == null)
            head = a; // head节点的后缀成为head
        else
            b.after = a; // 前缀指向后缀

        // 删除一个没有后缀的节点, 说明这是一个tail
        if (a == null)
            tail = b; // 前缀成为tail节点
        else
            a.before = b; // 后缀指向前缀;
    }

    /**
     * 插入新元素后, 通过{@link LinkedHashMap#removeEldestEntry(java.util.Map.Entry)}
     * 决定是否移除最老的节点
     */
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

    /**
     * 如果设置了访问顺序, 则将最新访问的节点, 从旧的位置删除, 并插入到尾部;
     * @param e
     */
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;

        if (accessOrder && (last = tail) != e) {
            // 如果新加入的节点为p, 那么b,a为null;

            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            // 当前节点为null的before为null, 则当前为头节点
            if (b == null)
                head = a;    // 头节点删除, 那么其后继成为头节点;
            else
                b.after = a; // 前缀指向后缀删除当前节点;



            // 后缀不为空, 后缀指向前缀
            if (a != null)
                a.before = b;
            else    // 后缀为空, 前缀成为链表尾巴
                last = b;

            // 这里说明之前链表为空
            if (last == null)
                head = p; // 当前节点成为头节点
            else {
                // 当前节点链接到原始的尾巴部分;
                p.before = last;
                last.after = p;
            }
            // 当前节点肯定为尾巴
            tail = p;
            ++modCount;
        }
    }

```

### 查询

#### 按value查询

从head指针开始迭代链表, 依次检测是否能找到目标对象;


```java
    public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
```


#### 按key查询

借助于 HashMap 的两个方法, 查找Node;
1. hash函数: 
2. 节点查询方法:
    final Node<K,V> getNode(int hash, Object key)

如果 accessOrder, 则调用 afterNodeAccess 方法将, 将最新访问的节点, 放在链表结尾;

```java

    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        // 触发访问钩子, 有助与实现LRU缓存
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

    /**
     * {@inheritDoc}
     */
    public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;
       if (accessOrder)
           afterNodeAccess(e);
       return e.value;
   }

```


### 迭代器

 核心是 nextNode()函数
 
 主要逻辑:
 1. 返回当前next节点, 并记录在current引用;
 2. 更新next节点为其next;
 
 比较巧妙的是使用一个引用地址 e 记录 next;
 并用 e 
 1. 更新next
 2. 更新current
 3. 返回e; 
 以上3条都只根e有关, 每一句都不会影响其他的;
 
```java


    abstract class LinkedHashIterator {
        LinkedHashMap.Entry<K,V> next;
        LinkedHashMap.Entry<K,V> current;
        int expectedModCount;

        LinkedHashIterator() {
            next = head;
            expectedModCount = modCount;
            current = null;
        }

        public final boolean hasNext() {
            return next != null;
        }

        final LinkedHashMap.Entry<K,V> nextNode() {
            // 1. 下一个节点为已存在节点;
            LinkedHashMap.Entry<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            // 2. 检测到该节点存在
            current = e;
            
            // 3. 缓存更新下一个节点到next;
            next = e.after;
            // 4. 返回当前节点;
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
    

    final class LinkedKeyIterator extends LinkedHashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().getKey(); }
    }
    

    final class LinkedValueIterator extends LinkedHashIterator
        implements Iterator<V> {
        public final V next() { return nextNode().value; }
    }

 
    final class LinkedEntryIterator extends LinkedHashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }
    }


```


### 视图

#### keySet

依次迭代 Entry 获取Key

```java
    transient Set<K>       keySet;
    transient Collection<V> values;

    public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new LinkedKeySet();
            keySet = ks;
        }
        return ks;
    }
    
    // 内部类;
    final class LinkedKeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { LinkedHashMap.this.clear(); }

        public final Iterator<K> iterator() {
            // 该迭代器每次每次只会返回key
            return new LinkedKeyIterator();
        }
        public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator()  {
            return Spliterators.spliterator(this, Spliterator.SIZED |
                                            Spliterator.ORDERED |
                                            Spliterator.DISTINCT);
        }
        public final void forEach(Consumer<? super K> action) {
            if (action == null)
                throw new NullPointerException();
            int mc = modCount;
            // 迭代链表并将 key 输入给输入函数;
            for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
                action.accept(e.key);
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }

```


#### values

依次迭代 Entry 获取值;

```java
    public Collection<V> values() {
        Collection<V> vs = values;
        if (vs == null) {
            vs = new LinkedValues();
            values = vs;
        }
        return vs;
    }

    final class LinkedValues extends AbstractCollection<V> {
        public final int size()                 { return size; }
        public final void clear()               { LinkedHashMap.this.clear(); }
        public final Iterator<V> iterator() {
            return new LinkedValueIterator();
        }
        public final boolean contains(Object o) { return containsValue(o); }
        public final Spliterator<V> spliterator() {
            return Spliterators.spliterator(this, Spliterator.SIZED |
                                            Spliterator.ORDERED);
        }
        public final void forEach(Consumer<? super V> action) {
            if (action == null)
                throw new NullPointerException();
            int mc = modCount;
            for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
                action.accept(e.value);
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
```


