##HashMap 简介
HashMap 是一个散列表，它存储的内容是键值对(key-value)映射。

HashMap 实现了 Map 接口，根据键的 HashCode 值存储数据，具有很快的访问速度，最多允许一条记录的键为 null，不支持线程同步。

HashMap 是无序的，即不会记录插入的顺序。

HashMap 继承于AbstractMap，实现了 Map、Cloneable、java.io.Serializable 接口。

###HashMap 的继承关系

首先看下集合框架图，Map 和 Collection 属于两个不同的结构，这里容易弄混淆。

![结构框架图](https://github.com/harusty/image/blob/master/java/arraylist/arraylist_01.gif)

HashMap 的继承关系

![HashMap 继承关系](https://github.com/harusty/image/blob/master/java/hashmap/HashMap继承关系.png)

## HashMap 简单使用

HashMap 的 key 与 value 类型可以相同也可以不同，可以是字符串（String）类型的 key 和 value，也可以是整型（Integer）的 key 和字符串（String）类型的 value。


	    HashMap<String, Integer> hhs = new HashMap<>();
	    hhs.put("aaa", 1);
	    hhs.put("bbb", 2);
	    hhs.put("aaa", 1);

![HashMap（key-value）](https://github.com/harusty/image/blob/master/java/hashmap/HashMap[key-value].svg)

## HashMap 的存储结构

在了解 HashMap 源码之前，先来看下 HashMap 的存储结构图。

![HashMap存储结构](https://github.com/harusty/image/blob/master/java/hashmap/HashMap存储结构.svg)

可以看到左方为数组存储方式，数组的方式特点是查询快[o(1)]，但是增加和删除比较慢[o(n)]。右方为链表的数据结构，链表数据结构查询慢[o(n)]，但是增加和删除[(o(1))]比较快。而 HashMap 采用两种方式的结合。相当于折中方案，提高效率。


## HashMap 的基本原理

1、首先判断Key是否为Null，如果为null，直接查找Enrty[0]，如果不是Null，先计算Key的HashCode，然后经过二次Hash。得到Hash值，这里的Hash特征值是一个int值。

2、根据Hash值，找到对应的数组，所以对 Entry[] 的长度 length 求余，得到的就是 Entry 数组的 index。

3、找到对应的数组，就是找到了所在的链表，然后按照链表的操作对Value进行插入、删除和查询操作。



### HashMap 的构造方法

首先来看 HashMap 的构造函数

    public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }


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
        threshold = initialCapacity;
        init();
    }


构造函数中初始化了一个 loadFactor 变量，默认值 DEFAULT_LOAD_FACTOR 为 0.75。 threshold 变量默认值为 16（1 左移四位） 表示为 Entry[] 数组的临界长度。 loadFactor 和 threshold 决定是否要对 Entry[] 的长度进行调整。 threshold 由 HashMap 数组长度和 loadFactor 相乘计算得出，例如数组长度为 16 ，loadFactor 为 0.75，则 threshold 应该为 12 。即当 HashMap 中的容量数据大于等于 12 的时候，则应该对 table 数组进行扩容处理。 此时初始化默认为数组长度（后续操作会对其更改）。 init() 为空调用方法。

可以看到构造函数只是初始化了一些变量。

接下来 HashMap 的 put 操作则是 HashMap 原理的主要研究点。

    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
		 // table 数组为空则根据 threshold 初始化数组，即为容量为 16 的数组，同时 threshold 更改为 数组长度 length * loadFactor
            inflateTable(threshold);
        }
        if (key == null) // key 为空 
            return putForNullKey(value);
        int hash = hash(key); // 对key 进行 hash 处理
        int i = indexFor(hash, table.length); // 通过 hash 值拿到 key 对应数组的位置
        for (Entry<K,V> e = table[i]; e != null; e = e.next) { // 对应 Entry
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

这里可以看到，table[] 数组中存入的为 Entry，默认初始容量为 16 。根据上述数据和 Entry 结构图知道 Entry 数据保存了数据的 key 、value、hash 、i（数组下标）的值。



传入 key 为 null 的时候，会将数据 Entry 数据存入 table[0] 的位置；

 
    private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) { 
            if (e.key == null) {// 替换 key 为 null 的value 
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);// 插入数据
        return null;
    }

### HashMap put方法的 hash 处理

如果key 不为空，则对 key 进行 hash 操作；

    final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }

针对 key 获取 hash 值，这里会再次进行一系列的逻辑操作， 这里注释也对其进行了说明，对 hash 值进行二次操作是为了不同的 key 算出来的 hash 值尽可能不一样，避免出现 hash 冲突的情况

接下来，拿到二次处理过的 hash 值生成对应 table 的下标 i；

    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }

 length 为 table 数组的长度，  h & (length-1) 实际为对数组长度取模运算（h % table.length()）,例如，现在数组长度为 16，算出来的 hash 值为 15，与运算之后即为 15。这里其实 length - 1 后对相应低位二进制全部改变，而再和 hash 值的低位进行 & 运算，对应拿到的就是 hash 对应低位的值。即为对 length 的取模。

而这里对二次处理的 hash 值进行取模运算，是为了我们的数据能够尽可能均匀分布在 table 数组中。提高 table 的使用效率。

到这里，已经找到可以找到 key 值对应在 table 数组的位置，前面的处理总结下有两个目的：

1. 二次 hash 处理，避免出现 hash 碰撞的情况。
2. 对数组长度取模处理，使数据均匀分布在数组中。

拿到了数组 table 的对应下标 index，取出对应的 Entry 数据。针对 Entey 链表进行遍历，如果查找到对应的 key 则进行替换，否则，调用 addEntry 插入数据。


接下来再看看 addEntry 方法；

    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) { // 当 size 达到 HashMap 临界值了，调整 table 的长度
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }

先看 createEntry 方法去创建数据；

    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }

将对应数据放在 table 数组引用的节点处，假如原来 bucketIndex 处有相应的数据，则存入当前存入数据的下一个节点中；

针对上述操作，如下图示展示：

![HashMap 继承关系](https://github.com/harusty/image/blob/master/java/hashmap/HashMap_put_01.png)

现在 HashMap 的存储的数据如上图所示，假设现在有个（k9,v9）的值需要插入到 HashMap 中。

当对 k9 进行 hash 等一系列运算之后，发现此时数据的下标为 1，即和 （k3，v3）、（k8，v8）的处理下标一致，进行如下存储；

![HashMap 继承关系](https://github.com/harusty/image/blob/master/java/hashmap/HashMap_put_02.png)

此时再次存储 （k3，v10）的数据，k3 通过 hash 之后算出来的下标肯定也是在 table[1] 处，替换进行如下存储；

![HashMap 继承关系](https://github.com/harusty/image/blob/master/java/hashmap/HashMap_put_03.png)

最后再插入一条（k10，v10）的数据，算出下标为 5 ，如下进行存储；

![HashMap 继承关系](https://github.com/harusty/image/blob/master/java/hashmap/HashMap_put_04.png)


### HashMap 的扩容处理

当 HashMap 里面存储的数据到了一定的临界值的时候，应当对 table 进行扩容处理；

刚刚上述 addEntry 方法中提到扩容处理情况；

        if ((size >= threshold) && (null != table[bucketIndex])) { // 当 size 达到 HashMap 临界值了，调整 table 的长度
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

可以看到当 size 大于等于 threshold 临界值的时候，且当前的数据不为空的情况下，我们就要进行 table 扩容了。threshold 我们知道由 loadFactor（加载因子）和 table 的容量共同决定，loadFactor 为大于 0 小于 1 的值，即此时已存数据 size 到达 table 容量的 loadFactor 倍数的时候，就必须进行扩容了。那么如何扩容的呢?

    void resize(int newCapacity) {
        Entry[] oldTable = table;// 旧的table
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];// 新table 容量翻倍
        transfer(newTable, initHashSeedAsNeeded(newCapacity));// 数据的转移
        table = newTable;// newtable 赋值给 table
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1); // 重新计算临界值
    }

    void transfer(Entry[] newTable, boolean rehash) {// rehash 默认为 true 未自定义 hash 算法
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }


看到扩容处理不仅仅把 table 容量增长两倍就完事了，原 table 中的数据该怎么存呢？

如果不进行更换的话，假如上述 （k9,v9）的数据查找，因为 table 扩容后，二次 hash 结果和 table.length 取模后获取的下标数据大概率是变化了，那样就找不到数据了。

于是 transfer 方法拿到旧的 table 数据，然后根据 newTable 的容量对原数据所有的 key 重新计算对应下标，存入到新的 table 中，这样保证了后续查找的正确性。

到这里，HashMap put 相关的分析已经完成，其中涉及到对数组和对应链表数据的插入。以及相应的数组进行扩容方案。


### HashMap 的 其他方法

分析完成了 put 方法之后，再看其他方法就会很容易。

get 方法：

    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }

    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);// hash 处理
        for (Entry<K,V> e = table[indexFor(hash, table.length)];// 找到数组对应的下标
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }

get 方法需要传入相应的 key ，如果为 null key ，肯定就是去找 table 数组第 0 位的数据。否则，key 的 hash 值获取以及二次处理，找到对应的数组下标，比对对应 Entry 一致就返回，否则就返回 null。


remove 方法：


    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

    final Entry<K,V> removeEntryForKey(Object key) {
        if (size == 0) {
            return null;
        }
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e) // 删除的是链表第一个数据
                    table[i] = next;
                else
                    prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }

remove 方法相对 get 方法，在拿到相应数据删除之后，同时要将链表对应的数据重新进行指向。即将当前要删除数据的前一项指向后一项。


### HashMap 的遍历方法

	// 方式1
	for(Map.Entry<String, Integer> entry : hhs.entrySet()){
	    System.out.println("key = " + entry.getKey() + ", value = " + entry.getValue());
	}

	//方式2
	Iterator<Map.Entry<String, Integer>> entries = hhs.entrySet().iterator();
	while (entries.hasNext()) {
		Map.Entry<String, Integer> entry = entries.next();
		System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
	}


由上述 HashMap 的操作可以看到 HashMap 中存储数据是无序的，所以遍历得到的数据也将会是无序的；通常 HashMap 使用到的两种遍历方式（其实两种方法最终都会走向方式 2 ），entrySet 和 entrySet 的迭代器 iterator；

第一种遍历方式；

	private transient Set<Map.Entry<K,V>> entrySet = null;

    public Set<Map.Entry<K,V>> entrySet() {
        return entrySet0();
    }

    private Set<Map.Entry<K,V>> entrySet0() {
        Set<Map.Entry<K,V>> es = entrySet;
        return es != null ? es : (entrySet = new EntrySet());
    }

这里 entrySet 存储 Entry 的集合，在之前 put 方法中，并没有发现往其中存入数据。所以，这里去实例化 EntrySet 方法中查看是否有相关获取数据的方法？

    private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return newEntryIterator();
        }
        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<K,V> e = (Map.Entry<K,V>) o;
            Entry<K,V> candidate = getEntry(e.getKey());
            return candidate != null && candidate.equals(e);
        }
        public boolean remove(Object o) {
            return removeMapping(o) != null;
        }
        public int size() {
            return size;
        }
        public void clear() {
            HashMap.this.clear();
        }
    }

EntrySet 的实例化又去创建了一个 EntryIterator；

    Iterator<Map.Entry<K,V>> newEntryIterator()   {
        return new EntryIterator();
    }

    private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() {
            return nextEntry();
        }
    }

EntryIterator 实则就是一个 HashIterator，到这里可以发现，似乎走到了第二种方式了，继续看 HashIterator；

    private abstract class HashIterator<E> implements Iterator<E> {
        Entry<K,V> next;        // next entry to return
        int expectedModCount;   // For fast-fail
        int index;              // current slot
        Entry<K,V> current;     // current entry

        HashIterator() {
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null) // 自增数组下标找到第一个不为null结束循环 即为HashMap要遍历的第一个值
                    ;
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

            if ((next = e.next) == null) {// index 处链表数据没有下一位了
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
            current = e;
            return e;
        }

        public void remove() {
            if (current == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Object k = current.key;
            current = null;
            HashMap.this.removeEntryForKey(k);
            expectedModCount = modCount;
        }
    }


HashIterator 实现了 Iterator，而增强 for 循环使用的就是 Iterator ，所以这里 Entry 覆盖了 Iterator 方法进行遍历数据。

HashIterator 构造方法回去遍历数组找到第一个不为 null 的数据结束循环，此数据作为遍历 HashMap 的起始数据；

这里主要看下 nextEntry 方法，因为 EntryIterator 的 next 方法其实调用的就是 nextEntry 方法；

可以看到 nextEntry 会将数据按照 table 的下标顺序进行查找，如果某个下标对应的链表数据查找完毕，继续向下遍历 table 数组。直到全部查找完毕。

![HashMap 继承关系](https://github.com/harusty/image/blob/master/java/hashmap/HashMap_put_04.png)

以上图 HashMap 数据来看，最终遍历出数据结果应该是(null,vaule)、(k9,v9)、(k3,v8)、(k8,v8)、(k10,v10),当然，存储顺序大概率不是按照这个顺序来的。所以我们说 HashMap 是无序的。

最后，可以看到 HashMap 也不支持多线程的操作 ，因为对 HashMap 的操作每次都会改变 modCount 值，而遍历的过程会比对相关值，所以遍历过程也不可对 HashMap 进行增删操作。





 














 
 














