---
title: "集合之HashMap"
date: 2020-10-26T00:28:37+08:00
draft: false
---



[TOC]



#### 关键知识点分解

1. [x] 数据结构
2. [x] 构造方法
3. [x] 插入过程
    1. [x] hash定位算法;
    2. [x] 红黑树插入
    3. [x] 链表插入
        1. [x] 链表插入触发树化过程;
4. [ ] 删除过程
4. [ ] 迭代器    
5. [ ] LinkedHashMap 钩子函数



#### uml


![hashmap](http://assets.processon.com/chart_image/5f8dbe2d07912906db314a57.png)
    
#### 数据结构

核心属性如下:
```java
transient Node<K,V>[] table;
transient Set<Map.Entry<K,V>> entrySet;
transient int size;
transient int modCount;
int threshold;
final float loadFactor;
```


#### 4个构造方法

1. 容量 + 负载因子构造
    1. initialCapacity : 检测有效性及赋值;
    2. loadFactor : 检测有效性及赋值;  
    3. 设置初始阈值为 最小满足的2次方数; 
        
    ```java
    public HashMap(int initialCapacity, float loadFactor) {
        // 检测容量有效性; 太小则报错;
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        // 太大则取最大值;                                       
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        
        // 检测负载因子的有效性;    
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);                                                                                      
        this.loadFactor = loadFactor;
        
        // 设置初始阈值
        this.threshold = tableSizeFor(initialCapacity);
    }
    ```
    最小满足的二次方数及后置处理
    1. 最小满足的二次方数, 容量 - 1, 在求二次方数, 最后在+1
        1. 小于0 则取 1;
        2. [ ] 这里和ArrayDeque不同: 
              [ArrayDeque](https://halley-eng.github.io/halley/jdk/%E9%9B%86%E5%90%88%E4%B9%8Barraydeque/)
    2. 不能超过最大容量, 否则取最大容量; 
        
    ```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    ```

2. 指定容量 + 默认负载因子

    默认负载因子是0.75
    ```java
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    ```
3. 默认容量 + 默认负载因子
    ```java
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }    
    ```

    ```java
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            // 使用参数map扩展信息 -- 等待put数据的时候会触发table初始化
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            // 对旧map进行扩容升级; 
            else if (s > threshold)
                resize();

            // 迭代旧map, 依次迁移到新字典;     
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
    ```

#### 新增 Put


```java
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
 }
```

其中先会对key取hash值

##### Hash定位算法

```java
   static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
1. 对于 null 的 key, hash 值取 0 

    后期查找会通过以下表达式 锁定一个Node

    ```java
     (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
    ```

    此时 (k = p.key) == null, key == null , 并且 (null == null) == true

    所以后期能够查出来Node;
    
##### Put 主流程 -- 负载均衡数组、链表、红黑树插入; 

1. 初始化表格 tab
2. 桶插入：
    1. 未 冲 突 : 数组(空桶) 直插
    2. 冲突处理: 其他插入或者获取已存节点;
        1. 冲突   在数组: 数组获取已存节点;
        2. 冲突不在数组上, 根据数组上节点类型再路由: 
            1. 红黑树节点: 红黑树插入或者获取已经存在的节点;
            1. 链表节点: 链表插入或者获取已经存在的节点;
                1. 触发树化过程
    3. 如果查找到已存节点则根据插入参数决定是否替换之; 
3. 回调处理：

    ```java
        final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {
                       
            Node<K,V>[] tab; Node<K,V> p; int n, i;
            // 1. tab 为 null, 则初始化, 并更新长度 n;
            //    确保表格已经初始化; 
            if ((tab = table) == null || (n = tab.length) == 0)
                n = (tab = resize()).length; // n 为数组长度

            // 2. 插入或者重新绑定表格中已存在的该key对应的节点;  
            
            // 2.1 空桶未冲突直接插入, 插入数组中:
            //     check 桶是否为null, 如果是则插入到桶中;
            if ((p = tab[i = (n - 1) & hash]) == null)  //  判断该bucket是否为空 顺便计算下标
                tab[i] = newNode(hash, key, value, null);   // bucket为null, 直接放在数组位置上;
            
            // 2.2 非空桶则会冲突, 则将插入链表或者是红黑树中, 或者是替换数组、链表、红黑树中节点;
            //     数组不为null, 则可能存在与数组、链表、红黑树，或者需要插入链表和红黑树;
            else {  // 该bucket 不为空插入原有链表头部或者是红黑树

                // 2.2.1 插入或者获得已经存在的节点;
                Node<K,V> e; K k;
                // 2.2.1.1 存在于数组中: 新key和原有的key equal或者为null或者引用相等, 直接替换
                //        该节点可能是链表或者是红黑树的头部; 
                if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                    e = p;
                // 2.2.1.2 插入红黑树 或者 获取链表中的节点;
                else if (p instanceof TreeNode)
                    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
                // 2.2.1.3 插入链表或者获取链表节点;
                else {
                    // 遍历链表直到到末尾, binCount为链表的长度;
                    for (int binCount = 0; ; ++binCount) { // 统计链表长度;
                        // 检测是否到达链表结尾, 是则插入; 
                        if ((e = p.next) == null) {
                            // 插入到链表结尾;
                            p.next = newNode(hash, key, value, null);
                            // 加上自己够8个, 就要转换成红黑树
                            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                                treeifyBin(tab, hash);  //
                            break;
                        }
                        // 出现相等的key退出
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                            break;
                        //  移动链表指针;
                        p = e;
                    }
                }

                // 3.2 得到已经存在的节点, 则替换值, 并返回;
                if (e != null) { // existing mapping for key
                    V oldValue = e.value;
                    // 无条件替换null值, 按配置替换旧值;
                    if (!onlyIfAbsent || oldValue == null)
                        e.value = value;
                    afterNodeAccess(e); // 为LinkHashMap提供钩子;
                    return oldValue;
                }
            }
            // 如果没有因为已经存在节点并提前跳出, 则会继续往下走;

            // 4. 无条件更新标记位置;
            ++modCount;
            // 5. 累计大小, 并扩容;
            if (++size > threshold)
                resize();
            // 6. 调用节点插入后置回调;
            afterNodeInsertion(evict);
            return null;
        }

    ```


##### 树插入

因为红黑树是在构建在基础的树结构上面的 
所以开始依然是二叉树的插入过程

1. 根据hash值确定下沉方向 dir;
    1. hash值冲突的情况下, 使用key类名或者是System.identityHashCode(a) 确定方向;


```java
        /**
         * Tree version of putVal.
         */
        final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;
            // 1. 根据hash值确定下沉方向dir;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                // 1. 根据hash值确定方向: 
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                // 2. hash值相等, 判定是否找到key:
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                // 3. hash值冲突, 通过comparable确认方向:    
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    // 3.1 去左右查找, 如果查找到则返回:
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    // 3.2 保底方案是通过比较类名和System.identityHashCode(a), 得到方向;
                    dir = tieBreakOrder(k, pk);
                }
                // 暂存当前节点;
                TreeNode<K,V> xp = p;
                // 从当前节点 P 和 方向 dir 移动到新节点 p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    // 链接到新节点后面; 
                    Node<K,V> xpn = xp.next;
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn); // 新节点插入原节点后面; 
                    // 新节点插入到树上;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    // 新节点的前缀接入前缀节点; 
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    // 保证红黑树的平衡性策略;     
                    moveRootToFront(tab, balanceInsertion(root, x));

                    return null;
                }
            }
        }


```


##### 红黑树插入过程:


###### 红黑树的5个特性

    1. 节点是红色或黑色。
    2. 根是黑色。
    3. 所有叶子都是黑色（叶子是NIL节点）。
    4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
       注意相反却是可以有两个连续的红色节点; 
    5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

    以上特性确保: 
        从根节点到叶子节点的最长可能路径(黑红节点交叉的情况) **不超过**, 最短可能路径(两个连续的黑色节点)的两倍长   
   
   
    性质一、三:  不会变;
    性质二：  
    性质四:  只在增加红色节点、重绘黑色节点为红色，或做旋转时受到威胁。
    性质五： 只在增加黑色节点、重绘红色节点为黑色，或做旋转时受到威胁。
    
   
###### 红黑树插入

balanceInsertion 处在一个循环子过程之中:

1. 循转子过程共分成五种情况
1. 前两种情况后退出循环
2. 后三种情况后触发颜色改变或者旋转;

###### 二种情况下会退出该循环:

前两种情况() 对应维基百科的 情形一和情形二: 

1. x 父亲节点xp为null
    1. x 改为黑色即可, 常见插入到一个空树中;
2. x 父亲节点xp为黑色
    1. 不影响性质二:  根是黑色;
    2. 不影响性质四： 红色节点必须有两个黑色节点;
    3. 不影响性质五: 每条简单路径上都有相同数目的黑色节点;
        1. 虽然新增的红色节点替换了, 原本的黑色的叶子节点, 但是同时它也新增了新的黑色叶子节点, 所以当前路径上的黑色节点数目是不变的; 
            1. 黑色节点什么时候会新增呢 ?
                在红色节点下面, 新增红色节点, 将会重绘前者为黑色节点;
    
3. x 祖先节点为null
    这种情况应该不存在
   
   
   
###### 旋转逻辑

下面树形逻辑, 共产生三条有效路径,对应三种情况: 

1. 父亲是左孩子(不是情形二, 则已经确保父亲节点xp为红色)
    所以下面会根据叔叔节点, 确定下一步处理方案:
    
    下面两条是选择执行(if-else):
    1. 情形三: 红色叔叔节点(好叔叔):  
            父亲和叔叔节点染成黑色, 祖父节点xpp染成红色, 在递归处理祖父节点xpp
    2. 黑色叔叔节点(环叔叔):
       再根据当前节点再父亲节点下面的位置, 决定旋转方案;
       注意下面两种情况是顺序执行(step-by-step):
        1. 情形四: 如果x为xp的右孩子: 
            1. 先左旋转(调整新节点x和父亲节点xp的位置), 这样左旋后以xp的角度来看, 其为父亲节点x的左孩子
              ![左旋](https://zh.wikipedia.org/wiki/File:Red-black_tree_insert_case_4.png)
        2. 情形五: x为xp的左孩子    
            1. 右旋转: 
               切换当前父亲节点xp为黑色, 祖先节点xpp为黑色, 并有旋转xpp
   

```java

        // 根节点为root, 新插入的节点为 x;
        static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            // 1. 新插入的节点为红节点;
            x.red = true;
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
            
                // 1.1 获取非NULL的父亲节点; 
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                } 
                // 1.2 获取非NULL的祖先节点, 并且父亲节点为红色; 
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                
                // 1.3 父亲节点为祖先节点的左孩子, 当前节点在祖先节点的左子树; 
                //     左孩子的父亲
                if (xp == (xppl = xpp.left)) {
                
                    // 1.3.1 祖先节点的右孩子 ( 叔叔节点 ) 为红色; 
                    //       拥有一个红右孩子;    --- 好爸爸和好叔叔
                    if ((xppr = xpp.right) != null && xppr.red) {
                        // 从当前节点(红色)开始 往上层回溯, 将其转化为黑红; 
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        // 并再次从红色祖先节点出发;
                        x = xpp;
                    }
                    else {
                    // 1.3.2 祖先节点的右孩子 (叔叔节点) 为 null 或者为黑色; 
                        
                        // 当前节点为父亲节点的右孩子;  --- 好爸爸和坏叔叔; 
                        if (x == xp.right) {  // 右新节点 有好爸爸和坏叔叔; 
                            // 左旋:  当前节点变为坏叔叔的左孩子;  当前节点x更新为父亲节点; 
                            //       左旋好爸爸; 
                            root = rotateLeft(root, x = xp);
                            // 更新上游节点xp xpp
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        // 染色并右旋, 如果上游两层节点不为null, 则旋转之; 
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }

                else {
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        // 
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
                
            }
        }

```

2. 左旋 


```java
/*************对红黑树节点x进行左旋操作 ******************/
/*
 * 左旋示意图：对节点x进行左旋
 *     p                       p
 *    /                       /
 *   x                       y
 *  / \                     / \
 * lx  y      ----->       x  ry
 *    / \                 / \
 *   ly ry               lx ly
 * 左旋做了三件事：
 * 1. 将y的左子节点赋给x的右子节点,并将x赋给y左子节点的父节点(y左子节点非空时)
 * 2. 将x的父节点p(非空时)赋给y的父节点，同时更新p的子节点为y(左或右)
 * 3. 将y的左子节点设为x，将x的父节点设为y
 */
private void leftRotate(RBNode<T> x) {
    //1. 将y的左子节点赋给x的右子节点，并将x赋给y左子节点的父节点(y左子节点非空时)
    RBNode<T> y = x.right;
    x.right = y.left;

    if(y.left != null) 
        y.left.parent = x;

    //2. 将x的父节点p(非空时)赋给y的父节点，同时更新p的子节点为y(左或右)
    y.parent = x.parent;

    if(x.parent == null) {
        this.root = y; //如果x的父节点为空，则将y设为父节点
    } else {
        if(x == x.parent.left) //如果x是左子节点
            x.parent.left = y; //则也将y设为左子节点
        else
            x.parent.right = y;//否则将y设为右子节点
    }

    //3. 将y的左子节点设为x，将x的父节点设为y
    y.left = x;
    x.parent = y;       
}


```

3. 右旋:

```java
        static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                l.right = p;
                p.parent = l;
            }
            return root;
        }
```



#### 链表插入

注意初始化的时候 p 为数组中的节点, 其肯定是和key不匹配的;
所以如下循环子过程会直接 获取并检测 p.next; 


```java
                // 遍历链表直到到末尾, binCount为链表的长度;
                for (int binCount = 0; ; ++binCount) {
                    // 获取下一个可能匹配的节点e, 
                    //     如果获取不到,则证明检测到链表尾巴, 则执行插入链表的过程; 
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 加上自己够8个, 就要转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);  //
                        break;
                    }
                    // 出现相等的key则触发退出;
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                        
                    // 移动链表指针: 相当于 p = p.next;
                    p = e;
                }
```

链表转红黑树的过程如下: 

```java
    /**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 表长度小于64 直接扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();

        else if ((e = tab[index = (n - 1) & hash]) != null) {
            // 将Node节点替换为树节点
            TreeNode<K,V> hd = null, tl = null;
            do {
                // 将Node转换为TreeNode
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p; // 更新头节点;
                else {
                    p.prev = tl;    // 将当前节点和上一个节点链接起来
                    tl.next = p;
                }
                tl = p; // 更新上一次插入的树节点;
            } while ((e = e.next) != null);

            // 树根节点更新到数组的位置;
            if ((tab[index] = hd) != null)
                hd.treeify(tab);    //
        }
    }
    
        


```

通过桶中的第一个节点触发, 链表转红黑树的过程：

1. 迭代链表, 每个节点
    1. 如果不曾初始化树, 升级为root节点;
    2. 根据root节点, 插入树中
        1. 确定位移方向， 不断逼近叶子节点;
        2. 插入叶子节点;
        3. 平衡性均衡;


```java

        // from java.util.HashMap.TreeNode

         /**
         * Forms tree of the nodes linked from this node.
         * @return root of tree
         */
        final void treeify(Node<K,V>[] tab) {
            TreeNode<K,V> root = null;
            for (TreeNode<K,V> x = this, next; x != null; x = next) {
                // 缓存下一个节点;
                next = (TreeNode<K,V>)x.next;

                x.left = x.right = null;
                // 初始化根节点为当前节点;
                if (root == null) {
                    x.parent = null;
                    x.red = false;
                    root = x;
                }
                // 二叉树插入过程 + 红黑树调整
                else {
                    K k = x.key;
                    int h = x.hash;
                    Class<?> kc = null;
                    // 
                    for (TreeNode<K,V> p = root;;) {
                        int dir, ph;
                        K pk = p.key;
                        if ((ph = p.hash) > h)
                            dir = -1;
                        else if (ph < h)
                            dir = 1;
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);

                        TreeNode<K,V> xp = p;
                        // 位移并确定是否找到叶子节点, 即插入位置;
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            x.parent = xp;
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            root = balanceInsertion(root, x);
                            break;
                        }
                    }
                }
            }
            moveRootToFront(tab, root);
        }
```

#### 迭代器

桶内元素通过 Node::next 属性找下一个节点;
当桶内元素访问完毕后, 通过数组索引, 扩展到其他桶中, 继续迭代;

```java

    // iterators

    abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            // 关键:   
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
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

    final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().key; }
    }

    final class ValueIterator extends HashIterator
        implements Iterator<V> {
        public final V next() { return nextNode().value; }
    }

    final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }
    }
    
```

关键点在于

```java
          if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
           }
```

如果当前桶通过next指针拿不到, 将移动数组索引index;


#### 参考链接

[维基百科](https://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91)
[无处不在的小土](https://gaoyichao.com/Xiaotu/?book=data_and_algorithm&title=%E7%BA%A2%E9%BB%91%E6%A0%91)



