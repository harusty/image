##LinkHashMap 简介
LinkHashMap 继承自 HashMap 同时也实现了 Map 接口。所以 LinkHashMap 的方法与 HashMap 中的方法类似，我们知道 HashMap 的存储的数据是无序的，而 LinkHashMap 则在 HashMap 的基础上实现了有序的数据存储，本文就主要查看源码看看 LinkHashMap 是如何实现有序的存储方式的；

![LinkHashMap 继承关系](https://github.com/harusty/image/blob/master/java/linkhashmap/LinkHashMap继承关系.png)

HashMap 的源码分析可参考链接处文章


### LinkHashMap 的数据存储结构

我们知道 HashMap 在 JDK1.7 上的数据是基于数组和链表结构进行存储的，而 LinkHashMap 在此基础上增加了链表的双向存储结构，先看下大致的示意图







## LinkHashMap 分析

LinkHashMap 在使用方法上和 HashMap 基本一致：

		LinkedHashMap<String, Integer> hss = new LinkedHashMap<>();
		hss.put("aaa", 1);
		hss.put("bbb", 2);
		hss.put("aaa", 3);
		
		hss.get("aaa");
		hss.remove("aaa");

	    for(Map.Entry<String, Integer> entry : hss.entrySet()){
	    	System.out.println("key = " + entry.getKey() + ", value = " + entry.getValue());
	    }
	    

		Iterator<Map.Entry<String, Integer>> entries = hss.entrySet().iterator();
		while (entries.hasNext()) {
			Map.Entry<String, Integer> entry = entries.next();
			System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
		}


接下来就看下，LinkHashMap 对数据的相关操作，如何能够做到有序地存取数据？

### LinkHashMap 的构造方法：

    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

    @Override
    void init() {
        header = new Entry<>(-1, null, null, null);
        header.before = header.after = header;
    }

LinkHashMap 在调用了 HashMap 的构造方法后，设置了一个 accessOrder 为 false，根据意思猜想这个和访问顺序有关。同时 LinkHashMap 重写了 init 方法，创建了一个 hear 的 节点 ，这里 LinkHashMap 对 Entry 也进行了重写，后续方法中会继续提到；

### LinkHashMap 的 put 方法：

    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }


LinkHashMap 的 put 方法其实就是调用了 HashMap 的 put 方法，前面逻辑和 HashMap 一致，直到调用到 addEntry 方法，LinkHashMap 进行了重写；

    void addEntry(int hash, K key, V value, int bucketIndex) {
        super.addEntry(hash, key, value, bucketIndex);

        // Remove eldest entry if instructed
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }

    
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMap.Entry<K,V> old = table[bucketIndex];
        Entry<K,V> e = new Entry<>(hash, key, value, old);
        table[bucketIndex] = e;
        e.addBefore(header);
        size++;
    }

addEntry 方法依然会调用到 HashMap 的 addEntry 方法，这里也可以再回顾下 HashMap 的 addEntry 方法；


    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {// 是否需要扩容处理
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);// 创建数据节点
    }

    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }

HashMap 的 addEntry 最终会调用 createEntry 方法，而 LinkHashMap 同时也重写了 createEntry 方法，所以这里重点分析就在 LinkHashMap 中的 addEntry 和 createEntry 方法了；

首先，当存入数据对数据扩容判断后，就直接调用了 createEntry  方法；

createEntry 方法中 LinkHashMap 比 HashMap 多一步，**e.addBefore(header)**


    private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }

       
        private void remove() {
            before.after = after;
            after.before = before;
        }

       
        private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

       
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }

        void recordRemoval(HashMap<K,V> m) {
            remove();
        }
    }

LinkHashMap 的 Entry 实现了 HashMap 的 EntryKey，前面在初始化的方法中提到会创建一个 header，LinkHashMap 的 addBefore 方法
其实就是双向链表指向的不断交换过程，下图演示 LinkHashMap 在依次添加 Num1、 Num2、 Num3 的过程中链表指向不断改变的过程；

未进行添加元素前：

![LinkHashMap 继承关系](https://github.com/harusty/image/blob/master/java/linkhashmap/LinkHashMap_001.png)

看到 Header 元素的 before 和 after 都指向自己；

接下来添加 Num1 ：

![LinkHashMap 继承关系](https://github.com/harusty/image/blob/master/java/linkhashmap/LinkHashMap_002.png)

Header 和 Num1 数据的 before 和 after 相互进行链接指向；

继续添加 Num2：

![LinkHashMap 继承关系](https://github.com/harusty/image/blob/master/java/linkhashmap/LinkHashMap_003.png)

此时，header 的 before 指向 Num2，同时 Num2 的 before 指向 Num1， Num1 的 before 指向 Header，这里的 before 就像是一个倒序的循环，Num2 -> Num1;而 after 指向的情况正好相反，Num1 -> Num2, 这正好符合我们加入的顺序；

最后加入 Num3 ，Num3 的数组索引位置和 Num1 相同，遵循 HashMap 的插入规则：

![LinkHashMap 继承关系](https://github.com/harusty/image/blob/master/java/linkhashmap/LinkHashMap_004.png)

这里 header 和 after 的指向和上述情况一致，只不过 Num3 有个 next 的指向为 Num1；


看完 LinkHashMap 的插入操作，可以发现通过双向的链表 LinkHashMap 其实已经完成了插入顺序的记录过程；

需要注意的是，上述过程在完成添加操作之后，还会做一个 removeEldestEntry 操作,默认返回 false ，供继承者使用；


### LinkHashMap 的 get 方法：

    public V get(Object key) {
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        e.recordAccess(this);
        return e.value;
    }

调用 HashMap 的 getEntry 方法，和 HashMap 处理基本一致；


### LinkHashMap 的 remove 方法：

    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }


同样调用 HashMap 的 removeEntryForKey 方法移除相应的数据即可；


### LinkHashMap 的遍历

通过 HashMap 的介绍我们可以知道，HashMap 是通过 HashIterator 实现 Iterator 来完成的，前面 LinkHashMap 通过双向链表已经记录了数据的插入顺序，接下来看看 LinkHashMap 如何去按顺序取出数据？

    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() { return nextEntry(); }
    }

    private abstract class LinkedHashIterator<T> implements Iterator<T> {
        Entry<K,V> nextEntry    = header.after; // 访问开始点
        Entry<K,V> lastReturned = null;

        int expectedModCount = modCount;

        public boolean hasNext() {
            return nextEntry != header;// 再次指向 header
        }

        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            LinkedHashMap.this.remove(lastReturned.key);
            lastReturned = null;
            expectedModCount = modCount;
        }

        Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (nextEntry == header)
                throw new NoSuchElementException();

            Entry<K,V> e = lastReturned = nextEntry;
            nextEntry = e.after;// 移到下一个点
            return e;
        }
    }

LinkHashMap 内部维护了自身的 Entry 类，Entry 类中维护的 EntryIterator 类集成自 LinkedHashIterator；


LinkedHashIterator 从 header.after 开始访问，以 after 进行访问点移动，直到节点指向 header 结束；以上述 Num1、Num2、Num3的案例来看，依次取出数据是 Num1、Num2、Num3 ，和存放的数据一致，从而实现了有序的排序；

在这里 modCount 和 expectedModCount 的判断过程以及内部操作无同步机制，表明 LinkHashMap 是非线程安全的集合类（同 HashMap）。


### LinkHashMap 的 accessOrder

最后，来看下 accessOrder 这里变量的意义。accessOrder 在 LinkHashMap 的初始化中默认为 false，代表按照插入顺序进行排序。加入设置true 则代表访问顺序进行排序，具体可以搜索到 LinkHashMap 中的相关代码：

        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }

LinkHashMap 中 recordAccess 方法会用到 accessOrder 这个变量，可以看到这个变量将当前数据进行删除，addBefore 方法会将其插入到双向链表最后一位。

    public V get(Object key) {
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        e.recordAccess(this);
        return e.value;
    }

而 recordAccess 方法则会在 LinkHashMap get 数据的时候进行调用。此时将访问的到的数据放到最末位。

	   public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
        }



		LinkedHashMap<String, Integer> hss = new LinkedHashMap<>(16,0.75f,true);
		
		hss.put("aaa", 1);
		hss.put("bbb", 2);
		hss.put("ccc", 3);
		
		hss.get("aaa");

	    for(Map.Entry<String, Integer> entry : hss.entrySet()){
	    	System.out.println("key = " + entry.getKey() + ", value = " + entry.getValue());
	    }
	    

上述案例将 accessOrder 设置为 true ，最终遍历到的顺序为 2、 3、 1；



## LinkHashMap 总结

最后，通过本篇文章对 LinkHashMap 做一个总结：

1、LinkHashMap 是一个有序的集合类，可存储 null key；
2、LinkHashMap 是非线程安全的集合类；
3、LinkHashMap 采用了双向链表保证数据的存储顺序；
4、LinkHashMap accessOrder 可以用来把最近访问的数据放在最后一位，这一点在实际相关 LRU 数据缓存操作经常用使用到；




