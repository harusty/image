## Vector 简介
 Vector 和 ArrayList 类似，顶级父类为 Collection，区别于 ArrayList 的重要一点，Vector 集合类为线程安全类，而本篇文章也将深入去解析为什么 Vector 可以做到线程安全。



vector 结构图

![vector结构图](https://github.com/harusty/image/blob/master/java/vector/vector.png)

##Vector 源码解析

因为和 ArrayList 使用类似，这里直接对 Vector 源码的相关增删改查操作进行分析；


ArrryList 源码分析

### Vector 的构造方法

    public Vector(int initialCapacity) {
        this(initialCapacity, 0);
    }


    public Vector() {
        this(10);
    }


    public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }

Vector 的构造方法初始化容量为 10 的数组，同时初始化变量 capacityIncrement 为 0；

### Vector 的 add 方法

    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);// 扩容
        elementData[elementCount++] = e;
        return true;
    }

    private void ensureCapacityHelper(int minCapacity) {
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

针对 add 方法 synchronized 加锁机制，当多线程进行访问的时候先要获取锁才能操作；ensureCapacityHelper 进行判断是否扩容，elementCount 为当前存入的数据数量，当下一位将要大于数组容量的时候就要扩容处理。和 ArrayList 不同的是，这里进行 2n 倍扩容；

### Vector 的 remove 方法

    public synchronized E remove(int index) {
        modCount++;
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);
        E oldValue = elementData(index);

        int numMoved = elementCount - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--elementCount] = null; // Let gc do its work

        return oldValue;
    }

synchronized 加锁保证同步性，删除之后数组后面位数进行前移动，原数组最后一位置空；同时可以看到每次增删操作都会对 modCount 自增处理；

### Vector 的 get 和 set 方法

    public synchronized E get(int index) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        return elementData(index);
    }



    public synchronized E set(int index, E element) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

synchronized 保证同步性；

看完 Vector 的增删改查操作方法可以看到 Vector 主要通过 synchronized 加锁机制保证了数据的同步；

### Vector的数据遍历

    public synchronized Iterator<E> iterator() {
        return new Itr();
    }

    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            // Racy but within spec, since modifications are checked
            // within or after synchronization in next/previous
            return cursor != elementCount;
        }

        public E next() {
            synchronized (Vector.this) {
                checkForComodification();
                int i = cursor;
                if (i >= elementCount)
                    throw new NoSuchElementException();
                cursor = i + 1;
                return elementData(lastRet = i);
            }
        }

        public void remove() {
            if (lastRet == -1)
                throw new IllegalStateException();
            synchronized (Vector.this) {
                checkForComodification();
                Vector.this.remove(lastRet);
                expectedModCount = modCount;
            }
            cursor = lastRet;
            lastRet = -1;
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }


对比 ArrayList 中的 Itertor 迭代器：

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

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

同样是通过 synchronized 加锁的方式实现数据的同步；


至此，可以得出结论，Vector 主要是在 ArrayList 的基础之上针对数据的操作进行加锁保护，从而实现数据的同步性；












