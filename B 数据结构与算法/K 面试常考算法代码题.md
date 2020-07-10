[TOC]

### 面试常考算法题

#### 写一个LRU算法

##### 1. 题目

设计并实现最近最少经⽤（LRU）缓存的数据结构。它应该⽀持以下操作： get 和 put。

- get(key) - 如果键存在于缓存中，则获取键的值（总是正数），否则返回 -1。
- put(key, value) - 如果键不存在，请设置或插⼊值。当缓存达到其最大**容量**时，它应该在插⼊新项⽬之前，
    使最近最少使⽤的项目无效。  

**进阶**：你是否可以在 O(1) 时间复杂度内执⾏两项操作？  

示例：

```java
LRUCache cache = new LRUCache(2);	// 设置最大容量为2
cache.put(1, 1);
cache.put(2, 2);
cache.get(1); // 返回1
cache.put(3, 3); // 去除key2
cache.get(2); // 返回-1(未找到key2)
cache.get(3); // 返回 3
cache.put(4, 4); // 去除key1
cache.get(1); // 返回-1(未找到key1)
cache.get(3); // 返回3
cache.get(4); // 返回4
```

##### 2. 题解

###### (1) 基于LinkedHashMap

别用这个，字节 HR 说用了直接 0 分。

还是贴一个代码，这里主要是要覆写 removeEldestEntry 方法才能设置**缓存容量**。

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    // 设置缓存最大值
    private static final int MAX_ENTRIES = 3;

    // 覆写这个方法使其在数量超出时返回true、
    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        // 如果当前的size大于最大缓存值就返回true
        return size() > MAX_ENTRIES;
    }

    LRUCache() {
        // 同时开启accessOrder为true
        super(MAX_ENTRIES, 0.75f, true);
    }
}
```

###### (2) 双向链表+哈希表

采⽤这两种数据结构的组合，get 操作就可以在 O(1) 时间复杂度内完成了。由于 put 操作我们要删除的节点⼀般是尾部节点，所以我们可以⽤⼀个变量 tail 时刻记录尾部节点的位置，这样 put 操作也可以在 O(1) 时间内完成了。  

这里实现就是需要先来一个结点类。

```java
private static class LRUNode {
    String key;
    Object value;
    LRUNode next;
    LRUNode pre;

    public LRUNode(String key, Object value) {
        this.key = key;
        this.value = value;
    }
}
```

注意：不管是 put 方法还是 get 方法，**都需要将访问的结点重新放到链表头部**，所以会抽出一个公共的方法。

```java
public class LRUCache {

	Map<String, LRUNode> map = new HashMap<>();
	LRUNode head;
	LRUNode tail;
	// 缓存最⼤容量大于1
	int capacity;

	public LRUCache(int capacity) {
		this.capacity = capacity;
	}

	/**
	 * 插入元素
	 * @param key 键
	 * @param value 值
	 */
	public void put(String key, Object value) {
		// 说明缓存中没有任何元素
		if (head == null) {
			head = new LRUNode(key, value);
			tail = head;
			map.put(key, head);
		}
		// 说明缓存中已经存在这个元素了
		LRUNode node = map.get(key);
		if (node != null) {
			// 更新值
			node.value = value;
			// 把这个结点从链表删除并且插⼊到头结点
			removeAndInsert(node);
		} else {
			// 说明当前缓存不存在这个值
			LRUNode newHead = new LRUNode(key, value);
			// 此处溢出则需要删除最近最近未使用的节点
			if (map.size() >= capacity) {
				map.remove(tail.key);
				// 删除尾结点
				tail = tail.pre;
				tail.next = null;
			}
			map.put(key, newHead);
			// 将新的结点插入链表头部
			newHead.next = head;
			head.pre = newHead;
			head = newHead;
		}
	}

	/**
	 * 从缓存中取值
	 */
	public Object get(String key) {
		LRUNode node = map.get(key);
		// 如果有值则需要将这个结点放到链表头部
		if (node != null) {
			// 把这个节点删除并插⼊到头结点
			removeAndInsert(node);
			return node.value;
		}
		return null;
	}

	private void removeAndInsert(LRUNode node) {
		// 特殊情况先判断，例如该节点是头结点或是尾部节点
		if (node == head) {
			return;
		} else if (node == tail) {
			tail = node.pre;
			tail.next = null;
		} else {
			node.pre.next = node.next;
			node.next.pre = node.pre;
		}
		// 插⼊到头结点
		node.next = head;
		node.pre = null;
		head.pre = node;
		head = node;
	}
}
```





