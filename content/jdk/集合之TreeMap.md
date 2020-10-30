---
title: "集合之TreeMap"
date: 2020-10-30T22:16:46+08:00
draft: false
---



![TreeMap](http://assets.processon.com/chart_image/5f98283b5653bb30178b9e38.png)



### 核心点分解

* [x] 插入过程
* [x] 删除过程
* [ ] 导航方法
    * [x] lowerEntry
    * [x] floorEntry
    * [x] ceilingEntry
    * [x] higherEntry
* [x] 视图
    * [x] headMap
    * [x] subMap
    * [x] tailMap
* [x] 迭代器  
* [x] 查询
    * [x] firstKey
    * [x] lastKey



### 插入

两种情况下返回null 
1. 根节点root == null 
2. 遍历整颗树后依然没有发现已存在的节点

空指针异常: 
1. 当key为null, 且没有指定comparator，
    使用key对象的compareTo(T t); 
2. 存在排序器 comparator 但是其不支持; 

ClassCastException:
    
    当没有制定排序器时，会提取Key值的 Compareble接口, 如果不能cast到则会抛出对应的ClassCastException


```java

    public V put(K key, V value) {

        // 本地化根节点 root 到 t;
        Entry<K,V> t = root;
        if (t == null) {
            compare(key, key); // type (and possibly null) check
            // 初始化root节点;
            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            // 返回旧节点为null;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        // 存在Comparator
        if (cpr != null) {
            do {
                // 继续位移起始点;
                parent = t;
                // 定方向
                cmp = cpr.compare(key, t.key);
                // 比较后位移或者设置值并返回;
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);

                // 成功位移的情况下，继续循环，或者是走到了结尾, 后期插入到parent后面即可;
            } while (t != null);
        }
        // 不存在Comparator, 但是key实现了Comparable接口的情况下也可以;
        else {
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        // 以上并没有查询到已存在到节点, 所以合理可以直接插入进去;
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        // 插入节点后进行平衡性修复;
        fixAfterInsertion(e);
        size++;
        modCount++;
        // 新插入节点返回旧节点为null;
        return null;
    }

```


### 删除

主程序如下, 主要是实现返回旧节点中的旧值, 旧节点非null的情况下从树中删除; 
注意值当然可以为null;

```java
    public V remove(Object key) {
        // 查询旧值
        Entry<K,V> p = getEntry(key);
        // 没有旧值 直接返回不用删除
        if (p == null)
            return null;
        // 提取旧值 后期返回
        V oldValue = p.value;
        // 从树中删除旧值; 
        deleteEntry(p);
        // 返回旧值;
        return oldValue;
    }
```

删除过程依赖一个找后继的函数如下:

找节点t的后继(下一个大于)节点;
```java
    /**
     * Returns the successor of the specified Entry, or null if no such.
     */
    static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
        // 空节点没有后继节点;
        if (t == null)
            return null;
        // 右子树上找最小值;
        else if (t.right != null) {
            // 右子树的左边极值;
            Entry<K,V> p = t.right;
            while (p.left != null)
                p = p.left;
            return p;
        // 空右子树: 迭代父亲和当前节点直到当前节点为父亲的左孩子; 
        } else {
            // 定位父节点;
            Entry<K,V> p = t.parent;
            // 当前节点;
            Entry<K,V> ch = t;
            // loop when 当前节点为父亲的右孩子;
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            // 直到找到当前节点为父亲节点的左孩子;
            return p;
        }
    }

```

删除主程序如下:

1. 对于内部节点(有两个孩子), 找到后继节点s，并拷贝到当前节点上, 后期在执行删除s的动作;
2. 找一个非null的孩子节点并替换当前节点： 
    1. 孩子的父亲指向当前节点p的父亲
    2. p 的父亲指向原本指向当前节点的指针, 指向孩子节点;
    3. [ ] TODO: 这里是如何保证只有一个孩子节点的;
3. 没有孩子、没有父亲: 
    1. 删除root节点; 
4. 没有孩子、有父亲：
    1. 如果当前节点为黑色，则需调整红黑树
    2. 删除父亲节点上的引用；

```java

    /**
     * Delete node p, and then rebalance the tree.
     */
    private void deleteEntry(Entry<K,V> p) {
        modCount++;
        size--;

        // If strictly internal, copy successor's element to p and then make p
        // point to successor.
        // 两个孩子都是非null的节点: 后继上的值复制到当前节点，并让p指向s, 后期相当于删除后继节点s;
        if (p.left != null && p.right != null) {
            // 找后继节点;
            Entry<K,V> s = successor(p);
            // 将后继节点内容复制到 p
            p.key = s.key;
            p.value = s.value;
            // p 重新指向 s;
            p = s;
        } // p has 2 children


        // 在替换节点上启动修复;
        // Start fixup at replacement node, if it exists.
        Entry<K,V> replacement = (p.left != null ? p.left : p.right);

        // 将左右任意一个孩子提升上来;
        if (replacement != null) {
            // Link replacement to parent
            replacement.parent = p.parent;
            if (p.parent == null)
                root = replacement;
            else if (p == p.parent.left)
                p.parent.left  = replacement;
            else
                p.parent.right = replacement;

            // Null out links so they are OK to use by fixAfterDeletion.
            p.left = p.right = p.parent = null;

            // Fix replacement
            if (p.color == BLACK)
                fixAfterDeletion(replacement);
        //  没有孩子，没有父亲：p 就是root, 所以其被删除后 root == null
        } else if (p.parent == null) { // return if we are the only node.
            root = null;
        // 没有孩子, 有父亲: 所以其被删除后,
        } else { //  No children. Use self as phantom replacement and unlink.
            // 如果删除黑色节点，根据红黑树的性质，每条路径的黑色节点数目相同，这样删除后则不同，所以需要调整；
            if (p.color == BLACK)
                fixAfterDeletion(p);

            if (p.parent != null) {
                if (p == p.parent.left)
                    p.parent.left = null;
                else if (p == p.parent.right)
                    p.parent.right = null;
                p.parent = null;
            }
        }
    }
```


### 导航方法

#### lowerEntry 小于key的最大值 

```java
    public Map.Entry<K,V> lowerEntry(K key) {
        return exportEntry(getLowerEntry(key));
    }
```




```java
    final Entry<K,V> getLowerEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {

            int cmp = compare(key, p.key);
            // key更大: 让对手更逼近key
            if (cmp > 0) { 
                // 增强对手，继续查询;
                if (p.right != null)
                    p = p.right;
                // 不能增强对手， 直接返回;
                else
                    return p;
            // key更小: 降低对手使得其小于key;       
            } else {
                // 找更小的对手节点
                if (p.left != null) {
                    p = p.left;
                // 回溯父亲节点, 继续查找更小的节点;    
                } else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    // 迭代右上角的父亲
                    while (parent != null && ch == parent.left) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    // 直到左拐:  为父亲的右孩子
                    // 直到当前节点为父亲节点的右孩子; 
                    return parent;
                }
            }
        }
        return null;
    }
```

#### floorEntry



```java
    final Entry<K,V> getFloorEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            // key 更大，则需要增强对手节点
            if (cmp > 0) {
                // 可以增强对手节点，则增强之；
                if (p.right != null)
                    p = p.right;
                    
                // 不能增强之，即没有右孩子的节点;    
                else
                    return p;
                    
            //  key 更小, 则需要降低对手节点;        
            } else if (cmp < 0) {
                // 可以降低对手节点则降低之。
                if (p.left != null) {
                    p = p.left;
                // 当前子树不能降低对手节点，则回退到当前右子树的父亲节点
                } else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    while (parent != null && ch == parent.left) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    // ch为parent的右孩子或者是parent == null
                    return parent;
                }
            } else
                return p;

        }
        return null;
    }

```

#### ceilingKey 找大于等于key的最小值

key需要小于等于目标值;

```java
    final Entry<K,V> getCeilingEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            // key 更小则降低对手节点; 
            if (cmp < 0) {
                // 能降低则降低
                if (p.left != null)
                    p = p.left;
                // 否则找到之;    
                else
                    return p;
            // key 更大则增强对手节点;        
            } else if (cmp > 0) {
                // 能增强则增强;
                if (p.right != null) {
                    p = p.right;
                // 不能增强, 则回退到当前左子树的父亲;    
                } else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    // 如果是右孩子则一直回溯;
                    while (parent != null && ch == parent.right) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    // 直到当前左子树完成; 
                    return parent;
                }
            } else
                return p;
        }
        return null;
    }

```
#### higherKey 


```java

    public K higherKey(K key) {
        return keyOrNull(getHigherEntry(key));
    }

    // 找更大的或者即使小也是相差最小的;
    // 找第一个大于key的Entry
    final Entry<K,V> getHigherEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            // key较小, 需要找更大的;
            if (cmp < 0) {
                if (p.left != null)
                    p = p.left;
                 // 不能在大则找到了;   
                else
                    return p;        
            } else {
            // key较大, 需要找更小的; 
                if (p.right != null) {
                    p = p.right;
                } else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    while (parent != null && ch == parent.right) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    return parent;
                }
            }
        }
        return null;
    }
```



### 视图

视图使用如下类型作为切片框架

```java
    /**
     * @serial include
     */
    static final class AscendingSubMap<K,V> extends NavigableSubMap<K,V> {
        private static final long serialVersionUID = 912986545866124060L;
        
        // 根据原始map和左右边界情况去对map进行切片;
        AscendingSubMap(TreeMap<K,V> m, // 原始map
                        // 是否如何定义左边界
                        boolean fromStart, K lo, boolean loInclusive,
                        // 是否如何定义有边界
                        boolean toEnd,     K hi, boolean hiInclusive) {
            super(m, fromStart, lo, loInclusive, toEnd, hi, hiInclusive);
        }

        public Comparator<? super K> comparator() {
            return m.comparator();
        }

        public NavigableMap<K,V> subMap(K fromKey, boolean fromInclusive,
                                        K toKey,   boolean toInclusive) {
            if (!inRange(fromKey, fromInclusive))
                throw new IllegalArgumentException("fromKey out of range");
            if (!inRange(toKey, toInclusive))
                throw new IllegalArgumentException("toKey out of range");
            return new AscendingSubMap<>(m,
                                         false, fromKey, fromInclusive,
                                         false, toKey,   toInclusive);
        }

        public NavigableMap<K,V> headMap(K toKey, boolean inclusive) {
            if (!inRange(toKey, inclusive))
                throw new IllegalArgumentException("toKey out of range");
            return new AscendingSubMap<>(m,
                                         fromStart, lo,    loInclusive,
                                         false,     toKey, inclusive);
        }

        public NavigableMap<K,V> tailMap(K fromKey, boolean inclusive) {
            if (!inRange(fromKey, inclusive))
                throw new IllegalArgumentException("fromKey out of range");
            return new AscendingSubMap<>(m,
                                         false, fromKey, inclusive,
                                         toEnd, hi,      hiInclusive);
        }

        public NavigableMap<K,V> descendingMap() {
            NavigableMap<K,V> mv = descendingMapView;
            return (mv != null) ? mv :
                (descendingMapView =
                 new DescendingSubMap<>(m,
                                        fromStart, lo, loInclusive,
                                        toEnd,     hi, hiInclusive));
        }

        Iterator<K> keyIterator() {
            return new SubMapKeyIterator(absLowest(), absHighFence());
        }

        Spliterator<K> keySpliterator() {
            return new SubMapKeyIterator(absLowest(), absHighFence());
        }

        Iterator<K> descendingKeyIterator() {
            return new DescendingSubMapKeyIterator(absHighest(), absLowFence());
        }

        final class AscendingEntrySetView extends EntrySetView {
            public Iterator<Map.Entry<K,V>> iterator() {
                return new SubMapEntryIterator(absLowest(), absHighFence());
            }
        }

        public Set<Map.Entry<K,V>> entrySet() {
            EntrySetView es = entrySetView;
            return (es != null) ? es : (entrySetView = new AscendingEntrySetView());
        }

        TreeMap.Entry<K,V> subLowest()       { return absLowest(); }
        TreeMap.Entry<K,V> subHighest()      { return absHighest(); }
        TreeMap.Entry<K,V> subCeiling(K key) { return absCeiling(key); }
        TreeMap.Entry<K,V> subHigher(K key)  { return absHigher(key); }
        TreeMap.Entry<K,V> subFloor(K key)   { return absFloor(key); }
        TreeMap.Entry<K,V> subLower(K key)   { return absLower(key); }
    }
```



#### headmap



```java

    /**
     * @throws ClassCastException       {@inheritDoc}
     * @throws NullPointerException if {@code toKey} is null
     *         and this map uses natural ordering, or its comparator
     *         does not permit null keys
     * @throws IllegalArgumentException {@inheritDoc}
     * @since 1.6
     */
    public NavigableMap<K,V> headMap(K toKey, boolean inclusive) {
        return new AscendingSubMap<>(this,
                                     //  左边界头开始， 闭区间
                                     true,  null,  true,
                                     //  右边界到toKey结束
                                     false, toKey, inclusive);
    }

```

#### tailMap

```java
    /**
     * @throws ClassCastException       {@inheritDoc}
     * @throws NullPointerException if {@code fromKey} is null
     *         and this map uses natural ordering, or its comparator
     *         does not permit null keys
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public SortedMap<K,V> tailMap(K fromKey) {
        return tailMap(fromKey, true);
    }
    
    
    /**
     * @throws ClassCastException       {@inheritDoc}
     * @throws NullPointerException if {@code fromKey} is null
     *         and this map uses natural ordering, or its comparator
     *         does not permit null keys
     * @throws IllegalArgumentException {@inheritDoc}
     * @since 1.6
     */
    public NavigableMap<K,V> tailMap(K fromKey, boolean inclusive) {
        return new AscendingSubMap<>(this,
                                     false, fromKey, inclusive,
                                     true,  null,    true);
    }
```



#### submap

```java
    /**
     * @throws ClassCastException       {@inheritDoc}
     * @throws NullPointerException if {@code fromKey} or {@code toKey} is
     *         null and this map uses natural ordering, or its comparator
     *         does not permit null keys
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public SortedMap<K,V> subMap(K fromKey, K toKey) {
        return subMap(fromKey, true, toKey, false);
    }
```


#### 迭代器




```java
    /**
     * Base class for TreeMap Iterators
     */
    abstract class PrivateEntryIterator<T> implements Iterator<T> {
        Entry<K,V> next;
        Entry<K,V> lastReturned;
        int expectedModCount;

        PrivateEntryIterator(Entry<K,V> first) {
            expectedModCount = modCount;
            lastReturned = null;
            next = first;
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
            // 拿到当前要返回或者校验的entry
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            // 每次使用后继查询器找到下一个节点    
            next = successor(e);
            lastReturned = e;
            return e;
        }

        final Entry<K,V> prevEntry() {
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            next = predecessor(e);
            lastReturned = e;
            return e;
        }

        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            // deleted entries are replaced by their successors
            if (lastReturned.left != null && lastReturned.right != null)
                next = lastReturned;
            deleteEntry(lastReturned);
            expectedModCount = modCount;
            lastReturned = null;
        }
    }

    final class EntryIterator extends PrivateEntryIterator<Map.Entry<K,V>> {
        EntryIterator(Entry<K,V> first) {
            super(first);
        }
        public Map.Entry<K,V> next() {
            return nextEntry();
        }
    }

    final class ValueIterator extends PrivateEntryIterator<V> {
        ValueIterator(Entry<K,V> first) {
            super(first);
        }
        public V next() {
            return nextEntry().value;
        }
    }

    final class KeyIterator extends PrivateEntryIterator<K> {
        KeyIterator(Entry<K,V> first) {
            super(first);
        }
        public K next() {
            return nextEntry().key;
        }
    }

    final class DescendingKeyIterator extends PrivateEntryIterator<K> {
        DescendingKeyIterator(Entry<K,V> first) {
            super(first);
        }
        public K next() {
            return prevEntry().key;
        }
        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            deleteEntry(lastReturned);
            lastReturned = null;
            expectedModCount = modCount;
        }
    }

```


### 查询
####  firstEntry


```java

    /**
    * @throws NoSuchElementException {@inheritDoc}
    */
    public K firstKey() {
        return key(getFirstEntry());
    }
    
    /**
     * Returns the first Entry in the TreeMap (according to the TreeMap's
     * key-sort function).  Returns null if the TreeMap is empty.
     */
    final Entry<K,V> getFirstEntry() {
        Entry<K,V> p = root;
        if (p != null)
            while (p.left != null)  // 找极左节点 就是首节点;
                p = p.left;
        return p;
    }
```


#### lastEntry

找极右节点就是最后一个元素;

```java
   /**
     * @throws NoSuchElementException {@inheritDoc}
     */
    public K lastKey() {
        return key(getLastEntry());
    }


    /**
     * Returns the last Entry in the TreeMap (according to the TreeMap's
     * key-sort function).  Returns null if the TreeMap is empty.
     */
    final Entry<K,V> getLastEntry() {
        Entry<K,V> p = root;
        if (p != null)
            while (p.right != null)
                p = p.right;
        return p;
    }
```
