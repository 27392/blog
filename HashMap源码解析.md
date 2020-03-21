# HashMap 

https://segmentfault.com/a/1190000012926722

## 构造函数
```java
class HashMap {

    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        // 使用阈值暂时保存初始化值,在扩容的时候会用上
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
}
```

## put

```java
class HashMap {

    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * 实现了Map.put相关方法。
     * @param hash            key的hash
     * @param key             key
     * @param value           要存放的值
     * @param onlyIfAbsent    如果为真,则不要更改现有值
     * @param evict           如果为假,则该表处于创建模式
     * @return                以前的值,如果没有则返回null
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        // p : 需要存放的桶中的链表
        Node<K,V>[] tab; Node<K,V> p;
        // n : 桶的长度, i : 桶的位置
        int n, i;
        
        // 初始化桶数组 resize() 方法
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        
        // 计算出要存放的桶,然后判断该桶中是否存在节点,如果没有直接放在该桶中
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);

        // 计算出要存放桶,且该桶中已经存在节点
        else {
            Node<K,V> e; K k;

            // 如果第一个节点的 hash、key 相等,则替换 value
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
        
            // 如果节点的类型为 TreeNode 则使用红黑树的方式插入
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
            // 如果与第一个节点(hash、key)不相同,则遍历链表进行查找
            else {
                for (int binCount = 0; ; ++binCount) {
                    
                    // 这里 e 是指向链表的下一个节点,如果没有找到的话 e 则是 null
                    // 如果找到最后一个位置任然是没有找到的话,则将该节点接在链表的最后
                    if ((e = p.next) == null) {
                        // 将该节点接在链表的最后
                        p.next = newNode(hash, key, value, null);
                        
                        // 如果链表长度大于或等于树化阈值(也就是`8`),则进行树化操作(但 table.length 不能小于64)
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 树化
                            treeifyBin(tab, hash);
                        break;
                    }
                    
                    // 如果找到 hash、key 相等,则返回.
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // 将下一个节点再次指向 p ,如此往复遍历
                    p = e;
                }
            }
            
            // 键存在,则替换 value 值,并返回之前的值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // onlyIfAbsent 如果为 true ,则不要更改现有值
                if (!onlyIfAbsent || oldValue == null)
                    // 替换 value
                    e.value = value;
                afterNodeAccess(e);
                // 返回之前的值
                return oldValue;
            }
        }
        ++modCount;

        // 是否超过阈值则,超过则扩容
        if (++size > threshold)
            // 进行扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }
}
```

## resize

```java
class HashMap {

    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
    
        // 如果 table 不为空
        if (oldCap > 0) {

            // 超过最大长度超过则不再扩容,并将阈值调至最大
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }

            // 将容量扩张两倍,并且小于最大容量在并且旧的容量大于等于(16)
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 旧的容量大于等于16的时候才会将阈值扩张两倍
                newThr = oldThr << 1; // double threshold
        }
        
        // 如果在构造器中指定了初始化容量,具体看构造器方法,这里将它取出作为新的初始化容量
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;

        // 初始化 table 容量默认(16), 阈值默认(12)
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        
        // 创建阈值.由于在构造方法中指定了初始化容量的原因,这里需要重新计算阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        
        // 赋值新的阈值
        threshold = newThr;
    
        // 创建新的桶数组,同时桶数组的初始化也是在这里初始化的
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];

        // 赋值扩容之后的桶数组
        table = newTab;
        
        // 如果旧的桶不为空 (在第一次初始化的时候为null)
        if (oldTab != null) {
            
            //遍历旧桶的数据
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                
                // 取出桶中数据,并且不等于空
                if ((e = oldTab[j]) != null) {
                    
                    // 清除旧桶的数据,方便 GC
                    oldTab[j] = null;
                    
                    // 如果该桶只有一个节点,则将这个节点放进新的桶中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    
                    // 如果是红黑树的情况,调用红黑树的方法
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    
                    // 遍历链表数据
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                                
                            // 容量是2的N次幂,(e.hash & oldCap == 0),说明此时e.hash小于原来数组的长度,因为索引的计算方式是(e.hash & (newCap - 1)) 所以Node的索引不会发生改变
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            
                            // 反之高于原先数组长度
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        
                        // 低位链表放在原先的位置
                        if (loTail != null) { 
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        
                        // 高位链表放在新的位置(j + oldCap)
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
}
```

在两种情况下可以扩容:

1. 当元素数量超过阈值

2. 当树化时(单桶链表超过8),并且桶的大小小于64时


## 参考资料

https://www.jianshu.com/p/f4a09386de22  

https://juejin.im/post/5dea048b5188251247686f08#heading-12

https://segmentfault.com/a/1190000012926722#item-2

https://www.bilibili.com/video/av93578466?p=13