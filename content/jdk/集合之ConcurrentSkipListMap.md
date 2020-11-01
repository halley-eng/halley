---
title: "集合之ConcurrentSkipListMap"
date: 2020-11-02T00:53:30+08:00
draft: false
---

## 结构图

### 总图 
       
   ![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/jdk/skiplist/skiplist.png)


下面对几个接口重点说明: 

1. SortedMap: 
    提供排序器查询、区间视图、头尾访问等
    ![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/jdk/skiplist/sortedmap.png)

2. ConcurrentMap:
   1. 不支持null值;
       1. 所以重写了Map接口中相关默认实现为不实现
   2. 提供线程安全和原子保证;
       1. 所以重写了Map接口中相关默认实现
        ![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/jdk/skiplist/concurrentmap.png)
    
3. ConcurrentNavigableMap
    1. 重写了NavigableMap接口方法的返回类型为ConcurrentNavigableMap
    ![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/jdk/skiplist/navigablemap.png)


### 源码解析

1. [ ] 数据结构
1. [x] 查询
2. [x] 新增
3. [x] 删除
4. [ ] 区间查询
5. [ ] 迭代器


#### 查询

注意两种情况下找不到节点: 
1. 当前节点为null, 不可能匹配key;
2. 前缀节点为header节点, 则没有找到前缀, key太大了, 不可能匹配key



```java
     /**
     * Gets value for key. Almost the same as findNode, but returns
     * the found value (to avoid retries during re-reads)
     *
     * @param key the key
     * @return the value, or null if absent
     */
    private V doGet(Object key) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            // 找前缀;
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                // 找不到条件1 : 当前节点为null, 不可能匹配key;
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;

                // todo: 检测一致性读的目的是什么?  failed fast
                if (n != b.next)                // inconsistent read
                    break;
                if ((v = n.value) == null) {    // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                // 前缀被删除了, 重新去找前缀;
                if (b.value == null || v == n)  // b is deleted
                    break;


                // 找到的节点正好指向目标节点;
                if ((c = cpr(cmp, key, n.key)) == 0) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    return vv;
                }

                // 找不到条件2: 前缀节点为header节点, 则没有找到前缀, key太大了。
                // 证明之前查到的前缀是header节点, 比当前节点还要大;
                // 说明比最小的那个节点还要小, 肯定找不到目标节点了;
                if (c < 0)
                    break outer;

                // 继续右边找;
                b = n;
                n = f;
            }
        }
        // 如果没有找到节点, 将会走到这里;
        return null;
    }
```

依赖下面的找前缀函数:

```java
    private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
        if (key == null)
            throw new NullPointerException(); // don't postpone errors
        for (;;) {  // 并发问题会重新开始

            // 在当前层横向移动;
            for (Index<K,V> q = head, r = q.right, d;;) {
                if (r != null) {
                    Node<K,V> n = r.node;
                    K k = n.key;
                    if (n.value == null) {
                        // 如果发现空节点, 将该节点删除;
                        if (!q.unlink(r))
                            break;           // restart
                        r = q.right;         // reread r
                        continue;
                    }

                    // 横向移动, 跳过小于自己的右节点;
                    // 如果比当前节点的右邻居还大, 那么直接往右移动;
                    if (cpr(cmp, key, k) > 0) {
                        q = r;
                        r = r.right;
                        continue;
                    }
                }

                // 纵向移动/直接返回
                // 到这里证明, 右节点其实比自己大;
                // 其实
                if ((d = q.down) == null)
                    return q.node;  // 找到想要的点;
                // 继续往下移动, 在下一层继续上面的步骤:
                // 1. 横向移动,
                //         1.1 跳过小于自己的右节点;
                //         1.2 遇到比较大的右边界, 开始往下探, 去更细粒度的查询;
                // 2. 直到无法往下探;
                q = d;
                r = d.right;
            }
        }
    }

```


#### 新增

核心流程

1. 找前缀b, 其后缀为n
2. 创建新节点z, 并插入到b和n之前
3. 创建索引层

```java

    /**
     * 跳表插入流程分析
     * https://www.cnblogs.com/wei-zw/p/8810135.html
     *
     * 1. 每次添加最多一层索引
     * 2. 先
     *
     *
     * Main insertion method.  Adds element if not present, or
     * replaces value if present and onlyIfAbsent is false.
     * @param key the key
     * @param value the value that must be associated with key
     * @param onlyIfAbsent if should not insert if already present
     * @return the old value, or null if newly inserted
     */
    private V doPut(K key, V value, boolean onlyIfAbsent) {
        Node<K,V> z;             // added node
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            // 找到key节点的前缀;
            // 前缀节点是三元组节点的第一个
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                if (n != null) {
                    Object v; int c;
                    Node<K,V> f = n.next;
                    if (n != b.next)               // inconsistent read
                        break;
                    if ((v = n.value) == null) {   // n is deleted
                        n.helpDelete(b, f);
                        break;
                    }
                    if (b.value == null || v == n) // b is deleted
                        break;
                    if ((c = cpr(cmp, key, n.key)) > 0) {
                        b = n;  // 当前节点升级为下一个节点
                        n = f;  // 孙子节点升级为当前节点
                        continue;
                    }
                    if (c == 0) {
                        if (onlyIfAbsent || n.casValue(v, value)) {
                            @SuppressWarnings("unchecked") V vv = (V)v;
                            return vv;
                        }
                        break; // restart if lost race to replace value
                    }
                    // else c < 0; fall through
                }

                // 新建节点 并插入到前缀节点的后面;
                z = new Node<K,V>(key, value, n);
                // 在b和n之间插入z
                if (!b.casNext(n, z))   // 如果插入失败将会重试
                    break;         // restart if lost race to append to b
                break outer;
            }
        }

        // 下面是插入后，
        int rnd = ThreadLocalRandom.nextSecondarySeed();
        if ((rnd & 0x80000001) == 0) { // test highest and lowest bits
            int level = 1, max;
            // 从右边连续的几个1 就是有几层索引;
            while (((rnd >>>= 1) & 1) != 0)
                ++level;

            // 旧层索引行;
            Index<K,V> idx = null;
            HeadIndex<K,V> h = head;
            // level小于最大层, 创建level进行创建层;
            if (level <= (max = h.level)) {
                for (int i = 1; i <= level; ++i)
                    // 对节点z创建索引, 并指向下一层节点idx,
                    // 但是这里只进行了纵向连接, 没有横向连接;
                    idx = new Index<K,V>(z, idx, null);
            }
            else { // try to grow by one level
                // 增加索引层;
                level = max + 1; // hold in array and later pick the one to use
                @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                    (Index<K,V>[])new Index<?,?>[level+1];
                // 从底层创建纵向的一列索引
                for (int i = 1; i <= level; ++i)
                    idxs[i] = idx = new Index<K,V>(z, idx, null);
                //
                for (;;) {
                    h = head;
                    int oldLevel = h.level;
                    // 当前线程没有拿到升级权限 todo: 为什么?
                    if (level <= oldLevel) // lost race to add level
                        break;
                    //
                    HeadIndex<K,V> newh = h;
                    Node<K,V> oldbase = h.node;
                    // 旧层到新层之间的首列创建 HeadIndex;
                    for (int j = oldLevel+1; j <= level; ++j)
                        newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                    // 将最上层的头节点指针h指向newh
                    if (casHead(h, newh)) {
                        h = newh;
                        // 原来的旧header层
                        idx = idxs[level = oldLevel];
                        break;
                    }
                }
            }
            // 找到插入点并拼接之;
            // find insertion points and splice in
            // level 为将新插入节点需要插入的索引层数
            splice: for (int insertionLevel = level;;) {
                // h为新的最高层; 坑
                int j = h.level;
                // 从新最高层逐渐往下走;  t/idx为老的最高索引层
                for (Index<K,V> q = h, r = q.right, t = idx;;) {
                    if (q == null || t == null)
                        break splice;
                    if (r != null) {
                        Node<K,V> n = r.node;
                        // compare before deletion check avoids needing recheck
                        int c = cpr(cmp, key, n.key);
                        // 右边节点被其他线程删除了, 当前线程帮助其删除后;
                        if (n.value == null) {
                            if (!q.unlink(r))
                                break;
                            r = q.right;
                            continue;
                        }
                        // 新插入节点较大, 将往右移动;
                        if (c > 0) {
                            q = r;
                            r = r.right;
                            continue;
                        }
                    }

                    // 找到新插入节点小于等于的那个节点(在树中的右边界)

                    // 当前的t为插入索引层
                    if (j == insertionLevel) {

                        //将t插入到q,r之间失败则重新开始
                        if (!q.link(r, t))
                            break; // restart

                        //插入成功，如果t的值为null，则需要删除
                        if (t.node.value == null) {
                            findNode(key);
                            break splice;
                        }

                        //如果到最低层则跳出外层循环
                        if (--insertionLevel == 0)
                            break splice;
                    }

                    // 促使t指向的是插入层的头索引; j是其高度
                    if (--j >= insertionLevel && j < level)
                        t = t.down;

                    // 当前Focus的节点 往下移动一层;
                    q = q.down;
                    r = q.right;
                }
            }
        }
        return null;
    }

```


#### 删除

```java
    final V doRemove(Object key, Object value) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                if (n != b.next)                    // inconsistent read
                    break;
                if ((v = n.value) == null) {        // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n)      // b is deleted
                    break;
                if ((c = cpr(cmp, key, n.key)) < 0)
                    break outer;
                if (c > 0) {
                    b = n;
                    n = f;
                    continue;
                }
                if (value != null && !value.equals(v))
                    break outer;

                // 将值输出为Null 表示为删除
                if (!n.casValue(v, null))
                    break;
                // todo: 这里marker的作用是 ?
                if (!n.appendMarker(f) || !b.casNext(n, f))
                    findNode(key);                  // retry via findNode
                else {
                    findPredecessor(key, cmp);      // clean index
                    if (head.right == null)
                        tryReduceLevel();
                }
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
        }
        return null;
    }

```



