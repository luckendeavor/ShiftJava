[TOC]

### LinkedList

#### 基础

##### 1. 杂记

- LinkedList 也实现了 List 接口，内部基于**双向链表**实现，所以其特点与 ArrayList 几乎**相反**。
- LinkedList 还实现了 **Deque** 接口，可以按照**队列、栈和双端队列**的方式进行操作。
- LinkedList 也是线程**不安全**的队列。

##### 2. API及基本使用

**笔试面试**的时候可以用来实现**队列、栈**等结构，非常方便，要**记牢几个 API**。实现了Queue 接口，所以 API 有点多。

```java
int size();  	// 它返回此列表中元素的数量
void clear();  	// 它删除列表中的所有元素
Object clone();	// 它用于制作现有链接列表的副本
Object set(int index，Object element);  // 它用于用新元素替换列表中的现有元素
boolean contains(Object element);		// 如果元素存在于列表中，则返回true
boolean add(Object element);			// 它将元素附加到列表的末尾
void add(int index，Object element);	   // 它将元素插入列表中'index'位置
boolean addAll(Collection C);			// 它将一个集合追加到链接列表
boolean addAll(int index，Collection C);// 它将一个集合追加到指定位置的链表中
void addFirst(Object element);			// 它将元素插入列表的开头
void addLast(Object element);			// 它将元素附加在列表的末尾
Object get(int index);	// 它返回列表中位置'index'处的元素。如果索引超出了列表的范围，它会抛出'IndexOutOfBoundsException'
Object getFirst();		// 它返回链表的第一个元素
Object getLast();		// 它返回链接列表的最后一个元素
int indexOf(Object element);	// 如果找到元素，它将返回元素第一次出现的索引。否则，它返回-1
int lastIndexOf(Object element);// 如果找到元素，它将返回元素最后一次出现的索引。否则，它返回-1
Object remove();		// 它用于从列表头部删除并返回元素
Object remove(int index);		// 它删除此列表中位置'index'处的元素。如果列表为空，它会抛出'NoSuchElementException'
boolean remove(Object O);		// 它用于从链表中移除一个特定的元素并返回一个布尔值
Object removeLast();			// 它用于删除并返回链接列表的最后一个元素
```

```java
public class Main {
    public static void main(String[] args) {
        LinkedList<String> linkedList1 = new LinkedList<>();
        String[] arr = {"H", "E", "L", "L", "O"};
        LinkedList<String> linkedList2 = new LinkedList<>(Arrays.asList(arr));
        System.out.println(linkedList2);
    }
}
```



#### 源码解析

##### 1. 内部结点类

基于**双向链表**实现，使用内部节点类 **Node** 存储链表节点信息。

```java
private static class Node<E> {
    E item;			// 数据
    Node<E> next;	// 后继结点指针
    Node<E> prev;	// 前驱结点指针
     
    Node(Node<E> prev, E element, Node<E> next) {
    	this.item = element;
     	this.next = next;
     	this.prev = prev;
 	}
}
```

每个链表存储了 next 和 prev 指针，LinkedList 中的**重要属性**如下

```java
// 链表的节点个数
transient int size = 0;
// 指向头节点的指针
transient Node<E> first;
// 指向尾节点的指针
transient Node<E> last;
```

头尾指针都是 **transient** 修饰的。

<img src="../../JavaNotes/A Java/assets/LinkedList_base-1591168327338.png" alt="1567326494424" style="zoom:60%;" />

##### 2. 添加元素

对于**链表**这种数据结构来说，添加元素的操作无非就是在**表头**插入元素，又或者在**指定位置**插入元素。因为 LinkedList 有**头指针和尾指针**，所以在**表头**进行插入元素只需要 **O(1)** 的时间，而在**指定位置**插入元素则需要先**遍历一下链表**，所以复杂度为 **O(N)**。

在队列尾部和在指定节点之前插入元素，如图所示：

<img src="../../JavaNotes/A Java/assets/LinkedList_add-1591168403304.png" alt="1567329526899" style="zoom:71%;" />

当向指定节点之前插入一个节点时，当前节点的**后继为指定节点**，而**前驱结点为指定节点的前驱节点**。此外，还要修改前驱节点的后继为当前节点，以及后继节点的前驱为当前节点。

最普通的插入元素方法为 **add**。

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

调用了 LinkLast 方法，所以最基本的 **add** 方法其实是**尾插入**的。由于是维护了一个**尾指针**，所以尾插入也很 easy。linkLast 对于包外是无法调用的。

```java
/**
 * 尾插入，即将节点值为e的节点设置为链表的尾节点
 */
void linkLast(E e) {
    // 获取当前尾结点引用
    final Node<E> l = last;
    //构建一个prev值为l,节点值为e,next值为null的新节点newNode
    final Node<E> newNode = new Node<>(l, e, null);
    //将newNode作为尾节点
    last = newNode;
    //如果原尾节点为null，即原链表为null，则链表首节点也设置为newNode
    if (l == null)
        first = newNode;
    else    //否则，原尾节点的next设置为newNode
        l.next = newNode;
    size++;
    modCount++;
}
```

还可以在**指定**的位置加入元素：

```java
/**
 * 在指定位置前插入指定元素
 *
 * @param index   指定元素将被插入的索引
 * @param element 要插入的元素
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public void add(int index, E element) {
    // 判断指定位置是否合法
    checkPositionIndex(index);
    // 如果指定位置在尾部，则直接通过尾插法来插入指定元素
    if (index == size)
        linkLast(element);
    else        // 如果指定位置不是尾部，则添加到指定位置前
        linkBefore(element, node(index));
}
```

**linkBefore** 方法如下，即在**链表中间**插入，在非空节点 succ 之前插入节点值 e。主要就是要各种**修改指针**。

````java
/**
 * 中间插入，在非空节点succ之前插入节点值e
 */
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    // 构建一个prev值为succ.prev,节点值为e,next值为succ的新节点newNode
    final Node<E> newNode = new Node<>(pred, e, succ);
    // 设置newNode为succ的前节点
    succ.prev = newNode;
    // 如果succ.prev为null，即如果succ为首节点，则将newNode设置为首节点
    if (pred == null)
        first = newNode;
    else        // 如果succ不是首节点
        pred.next = newNode;
    size++;
    modCount++;	// 记录修改次数
}
````

看看常用的几个 API 是如何实现的。

```java
public void addFirst(E e) {
    linkFirst(e);
}
```

```java
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}
```

在**头部**加元素时最后都会调用 linkFirst **私有方法**。

```java
/**
 * Links e as first element.
 */
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

对于在**尾部**添加数据，有

```java
public boolean offer(E e) {
    return add(e);
}
```

offer **默认**调用 add 方法。此外还有如下方法。

```java
public void addLast(E e) {
    linkLast(e);
}
```

```java
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

##### 3. 判断存在

```java
public boolean contains(Object o) {
     return indexOf(o) != -1;
 }
 // 从前向后查找，返回“值为对象(o)的节点对应的索引”  不存在就返回-1 
 public int indexOf(Object o) {
      int index = 0;
     // 当传入为null时则找第一个不为null的结点
      if (o == null) {
          for (Entry e = header.next; e != header; e = e.next) {
              if (e.element == null)
                  return index;
              index++;
         }
          // 否则顺序遍历找结点
      } else {
         for (Entry e = header.next; e != header; e = e.next) {
             if (o.equals(e.element))
                 return index;
             index++;
        }
    }
     return -1;
 }
```

**indexOf**(Object o) 判断 o 链表中是否存在节点的 element 和 o 相等，若相等则返回该节点在链表中的索引位置，若不存在则返回 -1。

**contains**(Object o) 方法通过判断 indexOf(Object o) 方法返回的值是否是 -1 来判断链表中是否包含对象 o。

##### 4. 获取数据

**get**(int) 方法用于获取**指定位置**的数据。首先判断位置信息是否**合法**（大于等于0，小于当前 LinkedList 实例的 Size），然后**遍历**到具体位置，获得节点的业务数据（element）并返回。

```java
public E get(int index) {
    // 检查索引合法性
    checkElementIndex(index);
    // 获取指定索引数据
    return node(index).item;
}
```

然后调用 node 的**内部方法**获取元素。注意：由于具有**双向指针**，所以查找时会判断待搜索的**索引**与整个链表的**长度**关系，如果**小于链表长度一半**，那么就从头遍历，否则从尾部遍历，这样最多也就遍历半个链表长度。可以**提高查找效率**。

```java
/**
 * 获取指定下标的结点，index从0开始
 */
Node<E> node(int index) {
    // 如果指定下标小于一半元素数量，则从首结点开始遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    // 否则，从尾结点开始遍历
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

注意细节：**位运算**与直接做除法的区别。先将 index 与长度 size 的**一半**比较，如果 index < size / 2，就只从位置 **0 往后**遍历到位置 index 处，而如果 index > size / 2，就只从位置 size 往前遍历到位置 index 处。

##### 5. 删除数据

**删除**指定位置的元素用 remove 方法。

<img src="../../JavaNotes/A Java/assets/LinkedList_remove.png" alt="1567330247769" style="zoom:71%;" />

```java
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```

unlink 实现真正的删除逻辑。

```java
/**
 * 删除指定非空结点，返回存储的元素
 */
E unlink(Node<E> x) {
    // 获取指定非空结点存储的元素
    final E element = x.item;
    // 获取指定非空结点的后继结点
    final Node<E> next = x.next;
    // 获取指定非空结点的前驱结点
    final Node<E> prev = x.prev;

    /**
     * 如果指定非空结点的前驱结点为空，则指定非空结点的后继结点设为首结点
     * 否则，指定非空结点的后继结点设为指定非空结点的前驱结点的后继结点，
     * 指定非空结点的前驱结点设为null
     */
    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    /**
     * 如果指定非空结点的后继结点为空，则指定非空结点的前驱结点设为尾结点
     * 否则，指定非空结点的前驱结点设为指定非空结点的后继结点的前驱结点，
     * 指定非空结点的后继结点设为null
     */
    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    // 指定非空结点存储的元素设为null
    x.item = null;
    size--;
    // 记录结构变化
    modCount++;
    // 返回指定非空结点存储的元素
    return element;
}
```

与 ArrayList 比较而言，LinkedList 的删除动作**不需要“移动”很多**数据，从而效率更高。而且删除时会**传入**这个结点，所以不用遍历去寻找这个结点，贼快。

##### 6. 其他

其他的很多方法其实都比较简单，多半都是对双向链表的简单操作。LinkedList 实现了 Cloneable 接口，所以看看**克隆**方法。

```java
/**
 * 克隆，浅拷贝
 * @return a shallow copy of this {@code LinkedList} instance
 */
public Object clone() {
    // 先调用默认的clone方法克隆一个对象再说
    LinkedList<E> clone = superClone();

    // 链表初始化
    clone.first = clone.last = null;
    clone.size = 0;
    clone.modCount = 0;

    // 插入结点这里是对每个节点元素进行克隆
    for (Node<E> x = first; x != null; x = x.next)
        clone.add(x.item);
    // 返回克隆后的对象引用
    return clone;
}

private LinkedList<E> superClone() {
    try {
        return (LinkedList<E>) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new InternalError(e);
    }
}
```



#### 与ArrayList比较

- **数据结构与实现**：ArrayList 基于**动态数组**实现，LinkedList 基于**双向链表**实现；
- **线程安全性**：ArrayList 和 LinkedList **都不**保证线程安全。
- **元素访问**：ArrayList 支持快速**随机访问**，LinkedList **不支持**。
- **增删元素效率**： ArrayList 采用**数组**存储，插入元素到数组末尾复杂度为 O(1)，但是在指定位置插入元素与删除元素复杂度为 O(N)。LinkedList 采用**链表**存储，在删除元素的时候不需要挪动后面的元素，操作基本都是修改指针，LinkedList 在**任意位置**添加删除元素**更快**。
- **内存空间占用**：ArrayList 的空间浪费主要体现在 list 列表的结尾会预留一定的**容量空间**，而 LinkedList 的空间花费则体现在它的每一个结点都需要消耗比 ArrayList 更多的空间。