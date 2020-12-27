## LinkList 简介

链表（Linked list）是一种常见的基础数据结构，是一种线性表，但是并不会按线性的顺序存储数据，而是在每一个节点里存到下一个节点的地址。
所有操作都是按照双重链接列表的需要执行的。在列表中编索引的操作将从开头或结尾遍历列表（从靠近指定索引的一端）。

LinkList 的双向链表数据结构图：

![双向链表](https://github.com/harusty/image/blob/master/java/linklist/双向链表.png)

LinkList 的主要继承关系：

![LinkList主要继承关系图](https://github.com/harusty/image/blob/master/java/linklist/linklist_01.png)


## LinkList 简单用法

和 ArrayList 类似，提供增删改查的基本用法：
    	
		LinkedList<String> heros = new LinkedList<>();
		heros.add("程咬金");
		heros.add("盾山");
		heros.add("孙悟空");
		heros.add("猪八戒");
		heros.add(0, "亚瑟");
		heros.addFirst("庄周");
		heros.addLast("狄仁杰");
		heros.remove(0);
		heros.remove("程咬金");
		heros.set(0, "高渐离");
		heros.get(0);
		heros.getFirst();
		heros.getLast();
		heros.clear();
		System.out.print(heros.isEmpty());



## LinkList 源码解析

### LinkList 的构造方法

    public LinkedList() {

    }

 
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

LinkList 提供带无参和带集合参数构造方法，这里无参构造方法什么事情都没做，推测相关初始化工作在添加时进行。

### LinkList 的 add 方法

    public boolean add(E e) { // 默认添加队列末尾
        linkLast(e);
        return true;
    }

    public void add(int index, E element) {
        checkPositionIndex(index);// 检测 index 的有效性

        if (index == size) // 添加到队列末尾
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

LinkList 提供两种方式 add 方法，这里可以看到主要逻辑在 linkLast 和 LinkBefore 方法中。先看 linkLast 方法。

    void linkLast(E e) {
        final Node<E> l = last;// 原最后一位数据
        final Node<E> newNode = new Node<>(l, e, null);// 构造新的数据 Node
        last = newNode;
        if (l == null)// 原来数据为空的情况
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

以上即为双链表结构的增加操作。

1. 先获取原队列末尾的数据结构 l;
2. 通过原队列末尾的数据结构 l 为参数构造将要插入的数据结构 newNode;
3. 如果原队列末尾数据为空（即原 LinkList 为空），新数据设置到链表 first 中，否则将原最后位数据的后继位指向新的数据 newNode；


链表结构操作完成后，对 size 和 modCount 进行自增操作。linkLast 即默认添加到队列末尾。

接下来 LinkBefore 方法稍微复杂一点。

    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {// 查看是否大于队列一半。加快查找速度，后续遍历次数降低。
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x; // 通过遍历找到即将插入位置的原 Node 数据结构
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;// 通过遍历找到即将插入位置的原 Node 数据结构
        }
    }

    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }

LinkBefore 主要涉及一下步骤：

1. node 方法查找即将插入位置的前一位；
2. linkBefore 方法中获取到原队列 index 位的前一个数据 pred；
3. 创建新的数据，根据前一个数据 pred 和原 index 位数据 succ 创建将要新增的 newNode 数据；
4. pred 数据的下一位指向新数据 newNode；
5. 如果 pred 为空，newNode 设置到 first 中。否则 pred 的后继位指向 newNode；


链表结构操作完成后，对 size 和 modCount 进行自增操作。linkLast 即默认添加到队列末尾。

addFirst 和 addLast 方法

    public void addFirst(E e) {
        linkFirst(e);
    }

    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null) // LinkList 无数据
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }


    public void addLast(E e) {
        linkLast(e);
    }

    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null) // LinkList 无数据
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

上述 add 方法中知道每次增加都会判断 last 和 first 对其赋值，这里实则就是对 first 和 last 进行赋值以及原数据的重新指向。


### LinkList 的 remove 方法

    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }

    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;// 拿到当前 index 数据的后一位
        final Node<E> prev = x.prev;// 拿到当前 index 数据的前一位

        if (prev == null) { // prev 为空表明当前数据在第一位 则下一位赋值为 first
            first = next;
        } else { 
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {// next 为空表明当前数据在最后一位 则上一位赋值为 first
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }

这里检测 index 合法后，主要关注 unlink 方法。 unlink 对当前数据是否在队列首位或者最后一位进行判断后，对相应当前位的前后位指向进行重新赋值完成删除操作。同时 size 和 modCount 自减。

    public boolean remove(Object o) {
        if (o == null) { // 查找队列中为 null 的数据
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

remove 数据对象其实就是先进行某个对象所在 index 数据查找后再执行 unlink 方法，不再累述。同时可以看到 LinkList 也是可以存储 null  数据的。

### LinkList 的 set 方法

    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }

set 方法调用 node 方法获取数据后进行替换即可。


### LinkList 的 get 方法

    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }

    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }

    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

get 获取到当前数据拿值即可，而 first 和 last 的定义使得取首位和末尾的值更加方便。


### LinkList 的数据遍历

		// 方式1
		for(int i = 0;i < heros.size();i++){
			System.out.print(heros.get(i));
		}
		
		// 方式2
		 Iterator it = heros.iterator();
	     while(it.hasNext()){
	    	 System.out.print("hero===" +  it.next());
	     }

方式1不再累述，主要看看 LinkList 的 Iterator 实现方式，和 ArrayList 又有什么区别？

    public ListIterator<E> listIterator() {// AbstractList 类中定义
        return listIterator(0);
    }

    public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);
        return new ListItr(index);
    }

    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;// 上一次返回的数据
        private Node<E> next;
        private int nextIndex;// 当前队列位置
        private int expectedModCount = modCount;

        ListItr(int index) {
            // assert isPositionIndex(index);
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }

        public boolean hasNext() {
            return nextIndex < size;
        }

        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }

        public boolean hasPrevious() {
            return nextIndex > 0;
        }

        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }

        public int nextIndex() {
            return nextIndex;
        }

        public int previousIndex() {
            return nextIndex - 1;
        }

        public void remove() {
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned)
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;
        }

        public void set(E e) {
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }

        public void add(E e) {
            checkForComodification();
            lastReturned = null;
            if (next == null)// 是否是最后一个元素，构造方法中最后一个元素会将 next 置空，next 到最后一个也有可能为空
                linkLast(e);
            else
                linkBefore(e, next);
            nextIndex++;
            expectedModCount++;
        }

        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            while (modCount == expectedModCount && nextIndex < size) {
                action.accept(next.item);
                lastReturned = next;
                next = next.next;
                nextIndex++;
            }
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

 ListItr 实现 ListIterator，而 ListIterator 实现 Iterator，相对于 ArrayList 中实现 Iterator 的方式，ListIterator 额外提供了 previous 、hasPrevious、nextIndex、previousIndex、set、add 等方法。同时 ListItr 也是 LinkList 的内部类。

ListItr 中几个变量解释，lastReturned 表示上一次的数据，nextIndex 表示当前最新的队列位置。

ListItr 构造方法

获取 index 处的数据 Node，同时赋值 index 给 newIndex。 

hasNext 方法

比对 nextIndex 是否小于 LinkList 队列的 size。

next 方法

获取 next 处的数据并返回，同时存储 lastReturned， next 指向后一位，nextIndex 自增，

hasPrevious

这里可以看到 nextIndex 派上用场了，用于当前位置前面有数据。

previous

获取到 previous 数据，lastReturned 存储变化，nextIndex 自减，返回 previous 的数据。

nextIndex

获取当前位置。

previousIndex

当前位置的上一位。

remove

针对 next == lastReturned 的情况什么时候出现？什么情况下会出现？这里本人也不是很清楚这段代码，有读者看明白麻烦告知。建议不要使用。
expectedModCount 的改变会导致 modCount != expectedModCount 情况报错。此方法在 AbstractList 未他提供，内部访问。

set

设置上一个遍历的数据。

add

会判断是否在队列尾部。 newIndex 增加。此方法在 AbstractList 未他提供，内部访问。

checkForComodification 

和 ArrayList 中类似，当使用 Iterator 过程中随意增加或者删除集合数据结构，可能会出现报错。

到这里 LinkList 的遍历方法基本查看结束，那么和 ArrayList 一样，内部对数据操作时未提供同步方法，所以 LinkList 也是线程不安全的集合。

如何获取线程安全的 LinkList 集合呢？

1. List<String> list = Collections.synchronizedList(new LinkedList<String>());
2. 使用 ConcurrentLinkedQueue 类；














