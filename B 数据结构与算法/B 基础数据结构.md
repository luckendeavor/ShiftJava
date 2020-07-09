[TOC]

### 基本数据结构

#### 结构与存储

##### 1. 线性结构与非线性结构

###### (1) 线性结构

线性结构作为最常用的数据结构，其特点是数据元素之间存在**一对一**的线性关系。线性结构有两种不同的存储结构，即**顺序存储结构和链式存储结构**。顺序存储的线性表称为顺序表，顺序表中的存储元素是连续的。链式存储的线性表称为链表，链表中的存储元素不一定是连续的，元素节点中存放数据元素以及相邻元素的地址信息。

线性结构见的有：**组队列、链表和栈**。

###### (2) 非线性结构

非线性结构包括：二维数组，多维数组，广义表，树结构，图结构。

##### 2. 存储方式的分析

###### (1) 数组存储

优点：通过下标方式访问元素，速度快。**对于有序数组**，还可使用二分查找提高检索速度。
缺点：如果要检索具体某个值，或者插入值(按一定顺序)**会整体移动**，效率较低。可能还涉及到数组扩容，这中间的数据复制开销较大。

###### (2) 链式存储

优点：在一定程度上对数组存储方式有优化 (比如：插入一个数值节点，只需要将插入节点，链接到链表中即可，
删除效率也很好)。

缺点：在进行检索查找时，效率仍然较低，比如(检索某个值，需要从头节点开始遍历) 。

###### (3) 树存储

能提高数据**存储，读取**的效率,  比如利用 **二叉排序树**(Binary Sort Tree)，既可以保证数据的检索速度，同时也可以保证数据的插入，删除，修改的速度。



#### 数组

##### 1. 稀疏数组

应用场景：记录一个**棋盘**。或者**地图**等，如下所示。

<img src="../../JavaNotes/B 数据结构与算法/assets/1565000508963-1569397664724.png" alt="1565000508963" style="zoom:67%;" />

因为该二维数组的很多值是**默认值 0**, 因此记录了很多没有意义的数据，考虑使用稀疏数组。

当一个数组中**大部分元素为０**，或者为同一个值的数组时，可以使用稀疏数组来保存该数组。

稀疏数组的处理方法是:

- 记录数组一共有**几行几列**，有多少个**不同的值**；
- 把具有不同值的元素的行列及值记录在一个小规模的数组中，从而缩小程序的规模。
- 行不确定，但是列是三列的动态数组。

**转化举例**

<img src="../../JavaNotes/B 数据结构与算法/assets/1565001222410-1569397664724.png" alt="1565001222410" style="zoom:60%;" />

上述数组 0 较多，使用稀疏数组表示，该数组第 0 个元素记录原始数组的行数、列数和有效数据数。后面的值就是记录有效的位置和数据值。



#### 线性表

##### 1. 概述

**线性表**存储方式分为**==顺序存储和链式存储==**。顺序存储使用数组进行存储。链式存储使用链表进行存储。

对比表格如下：

|                |             顺序表             |                    链式表                    |
| :------------: | :----------------------------: | :------------------------------------------: |
|  **存储方式**  |            **数组**            |                   **链表**                   |
| **地址连续性** |      内存地址**连续**存储      | 节点的地址不是连续的，是通过**指针**连起来的 |
|    **查找**    |       方便。直接内存寻址       |             不方便。需要遍历链表             |
|    **插入**    | 不方便。后面的元素需要整体后移 |              方便。修改指针即可              |
|    **删除**    | 不方便。后面的元素需要整体前移 |              方便。修改指针即可              |

**使用**：当线性表中的元素个数变化较大或者根本不知道有多大时，最好用单链表结构，这样可以不需要考虑存储空间的大小问题。而如果事先知道线性表的大致长度，用顺序存储结构效率会高很多。

##### 2. 数组实现线性表

元素的存储空间是**连续**的。在内存中是以顺序存储，内存划分的**区域是连续**的。如下图。

<img src="assets/image-20191216214220866.png" alt="image-20191216214220866" style="zoom:67%;" />

**插入元素**：在**指定的位置**添加元素时，需要把**后面的元素整体移动**，效率不高。位置编号越大， 插入时所需要移动的元素越少，时间越快。当加入的元素**超过容量之后也需要扩展数组**，并把原来的数组内容进行**复制**，这里也是效率不高。如插入 11 需要将 10 和 2 整体后移再插入。

<img src="assets/image-20191216214444081.png" alt="image-20191216214444081" style="zoom:60%;" />

**删除元素**：删除指定位置的元素之后，也需要把后面的元素**整体往前移动补空位**。如下图删除 3 之后，需要把后面的数据整体前移。

<img src="assets/image-20191216214610372.png" alt="image-20191216214610372" style="zoom:60%;" />

**缺点**就是**添、删**的时候比较麻烦，特别是添和改的时候要**移动数组**，数组容易越界。可能出现数组存储满的情况，需要进行**扩容并复制**。

#####  3. 单向链表

元素在内存中**不一定**是连续存储的。如下图所示。

<img src="assets/image-20191216215559974.png" alt="image-20191216215559974" style="zoom:57%;" />

链表是以**结点**方式来存储数据，每个结点包含 **data 域， next 域**，用于指向下一个结点。链表中的节点不一定是连续存储的。如上图头结点指针 head 指向 150 地址的 a1 元素。

##### 4. 双向链表

单向链表**查找的方向只能是一个方向**，而双向链表可以向前或者向后查找。单向链表**不能自我删除**，需要靠**辅助节点** ，而双向链表，则可以**自我删除**，所以前面我们单链表删除节点时，总是需要先找到 temp, temp 是待删除节点的前一个节点。双向链表维护了一个 next 指向下一个结点，维护了一个 pre 指向上一个结点。因此添加删除操作等需要**同时操作 next 和 pre** ，不要遗忘了。

<img src="assets/1565089271980-1569397664725.png" alt="1565089271980" style="zoom:67%;" />

#### 栈

##### 1. 概述

- 栈的操作端通常被称为**栈顶**，另一端被称为**栈底**。所有添加都位于**栈顶**。栈顶是最新的数据。删除也是先删除栈顶。**后进先出**（LIFO）结构。
- 基本操作：**入栈**（push）：增加元素；**出栈**（pop）：删除元素；**查看**（peak）：获取栈顶但不删除。
- 顺序存储的栈称为**顺序栈**；链式存储的栈称为**链式栈**。
- 栈的常见应用：十进制转 N 进制、行编辑器、校验括号是否匹配、中缀表达式转后缀表达式、表达式求值等。

栈的抽象接口如下：

```java
public interface MyStack<Item> extends Iterable<Item> {
    MyStack<Item> push(Item item);
    Item pop() throws Exception;
    boolean isEmpty();
    int size();
}
```

##### 2. 栈的实现

###### (1) 栈的数组实现

使用可变大小数组实现栈，容量不够时可以自动**扩容**。使用数组实现栈则数组的第一个位置是**栈底**，数组**最后占用**的位置才指向**栈顶**，否则压栈会整体**移动**元素位置。

<img src="assets/image-20191217125925596.png" alt="image-20191217125925596" style="zoom:42%;" />

弹栈操作可以将所有的元素依次弹栈即可。可以看看 **ArrayDeque** 的源码。这里给个demo。

```java
public class ArrayStack<Item> implements MyStack<Item> {

    // 栈元素数组，只能通过转型来创建泛型数组
    private Item[] a = (Item[]) new Object[1];

    // 元素数量
    private int N = 0;

    @Override
    public MyStack<Item> push(Item item) {
        check();
        a[N++] = item;
        return this;
    }

    @Override
    public Item pop() throws Exception {

        if (isEmpty()) {
            throw new Exception("stack is empty");
        }
        Item item = a[--N];
        check();
        // 避免对象游离
        a[N] = null;
        return item;
    }

    private void check() {
        if (N >= a.length) {
            resize(2 * a.length);
        } else if (N > 0 && N <= a.length / 4) {
            resize(a.length / 2);
        }
    }

    /**
     * 调整数组大小，使得栈具有伸缩性
     */
    private void resize(int size) {
        Item[] tmp = (Item[]) new Object[size];
        for (int i = 0; i < N; i++) {
            tmp[i] = a[i];
        }
        a = tmp;
    }

    @Override
    public boolean isEmpty() {
        return N == 0;
    }

    @Override
    public int size() {
        return N;
    }

    @Override
    public Iterator<Item> iterator() {

        // 返回逆序遍历的迭代器
        return new Iterator<Item>() {
            private int i = N;
            @Override
            public boolean hasNext() {
                return i > 0;
            }
            @Override
            public Item next() {
                return a[--i];
            }
        };
    }
}
```

###### (2) 栈的链式实现

需要使用链表的**头插法或尾插法**来实现，因为**头插法**中最后压入栈的元素在**链表的开头**，它的 next 指针指向前一个压入栈的元素，在弹出元素时就可以通过 **next** 指针遍历到前一个压入栈的元素从而让这个元素成为新的栈顶元素。

使用单链表实现栈，则**首节点**应该指向**栈顶**元素。

<img src="assets/image-20191217130931332.png" alt="image-20191217130931332" style="zoom:42%;" />

==**压栈**==操作就是分配一个新结点，使之指向目前的栈链，如下图所示。

<img src="assets/image-20191217131311652.png" alt="image-20191217131311652" style="zoom:39%;" />

a) 构造**新结点**并指向栈顶结点；

b) 头指针指向**栈顶新节点**。

**==弹栈==**操作就是将**首节点的引用赋给 topNode**，从而出栈。故将 topNode 指向链中的第二个结点。如下图所示。

![image-20191217131733305](assets/image-20191217131733305.png)

可以参考 LinkedList 的源码实现。

单链表一般使用头插法，而双链表则头插法和尾插法都可。

```java
public class ListStack<Item> implements MyStack<Item> {

    private Node top = null;
    private int N = 0;


    private class Node {
        Item item;
        Node next;
    }


    @Override
    public MyStack<Item> push(Item item) {

        Node newTop = new Node();

        newTop.item = item;
        newTop.next = top;

        top = newTop;

        N++;

        return this;
    }


    @Override
    public Item pop() throws Exception {

        if (isEmpty()) {
            throw new Exception("stack is empty");
        }

        Item item = top.item;

        top = top.next;
        N--;

        return item;
    }


    @Override
    public boolean isEmpty() {
        return N == 0;
    }


    @Override
    public int size() {
        return N;
    }


    @Override
    public Iterator<Item> iterator() {

        return new Iterator<Item>() {

            private Node cur = top;


            @Override
            public boolean hasNext() {
                return cur != null;
            }


            @Override
            public Item next() {
                Item item = cur.item;
                cur = cur.next;
                return item;
            }
        };

    }
}
```

###### (3) 栈的向量Vector类实现

Java 类库 **Vector 类**，其实例（称为向量）的行为类似于一个可变大小的**数组**。可用于构造栈。Vector 类的实现基于**==动态可变大小的数组==**，但是其内部已经实现了许多方法，可以轻松实现栈结构。

如果使用向量实现栈，则向量的**首元素**应该指向**栈底元素**。而向量的最后的占用位置指向栈顶元素。



#### 队列

##### 1. 概述

- 先进先出 **FIFO** 结构。添加元素在**后端**，出队列在**前端**。

##### 2. 队列的实现

###### (1) 队列的数组实现

使用**数组**来存放队列，维护一个 **frontIndex** 和 **backIndex** 来指示**队头队尾**。

使用==**带一个不用位置的循环数组**==来实现队列。即该数组中始终有一个位置**空缺**， 可以将其放在**队尾**。此时判断队列的**空与满有不同的条件**。所以存储对象的数组应比设定的大小**多 1**。

下图是队列**满与空**流程图。

<img src="assets/image-20191217144730730.png" alt="image-20191217144730730" style="zoom:33%;" />

<img src="assets/image-20191217144808583.png" alt="image-20191217144808583" style="zoom:35%;" />

队列**满**时：（公式源自《数据结构与抽象 Java 版》）

```java
frontIndex == (backIndex + 2) % queue.length
6 == (4 + 2)% 7    
```

队列**空**时：

```java
frontIndex == (backIndex + 1) % queue.length
```

**入队**即在**队列后端**添加元素，数组是**循环**的，需要使用 **==%==** 来确定索引位置。

**出队**过程即把 frontIndex 指向**下一个**元素。

**数组扩容**：容量不足实现**扩容**时，与之前的数组扩容有所差距如下图。将原来的循环队列**全都复制到新队列的开始**处。

<img src="assets/image-20191217145131381.png" alt="image-20191217145131381" style="zoom:40%;" />

###### (2) 队列的链式实现

队列的两端在**链的两端**。队列的**前端**放在链的**开头**，队列的**后端**放在链的**链尾**。**firstNode** 指向队列前端，**lastNode** 指向队列后端，如下图所示。当队列为**空**时两个**都为 null**。

<img src="assets/image-20191217153931658.png" alt="image-20191217153931658" style="zoom:40%;" />

**入队**是添加新结点到后端，需要判断链是否为**空**！如果是空链，则添加元素之后如下，即 firstNode 和 lastNode 均指向新结点。

<img src="assets/image-20191217154220360.png" alt="image-20191217154220360" style="zoom:40%;" />

如果是一般结点**入队**，则链中最后一个结点和 lastNode 指向新结点，如下。

- **最后一个结点**指向新结点；
- **lastNode** 指向新结点。

<img src="assets/image-20191217154744257.png" alt="image-20191217154744257" style="zoom:40%;" />

如果有多个结点，**出队**将 firstNode 指向链的第二个结点。如果队列中**仅有一个节点**，则出队之后为空。如下所示。

<img src="assets/image-20191217155148065.png" alt="image-20191217155148065" style="zoom:40%;" />

###### (3) 队列的循环链式实现

循环链中最后一个结点指向**第一个**结点。每个结点的 **nextNode** 域**不会为 null**。如下图有指向最后结点的外部引用的循环链。

<img src="assets/image-20191217155629685.png" alt="image-20191217155629685" style="zoom:40%;" />

只需要**一个 lastNode** 数据域即可，它属于**链尾**，使用 

```java
lastNode.getNextNode();
```

 就可以获取到**链头**。

##### 3. 双端队列（deque）

能在队列的**前端和后端**进行添加、删除与获取操作。行为上类似于**双端栈**。

##### 4. 优先级队列（Priority Queue）

根据**优先级**组织队列中的对象。对象优先级通过 **compareTo()** 方法确定。需要按项的优先级对项进行排序。Java中有对应的类库：**PriorityQueue** 类。重点是**堆**实现。

##### 5. Java队列实现

###### (1) Queue接口

```java
// 将元素插入队列
boolean add(E e);
// 将元素插入队列，与add相比，在容量受限时应该使用这个
boolean offer(E e);
// 将队首的元素删除，队列为空则抛出异常
E remove();
// 将队首的元素删除，队列为空则返回null
E poll();
// 获取队首元素，但不移除，队列为空则抛出异常
E element();
// 获取队首元素，但不移除，队列为空则返回null
E peek();
void clear();
int size();
```

入队、出队和取值两个一组，一个**抛出异常，一个返回 null**。

###### (2) Deque接口

继承于 Queue 接口。两种方法**成对**，出现问题时一个出现**异常**一个返回 **NULL**。

| 操作类型 | 第一个元素（Deque实例的开头） | 最后一个元素（Deque实例的结尾） |
| :------: | :---------------------------: | :-----------------------------: |
|   插入   |  addFirst(e)  offerFirst(e)   |    addLast(e)  offerLast(e)     |
|   移除   |  removeFirst()  pollFirst()   |    removeLast()  pollLast()     |
|   检索   |    getFirst()  peekFirst()    |      getLast()  peekLast()      |

###### (3) ArrayDeque类

实现了 **Deque 接口**。因为 Deque 接口声明了对应于**双端队列、队列和栈**的方法，因此可以用 ArrayDeque 类来创建这些数据集合的实例。==**不要**使用 Stack 类来创建栈，而是使用 ArrayDeque 类==。



#### 符号表

符号表是一种**存储键值对**的数据结构，支持两种操作：**插入**，即将一组新的键值存存入表中；**快速查找**，即根据特定的键得到相应的值。符号表的主要目的就是将**一个键和一个值**联系起来。

符号表分为**有序和无序**两种，有序符号表主要指支持 min()、max() 等根据键的**大小关系**来实现的操作，有序符号表保证的是**键**的有序性，比较的是键。

符号表有**多种实现**方式。**树或者散列表**等都可以。散列表的查找算法也可以基于前一节的基础查找算法，只需要有适当改变即可。

##### 1. 基础实现

###### (1) 链表实现无序符号表

一般不用这种实现的，只是理论上可以。

```java
public class ListUnorderedST<Key, Value> implements UnorderedST<Key, Value> {

    private Node first;

    private class Node {
        Key key;
        Value value;
        Node next;

        Node(Key key, Value value, Node next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    @Override
    public int size() {
        int cnt = 0;
        Node cur = first;
        while (cur != null) {
            cnt++;
            cur = cur.next;
        }
        return cnt;
    }

    @Override
    public void put(Key key, Value value) {
        Node cur = first;
        // 如果在链表中找到节点的键等于 key 就更新这个节点的值为 value
        while (cur != null) {
            if (cur.key.equals(key)) {
                cur.value = value;
                return;
            }
            cur = cur.next;
        }
        // 否则使用头插法插入一个新节点
        first = new Node(key, value, first);
    }

    @Override
    public void delete(Key key) {
        if (first == null)
            return;
        if (first.key.equals(key))
            first = first.next;
        Node pre = first, cur = first.next;
        while (cur != null) {
            if (cur.key.equals(key)) {
                pre.next = cur.next;
                return;
            }
            pre = pre.next;
            cur = cur.next;
        }
    }

    @Override
    public Value get(Key key) {
        Node cur = first;
        while (cur != null) {
            if (cur.key.equals(key))
                return cur.value;
            cur = cur.next;
        }
        return null;
    }
}
```

###### (2) 二分查找实现有序符号表

使用一对平行**数组**，一个存储键一个存储值。二分查找的 rank() 方法至关重要，当键在表中时，它能够知道该键的位置；当键不在表中时，它也能知道在何处插入新键。

二分查找最多需要 logN+1 次比较，使用二分查找实现的符号表的查找操作所需要的时间最多是对数级别的。但是插入操作需要移动数组元素，是线性级别的。

```java
public class BinarySearchOrderedST<Key extends Comparable<Key>, Value> implements OrderedST<Key, Value> {

    private Key[] keys;
    private Value[] values;
    private int N = 0;

    public BinarySearchOrderedST(int capacity) {
        keys = (Key[]) new Comparable[capacity];
        values = (Value[]) new Object[capacity];
    }

    @Override
    public int size() {
        return N;
    }

    @Override
    public int rank(Key key) {
        int l = 0, h = N - 1;
        while (l <= h) {
            int m = l + (h - l) / 2;
            int cmp = key.compareTo(keys[m]);
            if (cmp == 0)
                return m;
            else if (cmp < 0)
                h = m - 1;
            else
                l = m + 1;
        }
        return l;
    }

    @Override
    public List<Key> keys(Key l, Key h) {
        int index = rank(l);
        List<Key> list = new ArrayList<>();
        while (keys[index].compareTo(h) <= 0) {
            list.add(keys[index]);
            index++;
        }
        return list;
    }

    @Override
    public void put(Key key, Value value) {
        int index = rank(key);
        // 如果找到已经存在的节点键为 key，就更新这个节点的值为 value
        if (index < N && keys[index].compareTo(key) == 0) {
            values[index] = value;
            return;
        }
        // 否则在数组中插入新的节点，需要先将插入位置之后的元素都向后移动一个位置
        for (int j = N; j > index; j--) {
            keys[j] = keys[j - 1];
            values[j] = values[j - 1];
        }
        keys[index] = key;
        values[index] = value;
        N++;
    }

    @Override
    public Value get(Key key) {
        int index = rank(key);
        if (index < N && keys[index].compareTo(key) == 0)
            return values[index];
        return null;
    }

    @Override
    public Key min() {
        return keys[0];
    }

    @Override
    public Key max() {
        return keys[N - 1];
    }
}
```

##### 2. 其他实现

其他实现参考**二叉查找树、红黑树、散列表**等。

##### 3. 符号表算法比较

|             算法             | 插入 | 查找 | 是否有序 |
| :--------------------------: | :--: | :--: | :------: |
|   **链表实现**的无序符号表   |  N   |  N   |    是    |
| **二分查找实现**的有序符号表 |  N   | logN |    是    |
|        **二叉查找树**        | logN | logN |    是    |
|        **2-3 查找树**        | logN | logN |    是    |
|    **拉链法**实现的散列表    | N/M  | N/M  |    否    |
|  **线性探测法**实现的散列表  |  1   |  1   |    否    |

**无序**时应当优先考虑**散列表**，当需要**有序**性操作时使用**红黑树**。

符号表的各种实现方式优缺点对比。

|         数据结构         |                          优点                          |                          缺点                          |
| :----------------------: | :----------------------------------------------------: | :----------------------------------------------------: |
|   **链表**（顺序查找）   |                     适用于小型问题                     |                   对大型符号表效率低                   |
| **有序数组**（二分查找） | 最优的**查找效率和空间**需求，能够进行有序性相关的操作 |                  **插入**操作很**慢**                  |
|        **散列表**        |         能够快速的**查找和插入**常见类型的数据         | **无法进行有序性**相关的操作，链接与空节点需要额外空间 |
|      **二叉查找树**      |         实现简单，能够进行**有序性**的相关操作         |          没有性能上界的保证，链接需要额外空间          |
|    **平衡二叉查找树**    |   最优的**查找和插入**效率，能够进行有序性相关的操作   |                   链接需要额外的空间                   |

##### 4. Java的符号表实现

- java.util.TreeMap：**红黑树**。
- java.util.HashMap：**拉链法的散列表。**





#### 散列表

散列表是**符号表**的一种实现方式。散列表类似于数组，可以把散列表的散列值看成数组的索引值。访问散列表和访问数组元素一样快速，它可以在常数时间内实现**查找和插入**操作。由于无法通过散列值知道键的大小关系，因此散列表**无法实现有序性**操作。

散列表（Hash table，也叫哈希表），是根据**键值(Key value)** 而直接进行访问的数据结构。也就是说它通过把**键值映射到表中一个位置**来访问记录，以加快查找的速度。这个**映射函数叫做==散列函数==**，存放记录的数组叫做散列表。

##### 1. 散列函数

###### (1) 定义

每个**关键字**被映射到从 **0 到 TableSize - 1** 这个范围的某个数，并被放到合适的单元中，这个映射就是**==散列函数==**。

对于一个大小为 **M** 的散列表，散列函数能够把任意键转换为 **[0, M - 1]** 内的正整数，该正整数即为 **hash** 值。

我们需要寻找一个散列函数，该函数要在单元之间**均匀的分配**关键字。

散列函数应该满足以下**三个条件**：

- **一致性**：相等的键应当有相等的 hash 值，两个键相等表示调用 equals() 返回的值相等。
- **高效性**：计算应当简便，有必要的话可以把 hash 值缓存起来，在调用 hash 函数时直接返回。
- **均匀性**：所有键的 hash 值应当均匀地分布到 [0, M-1] 之间，如果不能满足这个条件，有可能产生很多冲突，从而导致散列表的性能下降。

###### (2) 常见散列函数

**除留余数法（取模）**可以将整数散列到 [0, M-1] 之间，例如一个正整数 k，计算 k % M 既可得到一个 [0, M-1] 之间的 hash 值。注意 **M 最好是一个素数**，否则无法利用键包含的所有信息。例如 M 为 10<sup>k</sup>，那么只能利用键的后 k 位。

对于其它数，可以将其**转换成整数**的形式，然后利用除留余数法。例如对于**浮点数**，可以将其的**二进制形式**转换成整数。

对于**多部分组合**的类型，每个部分都需要计算 hash 值，这些 hash 值都具有同等重要的地位。为了达到这个目的，可以将该类型看成 R 进制的整数，每个部分都具有不同的**权值**。

例如，字符串的散列函数实现如下：

```java
int hash = 0;
for (int i = 0; i < s.length(); i++)
    hash = (R * hash + s.charAt(i)) % M;
```

再比如，拥有**多个成员**的自定义类的哈希函数如下：

```java
int hash = (((day * R + month) % M) * R + year) % M;
```

R 通常取 **31**。

Java 中的 **hashCode**() 实现了哈希函数，但是默认使用对象的**内存地址值**。在使用 hashCode() 时，应当结合除留余数法来使用。因为**内存地址是 32 位整数**，我们只需要 **31 位**的非负整数，因此应当屏蔽符号位之后再使用除留余数法。

```java
int hash = (x.hashCode() & 0x7fffffff) % M;
```

使用 Java 的 HashMap 等自带的哈希表实现时，只需要去实现 **Key 类型**的 hashCode() 函数即可。Java 规定 hashCode() 能够将键均匀分布于所有的 32 位整数，Java 中的 String、Integer 等对象的 hashCode() 都能实现这一点。以下展示了**自定义类型**如何实现 hashCode()：

```java
public class Transaction {

    private final String who;
    private final Date when;
    private final double amount;

    public Transaction(String who, Date when, double amount) {
        this.who = who;
        this.when = when;
        this.amount = amount;
    }

    // 自定义散列函数
    public int hashCode() {
        int hash = 17;
        int R = 31;
        hash = R * hash + who.hashCode();
        hash = R * hash + when.hashCode();
        hash = R * hash + ((Double) amount).hashCode();
        return hash;
    }
}
```

##### 2. 散列冲突

当两个关键字散列到**同一个值**的时候，就产生了散列冲突。散列表存在**==冲突==**，也就是两个**不同的键可能有相同的 hash 值**。解决散列冲突的简单方法有：**拉链法和开放定址法**。

######  (1) 拉链法

拉链法使用**数组 + 链表**来存储 hash 值相同的键，从而解决冲突（比如 HashMap 类）。**查找**需要分两步，首先查找 Key 所在的**链表**（对应的数据槽），然后在链表中**顺序查找**。对于 N 个键，M 条链表 (N > M)，如果哈希函数能够满足均匀性的条件，每条链表的**长度趋向于 N/M**，因此未命中的查找和插入操作所需要的比较次数为 \~N/M。

<img src="assets/1563523769927.png" alt="1563523769927" style="zoom:70%;" />

这个详细可看 **HashMap** 的源码。

###### (2) 开放地址法

线性探测法使用**空位**来解决冲突，当冲突发生时，**向前探测一个空位**来存储冲突的键。

更常见的是，单元 h~0~(x), h~1~(x), ....  相继被试选，其中

***h ~i~ (x) = (hash(x) + f ~i~ (x) ) mod TableSize, 且 f(0) = 0*。**

**函数 f  是解决冲突解决方法（冲突函数）**。此时**所有的数据都需要放入表**内，所以**需要的表比拉链法散列需要的表更大**。把这样的表叫做**探测散列表**。

使用线性探测法，数组的大小 M 应当**大于键的个数 N**（M>N)。

<div align="center"> <img src="assets/0dbc4f7d-05c9-4aae-8065-7b7ea7e9709e.gif" width="350px"> </div><br>

现考察三种具体的冲突解决方案。**线性探测法、平方探测法、双散列法。**

> **线性探测法**

线性探测法中典型的情形是**冲突函数 f 为一次函数**：

```
f(i) = i
```

这相当于相继**逐个探测单元**（必要时可以回绕）以查找出一个**空**单元。

存在的**问题**：有时候占据的单元会形成一些**区块**，其结果成为==**一次聚集**==，就是说散列到**区块**中的任何关键字都需要**多次尝试**才能解决冲突。

**代码实现**

```java
public class LinearProbingHashST<Key, Value> implements UnorderedST<Key, Value> {

    private int N = 0;
    private int M = 16;
    private Key[] keys;
    private Value[] values;

    public LinearProbingHashST() {
        init();
    }

    public LinearProbingHashST(int M) {
        this.M = M;
        init();
    }

    private void init() {
        keys = (Key[]) new Object[M];
        values = (Value[]) new Object[M];
    }

    private int hash(Key key) {
        return (key.hashCode() & 0x7fffffff) % M;
    }
}
```

**查找**

```java
public Value get(Key key) {
    for (int i = hash(key); keys[i] != null; i = (i + 1) % M)
        if (keys[i].equals(key))
            return values[i];

    return null;
}
```

**插入**

```java
public void put(Key key, Value value) {
    resize();
    putInternal(key, value);
}

private void putInternal(Key key, Value value) {
    int i;
    for (i = hash(key); keys[i] != null; i = (i + 1) % M)
        if (keys[i].equals(key)) {
            values[i] = value;
            return;
        }

    keys[i] = key;
    values[i] = value;
    N++;
}
```

**删除**

删除操作应当将右侧所有相邻的键值对**重新**插入散列表中。

```java
public void delete(Key key) {
    int i = hash(key);
    while (keys[i] != null && !key.equals(keys[i]))
        i = (i + 1) % M;

    // 不存在，直接返回
    if (keys[i] == null)
        return;

    keys[i] = null;
    values[i] = null;

    // 将之后相连的键值对重新插入
    i = (i + 1) % M;
    while (keys[i] != null) {
        Key keyToRedo = keys[i];
        Value valToRedo = values[i];
        keys[i] = null;
        values[i] = null;
        N--;
        putInternal(keyToRedo, valToRedo);
        i = (i + 1) % M;
    }
    N--;
    resize();
}
```

**调整数组大小**

线性探测法的成本取决于连续条目的长度，连续条目也叫**聚簇**。当聚簇很长时，在查找和插入时也需要进行**很多次探测**。**装填因子**的选取很重要。

**α = N/M**，把 α 称为**使用率**。理论证明，当 α 小于 1/2 时探测的预计次数只在 1.5 到 2.5 之间。为了保证散列表的性能，应当调整数组的大小，使得 **α 在 [1/4, 1/2]** 之间。

```java
private void resize() {
    if (N >= M / 2)
        resize(2 * M);
    else if (N <= M / 8)
        resize(M / 2);
}

private void resize(int cap) {
    LinearProbingHashST<Key, Value> t = new LinearProbingHashST<Key, Value>(cap);
    for (int i = 0; i < M; i++)
        if (keys[i] != null)
            t.putInternal(keys[i], values[i]);

    keys = t.keys;
    values = t.values;
    M = t.M;
}
```

> **平方探测法**

平方探测是**消除**线性探测中**一次聚集**问题的散列冲突解决方法。

平方探测就是**冲突函数为二次**的探测方法。典型的情形是：

```
f(i) = i * i
```

**定理**：如果使用**平方探测**，且表的大小是**素数**，那么当表至少有**一半是空**的时候，**总能够**插入一个新的元素。即使表被填充的位置仅仅比**一半多一个**，那么插入都**可能失败**。

平方探测也可能产生**二次聚集**问题。

> **双散列法**

双散列法冲突函数一般的选择是：

```
f(i) = i * hash2(x)
```

这个公式是说将第二个散列函数应用到 x 并在距离 **hash~2~(x)**,  **2 hash~2~(x)....** 等处进行探测。

##### 3. 再散列

如果散列表装的太满，那么再插入新元素的时候可能消耗时间很长，而且可能失败。解决方法是建立另一个大约 2 倍大的表，然后扫描整个原始散列表，**重新计算元素的新散列值**并装入到新的散列表中。这个操作就是**再散列**。

再散列显然开销较大。

**再散列策略**

- 散列表到一半满就再散列。
- 当插入元素失败才再散列（比较极端）。
- **途中策略**：当散列表达到一个**装填因子**时进行再散列（较好）。

##### 4. 高级散列

介绍几个高级一定的散列表。

###### (1) 完美散列

**完美散列的定义**：在关键字集不再变化的情况下，运用某种散列技术，将所有的关键字存入散列表中，可以在最坏运行时间为 O(1) 的情况下完成对散列表的查找工作，这种散列方法就是**完美散列**。

我们期望最坏的情况下，查找的时间函数也是 O(1) — **完美散列**。

使用**二级散列表**可以实现完美散列。每个二级散列表将用一个**不同的散列函数**进行构造，直到没有冲突为止。

###### (2) 布谷鸟散列

CuckooHash（布谷鸟散列）是为了解决**哈希冲突问题**而提出，利用较少的计算换取较大的**空间**。

假设有 N 个项，分别维护**两个**超过半空的表，且有**两个独立的散列函数**，可以把每个项分配到每个表的一个位置。布谷鸟散列保持不变的是一个项总是会被存储在这两个位置之一。

**算法描述**

使用 hashA、hashB 计算对应的 key 位置：

1、两个位置均为空，则任选一个插入；
2、两个位置中一个为空，则插入到空的那个位置
3、两个位置均不为空，则踢出一个位置后插入，被踢出的对调用该算法，再执行该算法找其另一个位置，循环直到插入成功。
4、如果被踢出的次数达到一定的阈值，则认为 hash 表已满，并进行再哈希 rehash。

布谷鸟散列通常被实现成一张巨大的表，且带有**两个（或多个）可以探测整表的散列函数**。

具体实现参考：https://www.jianshu.com/p/68220564f341

###### (6) 跳房子散列

线性探测法是在散列位置的相邻点开始探测，这会引起很多问题，于是各种优化版本例如平方探测、双散列等被提出来改进其中的聚集问题。但是探测相邻位置和第二次散列相比，显然探测相邻位置更有优势，所以线性探测仍然是实用的，甚至是最佳选择。

跳房子散列的思路：**用事先确定的，对计算机底层体系结构而言最优的一个常数**，给探测序列的最大长度加个上界。这样做可以给出常数级的最坏查询时间，并且与**布谷鸟散列**一样，查询可以并行化，以同时检查可用位置 的有限集。

 **要点：**

 a）依然是线性探测。

 b）探测长度 *i* 有个上限。

 c）上限是提前定好的，跟计算机底层体系结构有关系。

 但是布谷鸟散列和跳房子散列还处于实验室状态，能否在实际中代替线性探测法或者平方探测法，还有待验证。



#### **参考资料**

- https://www.jianshu.com/p/68220564f341




