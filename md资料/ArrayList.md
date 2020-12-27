###ArrayList简介：
我们知道 ArrayList 属于 Java 中的高级数据的集合框架。项目包在  java.util 包下。具体的数据框架图如下图所示：


![结构框架图](https://github.com/harusty/image/blob/master/java/arraylist/arraylist_01.gif)

ArrayList 的主要继承关系：

![结构框架图](https://github.com/harusty/image/blob/master/java/arraylist/arraylist_01.gif)


### ArrayList 的简单使用：

    public static void main(String[] args) {
		ArrayList heros = new ArrayList<>();
		heros.add("貂蝉");
		heros.add("孙尚香");
		heros.add("鲁班七号");
		heros.add(0,"百里守约");
		heros.set(0, "小乔");
		heros.get(0);
		heros.remove("鲁班七号");
		heros.remove(0);
		heros.clear();
		boolean isEmpty = heros.isEmpty();
	}

以上大致为我们在开发中经常使用到的一些方法。


### ArrayList 源码探析

在探究 ArrayList 源码之前，先声明 ArrayList 内部维护了一个数组对数据进行操作，带着这个问题继续查看源码看如何数组相关操作。

ArrayList 的主要使用方法都在其抽象子类 AbstractList 类中进行定义。


    public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
   
	    protected AbstractList() {
	    }
	
	    
	    public boolean add(E e) {
	        add(size(), e);
	        return true;
	    }
	
	    
	    abstract public E get(int index);
	
	   
	    public E set(int index, E element) {
	        throw new UnsupportedOperationException();
	    }
	
	   
	    public void add(int index, E element) {
	        throw new UnsupportedOperationException();
	    }
	
	   
	    public E remove(int index) {
	        throw new UnsupportedOperationException();
	    }
	
	
	   
	    public int indexOf(Object o) {
	        ListIterator<E> it = listIterator();
	        if (o==null) {
	            while (it.hasNext())
	                if (it.next()==null)
	                    return it.previousIndex();
	        } else {
	            while (it.hasNext())
	                if (o.equals(it.next()))
	                    return it.previousIndex();
	        }
	        return -1;
	    }
	
	  
	    public int lastIndexOf(Object o) {
	        ListIterator<E> it = listIterator(size());
	        if (o==null) {
	            while (it.hasPrevious())
	                if (it.previous()==null)
	                    return it.nextIndex();
	        } else {
	            while (it.hasPrevious())
	                if (o.equals(it.previous()))
	                    return it.nextIndex();
	        }
	        return -1;
	    }
	
	
	
	    public void clear() {
	        removeRange(0, size());
	    }
	
	   
	    public boolean addAll(int index, Collection<? extends E> c) {
	        rangeCheckForAdd(index);
	        boolean modified = false;
	        for (E e : c) {
	            add(index++, e);
	            modified = true;
	        }
	        return modified;
	    }
	}

### ArrayList 的构造方法

    
	
	
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

   
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }


    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

java 提供了三种构造方式进行构造 ArrayList ,我们经常用到 ArrayList 的空构造方法，而这里可以看到只是对 elementData 进行了赋值操作。

    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
  
    transient Object[] elementData;

 
上述代码可以看出 ArrayList 的空构造方法其实就是初始化了一个空数组。而传参构造无非两个主要目的，1 构造数据长度；2 构造数据中的数据。


接下来对 ArrayList 里的数据进行操作；


### ArrayList 的 add 方法

    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

add 方法会调用 ensureCapacityInternal 检测数组目前的长度是否安全。
	

    private static final int DEFAULT_CAPACITY = 10;

	private void ensureCapacityInternal(int minCapacity) {
		
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {// 只有初始化第一次的时候会执行此处
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

	
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

当首次往 ArrayList 增加元素的时候，数组初始化默认长度为 10 。当以后继续增加元素的时候同样会对 elementData 数组数据长度进行判断，如果发现 minCapacity 的长度将要大于数组长度的时候，则调用 grow 方法对数组长度进行增长；

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

当发现数组长度将要超过原有数组长度的时候，新建新的数组，长度为 n + n/2；同时将数据转移到新的数组中赋值给 elementData，这样就避免了添加数据过程中可能导致数组越界的错误；当完成了数组长度安全性的操作后继续执行 add 方法后续数据赋值的操作；

在固定位置添加元素

    public void add(int index, E element) {
        rangeCheckForAdd(index);// 检测index 是否合法，超过长度？或者小于0

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }

可以看到此方法依然会检测此次加入是需要增加数组长度，同时将从 index 处开始的数据进行往后移动，把 element 值赋在 index 处。



### ArrayList 的 set 方法

    public E set(int index, E element) {
        rangeCheck(index);//检测index'的合法性

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

    E elementData(int index) {
        return (E) elementData[index];
    }

set 方法很简单，将原来数组 index 处的数据进行替换即可，并返回旧的存储数据。



### ArrayList 的 get 方法

    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

    E elementData(int index) {
        return (E) elementData[index];
    }

get 方法返回数据 index 位置的数据。


### ArrayList 的 remove 方法

    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }


    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
remove 方法中可以看到会删除为 null 的数据，且第最先被加入为 null 的数据，这也同时说明 ArrayList 可以添加 null 数据。fastRemove 类似 add 方法中数组长度增加的方法，不过此处是将 index 后的元素前移。然后数组最后一位置空。固定位置的删除同理。

### ArrayList 的 clear 和 isEmpty 方法

    public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }

    public boolean isEmpty() {
        return size == 0;
    }


clear 方法将 lementData 所有数据置空，同时长度置 0 。isEmpty 返回 size 参数即可。



### ArrayList 的遍历。

		ArrayList heros = new ArrayList<>();
		heros.add("貂蝉");
		heros.add("孙尚香");
		heros.add("鲁班七号");
		
		// 方式1
		for(int i = 0;i < heros.size();i++){
			System.out.print("hero===" + heros.get(i));
		}

		方式二
        Iterator it = heros.iterator();
        while(it.hasNext()){
			System.out.print("hero===" +  it.next());
        }

大多数情况下我们使用以上方式进行遍历，ArrayList 同样也提供了 Iterator 遍历器方式进行遍历，这里主要看下 Iterator 方式。

    
    public Iterator<E> iterator() {
        return new Itr();
    }

    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

Itr 为 ArrayList 的内部类，继承自 Iterator，可以访问到 ArrayList 类中定义的变量。

hasNext()方法很简单，查看访问的位置是否到到达了 size 处，size 上面提到为 ArrayList 当前存储的数据的长度。hasNext() 方法与 next() 息息相关；
next() 方法会取到 i = cursor 数据，cursor 默认初始为 0，调用 next 方法后增加，同时将 i 赋值 lastRet；
remove() 方法默认删除 lastRet 即最后一位的数据。

这里重点看下 checkForComodification 方法，在 next 方法遍历以及 remove 方法中都会调用到此方法,这里当 modCount 不等于 expectedModCount 的时候会抛出异常，Itr类中未发现两者不相等的情况。modCount 方法在 ArrayList 中定义，而在 ArrayList 类中 add 和 remove 方法均会对 modCount 改变。所以当我们在使用 Iterator 进行遍历的时候，不可以对原本的数据集合进行增加或者删除操作。不然会出现一些未知的错误（越界），同理，第一种方式在遍历过程中对集合本身操作也有可能出现错误的风险。


以上，ArrayList 的主要方法已经大致了解。我们分析过程发现 ArrayList 并没有对相关操作有同步代码处理。所以，ArrayList 是线程不安全的集合类，在实际开发中如果多线程操作 ArrayList 集合则有可能出错，如果想要使用线程安全的集合，该怎么办呢。 java 提供了一下方式获取线程安全的集合。

1 List<Map<String,Object>> data=Collections.synchronizedList(new ArrayList<Map<String,Object>>());
2 使用 CopyOnWriteArrayList；

后续 我们继续查看源码看看如何实现集合的安全线程。


   

  













