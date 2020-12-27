## HashTable 介绍（JDK 1.7）

首先，依然看下 HashTable 的集合框架结构

![HashTable](https://github.com/harusty/image/blob/master/java/hashtable/HashTable.png)

查看 HashTable 代码中继承关系；

	public class Hashtable<K,V>
    	extends Dictionary<K,V>
    	implements Map<K,V>, Cloneable, java.io.Serializable {
	}

HashTable 主要和 HashMap 经常放在一起对比，看看 HashMap 的继承关系；

	public class HashMap<K,V>
    	extends AbstractMap<K,V>
    	implements Map<K,V>, Cloneable, Serializable{
	}

了解 HashMap 的内部实现原理可查看这篇文章：



可以看到 HashTable 和 HashMap 的父类是不同的，接下来就通过使用 HashTable 查看 HashTable 的运行原理；

## HashTable 的数据结构

HashTable 和 HashMap 的数据结构是一致的，均为数组和链表结合的数据结构方式；

![HashTable](https://github.com/harusty/image/blob/master/java/hashtable/HashTable.png)


## HashTable 和 HashMap 的区别

HashTable 经常和 HashMap 进行对比，他们之间的主要区别有一下两点

1. HashTable 是线程安全的，而 HashMap 是非线程安全的集合类；
2. HashTable key 和 value 只能是非 null ，而 HashMap 可以存储一个 null key，value 可以随意取值；

而其他的区别将在接下来源码中一一发现；


 


## HashTable 的使用

	public static void main(String[] args) {
	    Hashtable<String, Integer> hhs = new Hashtable<>();
	    hhs.put("aaa", 1);
	    hhs.put("bbb", 2);
	    hhs.put("aaa", 1);
	    
	    hhs.get("aaa");
	    hhs.remove("aaa");
	    
	    for(Map.Entry<String, Integer> entry : hhs.entrySet()){
	    	System.out.println("key = " + entry.getKey() + ", value = " + entry.getValue());
	    }
	    

		Iterator<Map.Entry<String, Integer>> entries = hhs.entrySet().iterator();
		while (entries.hasNext()) {
			Map.Entry<String, Integer> entry = entries.next();
			System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
		}
	}

HashTable 的使用方式和 HashMap 基本一致，都是实现 Map 类相关的操作接口，下面重点关注 HashTable 的内部实现原理；

## HashTable 的实现原理

### HashTable 的构造方法

    public Hashtable(int initialCapacity, float loadFactor) {// 初始化容量，以及扩容因子
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        initHashSeedAsNeeded(initialCapacity);
    }


    public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }

    public Hashtable() {
        this(11, 0.75f);
    }


    public Hashtable(Map<? extends K, ? extends V> t) {
        this(Math.max(2*t.size(), 11), 0.75f);
        putAll(t);
    }

HashTable 为我们提供了四种构造方法，主要目的则是为了构造一个初始化容量的数组，以及通过 threshold 和扩容因子 loadFactor 确定数组将要在什么时候进行扩容处理，这一点和 HashMap 的构造基本一致；区别在于，HashTable 默认的数组容量为 11，而 HashMap 默认数组的容量为 16；而且 HashMap 在空构造函数中并没有去初始化数组，在加入第一个元素的时候才去初始化数组；

### HashTable 的 put 方法

继续查看 put 方法

    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry tab[] = table;
        int hash = hash(key); // key 为 null 会报错
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {// 找到相同的 key 即覆盖值
            if ((e.hash == hash) && e.key.equals(key)) {
                V old = e.value;
                e.value = value;
                return old;
            }
        }

        modCount++;
        if (count >= threshold) {// 当count 要临界值的时候，扩容，再次 hash
            // Rehash the table if the threshold is exceeded
            rehash(); // 

            tab = table;
            hash = hash(key);
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        Entry<K,V> e = tab[index]; // 原table[index]处的值，可能为null
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
        return null;
    }

首先可以看到，put 方法使用 synchronized 关键字进行修饰，保证线程的安全性；

对 key 的 hash 值处理之前，首先确保 value 不为 null，同时，当 key 为空的时候，hash 方法也会报错；

对 key 进行 hash 值处理的时候，我们先简单看下 HashMap 对 key hash 处理情况：

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

接下来看下 HashTable 中的 hash 方法：

    transient int hashSeed;

    private int hash(Object k) {
        // hashSeed will be zero if alternative hashing is disabled.
        return hashSeed ^ k.hashCode();
    }

HashMap 对 key 的 hash 值进行的二次处理，是为了不同的 key 值在数组中尽量不同（避免 hash 碰撞）；

HashTable 中仅仅是取了 key 的 hash 值(hashSeed 默认为 0 ，异或处理后数据不变);那么 HashTable 为什么不再处理 hash 值，从而避免 hash 碰撞 ？

可能这里大概是觉得没有必要再去处理碰撞，因为保证数据同步已经消耗性能了，不必在对碰撞做太多要求，即使碰撞了，也可以通过后续拉链法解决；



接着继续看 HashTable 在拿到 key 的 hash 值了之后，使用了 hash 值的低 31 位值和 hashtable 中数组的长度取模运算，算出的结果即该 key 值对应 HashTable 中数组的下标 index； length 在小于 31 位数的情况下，和 HashMap 算法一致。在效率上 ，& 运算高于 % 运算； 


    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }

到这里，我们发现，HashTable 中不同的 key 值还是有很大概率出现相同的数组下标 index 的；

但是，当 HashTable 中添加的数据数量将要大于临界值的时候，会调用 rehash 方法：

    protected void rehash() {
        int oldCapacity = table.length;
        Entry<K,V>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1; // 扩容至 2n+1 HashMap 为 2n 倍
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<K,V>[] newMap = new Entry[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);// 重新计算临界值
        boolean rehash = initHashSeedAsNeeded(newCapacity); // 默认为 true 没有自定义 hash 处理方法

        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                if (rehash) {
                    e.hash = hash(e.key);
                }
                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = newMap[index];
                newMap[index] = e;
            }
        }
    }

rehash 方法和 HashMap 扩容 resize 方法类似，HashTable 为 2n+1 倍，HashMap 为 2n 倍；

    void resize(int newCapacity) {
        Entry[] oldTable = table;// 旧的table
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];// 新table 容量翻倍
        transfer(newTable, initHashSeedAsNeeded(newCapacity));// 数据的转移 重新计算 key 对应数组下标
        table = newTable;// newtable 赋值给 table
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1); // 重新计算临界值
    }

当 HashTable 处理完成上述工作后，插入数组链接链表的第一位，下一位指向原 table[index] 处的数据(若为空，则没有)；

### HashTable 的 get 和 remove 方法

    public synchronized V get(Object key) {
        Entry tab[] = table;
        int hash = hash(key);
        int index = (hash & 0x7FFFFFFF) % tab.length; // 找到对应下标
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {// 查找相同的值
            if ((e.hash == hash) && e.key.equals(key)) {
                return e.value;
            }
        }
        return null;
    }



    public synchronized V remove(Object key) {
        Entry tab[] = table;
        int hash = hash(key);
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index], prev = null ; e != null ; prev = e, e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                modCount++;
                if (prev != null) {
                    prev.next = e.next;
                } else {
                    tab[index] = e.next;
                }
                count--;
                V oldValue = e.value;
                e.value = null;
                return oldValue;
            }
        }
        return null;
    }

get 和 remove 方法比较简单，和 HashMap 的方法基本类似， 同时 synchronized 实现代码的同步；

### HashTable 的遍历

	    for(Map.Entry<String, Integer> entry : hhs.entrySet()){
	    	System.out.println("key = " + entry.getKey() + ", value = " + entry.getValue());
	    }
	    

		Iterator<Map.Entry<String, Integer>> entries = hhs.entrySet().iterator();
		while (entries.hasNext()) {
			Map.Entry<String, Integer> entry = entries.next();
			System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
		}


通过 HashMap 的 entrySet 方法可以知道，最终会走向 Iterator 的方法，那么 HashTable 是不是也一样的呢？
	


    public Set<Map.Entry<K,V>> entrySet() {
        if (entrySet==null)
            entrySet = Collections.synchronizedSet(new EntrySet(), this);
        return entrySet;
    }

首先，HashTable 会通过 EntrySet 实例作为参数获取到一个 set 对象；

	private static final int ENTRIES = 2;

    private class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return getIterator(ENTRIES);
        }

        public boolean add(Map.Entry<K,V> o) {// 添加一个 map
            return super.add(o);
        }

        public boolean contains(Object o) { // 是否包含某个 Entry
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry entry = (Map.Entry)o;
            Object key = entry.getKey();
            Entry[] tab = table;
            int hash = hash(key);
            int index = (hash & 0x7FFFFFFF) % tab.length;

            for (Entry e = tab[index]; e != null; e = e.next)
                if (e.hash==hash && e.equals(entry))
                    return true;
            return false;
        }

        public boolean remove(Object o) {// 删除某个 Entry
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<K,V> entry = (Map.Entry<K,V>) o;
            K key = entry.getKey();
            Entry[] tab = table;
            int hash = hash(key);
            int index = (hash & 0x7FFFFFFF) % tab.length;

            for (Entry<K,V> e = tab[index], prev = null; e != null;
                 prev = e, e = e.next) {
                if (e.hash==hash && e.equals(entry)) {
                    modCount++;
                    if (prev != null)
                        prev.next = e.next;
                    else
                        tab[index] = e.next;

                    count--;
                    e.value = null;
                    return true;
                }
            }
            return false;
        }

        public int size() {
            return count;
        }

        public void clear() {
            Hashtable.this.clear();
        }
    }

可以看到 HashTable 的 EntrySet 内部同样维护了一个 Iterator 迭代器；对比下 HashMap 的 EntrySet 类，提供同样的方法；

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



HashTable 通过 getIterator 构造出一个迭代器，ENTRIES 默认为整数 2；

    private <T> Iterator<T> getIterator(int type) {
        if (count == 0) {// 没有数据的时候
            return Collections.emptyIterator();
        } else {
            return new Enumerator<>(type, true);
        }
	}

HashTable Iterator 的实现方式为 Enumerator ；


    private class Enumerator<T> implements Enumeration<T>, Iterator<T> {
        Entry[] table = Hashtable.this.table;
        int index = table.length;
        Entry<K,V> entry = null;
        Entry<K,V> lastReturned = null;
        int type;

      
        boolean iterator;

  
        protected int expectedModCount = modCount;

        Enumerator(int type, boolean iterator) { // 初始化
            this.type = type;
            this.iterator = iterator;
        }

        public boolean hasMoreElements() {// 判断是否有下一个元素
            Entry<K,V> e = entry;
            int i = index;// index为数组最后一位
            Entry[] t = table;
            /* Use locals for faster loop iteration */
            while (e == null && i > 0) {// 找到 table[index] 不为null 的 数据
                e = t[--i];
            }
            entry = e;
            index = i;
            return e != null;
        }

        public T nextElement() {// 查找下一个数据
            Entry<K,V> et = entry;
            int i = index;
            Entry[] t = table;
            /* Use locals for faster loop iteration */
            while (et == null && i > 0) {
                et = t[--i];
            }
            entry = et;
            index = i;
            if (et != null) {
                Entry<K,V> e = lastReturned = entry;// 赋值当前数据为 lastReturn
                entry = e.next; // 指针指向下一位
                return type == KEYS ? (T)e.key : (type == VALUES ? (T)e.value : (T)e);// 返回 key 或者 value
            }
            throw new NoSuchElementException("Hashtable Enumerator");
        }

        // Iterator methods
        public boolean hasNext() {
            return hasMoreElements();
        }

        public T next() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            return nextElement();
        }

        public void remove() {
            if (!iterator)
                throw new UnsupportedOperationException();
            if (lastReturned == null)
                throw new IllegalStateException("Hashtable Enumerator");
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            synchronized(Hashtable.this) {// 同步代码
                Entry[] tab = Hashtable.this.table;
                int index = (lastReturned.hash & 0x7FFFFFFF) % tab.length;

                for (Entry<K,V> e = tab[index], prev = null; e != null;
                     prev = e, e = e.next) {
                    if (e == lastReturned) {// 获取 lastReturned 的指向位置后删除
                        modCount++;
                        expectedModCount++;
                        if (prev == null)// 如果是在 链表第一位
                            tab[index] = e.next;
                        else   
                            prev.next = e.next;
                        count--;
                        lastReturned = null;
                        return;
                    }
                }
                throw new ConcurrentModificationException();
            }
        }
    }

Iterator 迭代器的 hasNext 和 next 方法相应又调用了 hasMoreElements 和 nextElement 来判断以及获取下一个数据，对比 HashMap 迭代器的方法，HashTable 对内部 Entry 数组从后往前遍历，而 HashMap 对应从前往后遍历；  remove 方法则直接删除当前迭代器指向的位置进行删除；

到这里，以及基本把 HashTable 的基础数据操作方法分析完成，再次对 HashTable 与 HashMap 对比进行总结，可以得出其他的一些异同点：

1. HashTable 是线程安全的集合类，而 HashMap 是线程非安全的；
2. HashTable 初始化大小默认为 11，同时构造函数会初始化内部 Entry 数组的容量；HashMap 初始化大小默认为 16 ，当添加第一个元素的时候才会初始化内部 Entry 数组；
3. HashTable 不能存 null key 和 null value；HashMap 可以存储一个 null key，value 无限制；
4. HashTable 根据 key 计算数组下标的时候不会二次 hash，HashMap 会进行二次 hash，两者都会以数组长度取模运算，保证数据在数组的分布更为均匀，HashMap 的计算方式效率比 HashTable 高（& 运算效率高于 % 运算）；
5. HashTable 扩容数据为 2n + 1；HashMap 扩容数据为 2n；
6. HashTable 迭代器的遍历对数组是从后往前遍历；HashMap 的遍历方式是对数组从前往后遍历；但两者遍历出来的结果都是无序的；
 

























