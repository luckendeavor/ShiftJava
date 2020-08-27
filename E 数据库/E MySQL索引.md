[TOC]

### MySQL索引

#### 索引概述

索引是帮助 MySQL 高效获取数据的**排好序**的**==数据结构==**。可以使用不同的数据结构实现索引，比如用 **B+Tree，Hash** 数据结构等实现索引。索引是在存储引擎层实现的，而不是在服务器层实现的，所以不同存储引擎具有不同的索引类型和实现。

索引能够轻易将查询性能提升几个数量级。

1. 对于**非常小的表**、大部分情况下简单的**全表扫描**比建立索引更高效。
2. 对于**中到大型的表**，索引就非常有**效**。
3. 但是对于特大型的表，建立和维护索引的代价将会随之增长。这种情况下，需要用到一种技术可以直接区分出需要查询的一组数据，而不是一条记录一条记录地匹配，例如可以使用分区技术。

##### 1. 索引优缺点

###### (1) 优点

索引可以大大减少服务器需要扫描的**数据行数**，可以大大加快数据的**检索速度**（大大减少的检索的数据量）, 这也是创建索引的最主要的原因。索引帮助服务器**避免进行排序和分组**，以及**避免创建临时表**（B+Tree 索引是**有序**的，可以**用于 ORDER BY 和 GROUP BY 操作**。临时表主要是在排序和分组过程中创建，因为不需要排序和分组，也就不需要创建临时表）。索引将**随机 I/O** 变为**顺序 I/O**（B+Tree 索引是**有序**的，会将相邻的数据都**存储在一起**）。

此外，还可以通过创建唯一性索引，可以保证数据库表中每一行数据的**唯一性**。可以加速表和表之间的**连接**，特别是在实现数据的参考完整性方面特别有意义。

###### (2) 缺点

索引这么多优点，**为什么不对表中的每一个列创建一个索引**呢？

创建索引和维护索引要**耗费时间**，这种时间随着数据量的增加而增加。索引需要占用物理空间，除了数据表占用数据空间之外，每一个索引还要占一定的物理空间，如果建立聚簇索引，那么需要的空间就会更大。当对表中的数据进行增加、删除和修改的时候，索引也需要维护，降低数据维护的速度。

##### 2. MySQL索引分类

###### (1) B+Tree索引

是大多数 MySQL 存储引擎的**默认索引**类型。因为不再需要进行全表扫描，只需要对**树进行搜索**即可，所以查找速度快很多。除了用于查找，还可以用于**排序和分组**（数据在叶子结点的存储方式决定）。可以指定**多个列**作为索引列，多个索引列**共同**组成键。适用于**全键值、键值范围和键前缀**查找，其中键前缀查找只适用于==**最左前缀**==查找。如果**不是按照索引列的顺序进行查找，则无法使用索引。**

InnoDB 的 B+Tree 索引分为**主索引和辅助索引**。主索引的**叶子节点** data 域记录着**完整**的**数据**记录，这种索引方式被称为==**聚簇索引**==。数据行是存储在同一个地方的，所以**一个表**只能有**一个**聚簇索引。

<img src="assets/1563519537753.png" alt="1563519537753" style="zoom:77%;" />

**辅助索引**的**叶子节点**的 data 域记录着**主键的值**，因此在使用**辅助索引**进行查找时，需要先查找到**主键值**，然后再到主索引中进行查找。

<img src="assets/1563519549097.png" alt="1563519549097" style="zoom:80%;" />

###### (2) 哈希索引

**哈希索引**能以 **O(1)** 时间进行查找，但是**失去了有序性**：

- **无法**用于**排序与分组**；
- 只支持**精确查找**，无法用于**部分查找和范围查找**。

**InnoDB 存储引擎**有一个特殊的功能叫“**自适应哈希索引**”，当某个索引值被使用的**非常频繁**时，会在 **B+Tree 索引之上**再创建一个**哈希索引**，这样就让 B+Tree 索引具有哈希索引的一些优点，比如**快速**的哈希查找。

哈希索引底层基于哈希表实现，就是对需要**查询的字段做一次哈希**，然后进行查找。查找速度非常快，性能也很高。

###### (3) 全文索引

MyISAM 存储引擎支持**全文索引**，用于查找文本中的**关键词**，而不是直接比较是否相等。但是查找条件使用 ==**MATCH AGAINST**==，而不是普通的 WHERE。全文索引使用**倒排索引**实现，它记录着关键词到其所在**文档的映射**。InnoDB 存储引擎在 MySQL 5.6.4 版本中也开始支持全文索引。

###### (4) 空间数据索引

MyISAM 存储引擎支持空间数据索引（**R-Tree**），可以用于**地理数据**存储。空间数据索引会从所有维度来索引数据，可以有效地使用任意维度来进行组合查询。必须使用 GIS 相关的函数来维护数据。

##### 3. 索引基本使用

###### (1) 创建索引

在执行 **CREATE TABLE** 语句时可以创建索引，也可以单独用 **CREATE INDEX 或 ALTER TABLE** 来为表增加索引。

**ALTER TABLE**：用来创建**普通索引、UNIQUE 索引或 PRIMARY KEY 索引**。例如：

```mysql
 ALTER TABLE table_name ADD INDEX index_name (column_list)
 ALTER TABLE table_name ADD UNIQUE (column_list)
 ALTER TABLE table_name ADD PRIMARY KEY (column_list)
```

其中 table_name 是要**增加索引的表名**，column_list 指出对**哪些列进行索引**，**多列时各列之间用逗号分隔**。索引名 index_name 可选，缺省时，MySQL 将根据第一个索引列赋一个名称。另外，ALTER TABLE 允许在单个语句中更改多个表，因此可以在同时创建多个索引。

**CREATE INDEX**：可对表**增加普通索引或 UNIQUE 索引**。例如：

```mysql
 CREATE INDEX index_name ON table_name (column_list)
 CREATE UNIQUE INDEX index_name ON table_name (column_list)
```

table_name、index_name 和 column_list 具有与 ALTER TABLE 语句中相同的含义，索引名不可选。另外，不能用 CREATE INDEX 语句创建 PRIMARY KEY 索引。

###### (2) 唯一索引

在创建索引时，可以规定索引**能否包含重复值**。如果不包含，则索引应该创建为 **PRIMARY KEY 或 UNIQUE 索引**。对于单列唯一性索引，这保证单列不包含重复的值。对于多列惟一性索引，保证多个值的组合不重复。PRIMARY KEY 索引和 UNIQUE 索引非常类似。

###### (3) 删除索引

可利用 **ALTER TABLE** 或 **DROP INDEX** 语句来删除索引。类似于 CREATE INDEX 语句，DROP INDEX 可以在 ALTER TABLE 内部作为一条语句处理，语法如下。

```mysql
 DROP INDEX index_name ON talbe_name
 ALTER TABLE table_name DROP INDEX index_name
 ALTER TABLE table_name DROP PRIMARY KEY
```

其中，前两条语句是**等价**的，删除掉 table_name 中的索引 index_name。第 3 条语句只在删除 PRIMARY KEY 索引时使用，因为一个表只可能有一个 PRIMARY KEY 索引，因此不需要指定索引名。如果没有创建 PRIMARY KEY 索引，但表具有一个或多个 UNIQUE 索引，则 MySQL 将删除第一个 UNIQUE 索引。如果从表中删除了某列，则索引会受到影响。对于多列组合的索引，如果删除其中的某列，则该列也会从索引中删除。如果删除组成索引的所有列，则整个索引将被删除。



#### B-Tree

B- 树的结点会**同时存数据与索引**，通常一个结点大小有限制，如果一个结点存储的数据过多，就会导致能存储的索引变少，树的分叉变少，使得树的高度变高。所以需要用 B+ 树。

特点：叶节点具有相同的深度，叶节点的指针为空，所有**索引元素不重复**，节点中的数据索引从左到右递增排列。

B-Tree 的**非叶子节点可能也会存储数据**，而且叶子节点之间**不含有**指针。

#### B+Tree原理

##### 1. 数据结构

B Tree 指的是 **Balance Tree**，也就是**平衡树**。平衡树是一颗==**查找树**==，并且**所有==叶子节点位于同一层==**。B+ Tree 是基于 B Tree 和**叶子节点顺序访问指针**进行实现，它具有 B Tree 的平衡性，并且通过**顺序访问指针**来提高区间查询的性能。

在 **B+ Tree** 中，一个**节点**中的 **key 从左到右非递减排列**，如果某个指针的左右相邻 key 分别是 key<sub>i</sub> 和 key<sub>i+1</sub>，且不为 null，则该指针指向节点的所有 key 大于等于 key<sub>i</sub> 且小于等于 key<sub>i+1</sub>。

<img src="assets/1563519516758.png" alt="1563519516758" style="zoom:70%;" />

B+ 树的**全部数据**都存在**叶子结点**上，而且**叶子结点还有顺序指针**，这样可以方便进行**范围查询**。而 B 树中数据可能在非叶子结点上，而且不含有指针。

加载一层**结点**就对应了一次**磁盘 IO 操作**。

B+Tree 中非叶子节点不存储 data，**只存储索引(冗余)**，可以放更多的索引。B+ 树存储索引时存在**冗余**，比如上图中 3 号在叶子结点和中间结点**都存在**，但是 B-Tree 中**不存在冗余**。叶子节点之间可以用**指针连接**，提高**区间访问**的性能。这样维护数据结构的话，叶子节点之间的元素是从小到大依次递增的，方便进行**范围查找**。

InnoDB 中一个结点默认的大小 **Innodb_page_size** 为 **16K**。如果存放的是 INT 类型的索引，那一个结点也可以存**上千个值**，可以有上千个分叉。

##### 2. 查询操作

进行**查找**操作时，首先在**根节点**进行**二分查找**，找到一个 key 所在的指针，然后递归地在指针所指向的节点进行查找。直到查找到叶子节点，然后在叶子节点上进行**二分查找**，找出 key 所对应的 data。插入删除操作会破坏平衡树的平衡性，因此在插入删除操作之后，需要对树进行一个分裂、合并、旋转等操作来**维护平衡性**。

##### 3. 与红黑树的比较

**红黑树**等平衡树也可以用来实现**索引**，但是文件系统及数据库系统普遍采用 **B+ Tree 作为索引结构**，主要有以下两个原因：

(1) **更少的查找次数**：平衡树查找操作的时间复杂度和**树高 h 相关**，O(h)=O(log<sub>d</sub>N)，其中 d 为每个节点的出度。红黑树的**出度为 2**，而 B+ Tree 的出度一般都**非常大**，所以红黑树的树**高 h 很明显比 B+ Tree 大非常多**，查找的次数也就更多。B+ 树比红黑树更加**矮平**，树越低，**磁盘 IO 操作**次数就越少。

(2) **B+树可以利用磁盘预读特性**：为了减少磁盘 I/O 操作，磁盘往往不是严格按需读取，而是每次都会**预读**。预读过程中，磁盘进行**顺序读取**，顺序读取不需要进行磁盘寻道，并且只需要很短的旋转时间，速度会非常快。操作系统一般将内存和磁盘分割成**固定大小的块**，每一块称为**一页**，内存与磁盘以**页为单位**交换数据。数据库系统将索引的一个**节点**的大小设置为页的大小，使得一次 I/O 就能完全载入一个节点。并且可以利用**预读特性**，相邻的节点也能够被**预先载入**。



#### 索引与存储引擎实现

##### 1. 概述

索引分为**主键索引与非主键索引**（普通索引、辅助索引）。索引是在**存储引擎层**实现的，而不是在服务器层实现的，所以**不同存储引擎具有不同的索引类型**和实现。

MySQL 索引使用的**数据结构**主要有 **BTree索引** 和 **哈希索引** 。对于哈希索引来说，底层的数据结构就是**哈希表**，因此在绝大多数需求为单条记录查询的时候，可以选择哈希索引，查询性能最快；**其余大部分场景，建议选择 BTree 索引**。

MySQL 的 BTree 索引使用数据结构是 **B+Tree**，但对于主要的**两种存储引擎的实现方式**是不同的。

##### 2. MyISAM索引实现

MyISAM 存储引擎的表在**磁盘**中会存储为三个文件：**==.myd==（数据文件）和 ==.myi==（索引文件）组成，==.frm== 文件存储表结构（所以存储引擎都有）**。

所以 MyISAM **主键索引**的**索引文件和数据文件是分离**的，这就是**非聚集索引**。MyISAM 存储引擎中**非主键索引**（普通索引）与**主键索引组织方式是类似**的，结构都差不多。组织方法如下，B+Tree **索引叶子结点**存储的是**真实数据的地址**：

<img src="assets/image-20200617171720198.png" alt="image-20200617171720198" style="zoom:60%;" />

也就是通过索引叶子节点存储的数据地址去数据文件获取真实的数据。

##### 3. InnoDB索引实现

InnoDB 存储引擎的表在磁盘中会存储为两个文件：**==.ibd== 文件（索引+数据文件）和 ==.frm== 文件（表结构文件）**。

**MyISAM** 的主键索引与非主键索引的结构**类似**，而 **InnoDB** 主键索引与非主键索引（普通索引）的组织结构不同，因此这里分别讨论。

###### (1) 主键索引

对于**主键索引**，InnoDB 的**主键索引**就是**聚集索引**。InnoDB 通过**主键聚集索引**。聚集索引的含义：

- 表数据文件本身就是按 **B+Tree** 组织的一个索引结构文件。
- **聚集索引-叶节点包含了完整的数据记录**。

其组织方式如下。可以看到在**叶子结点上索引是主键**（绿色格子），其下面存储的是**全部完整的数据**。这也是聚集的含义。

<img src="assets/image-20200617171816226.png" alt="image-20200617171816226" style="zoom: 67%;" />

###### (2) 非主键索引

对于**非主键索引（辅助索引）**，InnoDB 组织方式大致如下，**索引叶子**结点存储的是**主键的值**。

<img src="assets/image-20200617171831994.png" alt="image-20200617171831994" style="zoom:67%;" />

非主键索引下面记录的是**主键值**！而不是像主键索引那样存储的是全部数据，得到**主键值之后通过主键进行查找**。

为什么这样做？这是为了保证**一致性和节省存储空间**。一致性是指，比如当插入一条数据的时候，如果**非主键索引上也存储全部元素**，那么就需要保证在不同的地方都**全部插入数据**，这就难以保证一致性；只在主键索引下面存放完整数据，只需要维护一份完整数据，既能保证一致性也能**节省空间**（数据只需要保存一份就行）。

###### (3) 联合索引

**两个或更多个列上的索引**被称作**联合索引**（复合索引）。对于联合索引，MySQL **从左到右**的使用索引中的字段，一个查询可以只使用索引中的**一部分**，但只能是**最左侧部分**。

对于**联合索引**（多个索引字段组合），**存储结构**就是把多个字段同时放到了结点的 Key 中了。联合索引怎么比大小排序？比如有三个索引字段 (a, b, c)，那么此时比较大小是依次比较索引字段，先比较 a 字段，再比较 b 字段。注意：如果索引相同，那么是**挨着排存储在结点**中的，只要保证右边的值大于等于左边的即可。

下面是一个创建联合索引的例子。

```mysql
KEY `idx_film_actor_id` (`film_id`,`actor_id`)
```

例如索引是 key index (a, b, c)，可以支持 [a]、[a, b]、[a, b, c] 等 3 种组合进行索引命中查找，不支持 [b, c] 进行查找。当最左侧字段是**常量引用**时，索引就十分有效。

**联合索引**在 InnoDB 中的组织方式如下，可以看到**多个索引字段存储到了一起**，索引叶子结点中，**绿色的部分是联合索引的全部值**，紫色部分是非索引的其他数值。其实也是**全部数据**都存储到了叶子结点上。

<img src="assets/image-20200617172002929.png" alt="image-20200617172002929" style="zoom:47%;" />

**少用单值索引**，比如有一个表有 20 个字段，有 5 个需要建索引，如果建 5 个单值索引，那么就会有**五个 B+Tree** 结构存放在 InnoDB 引擎的 **.ibd 文件**中。推荐使用**联合索引**，如果把五个字段联合起来**建立联合索引**，那么只会有一棵 B+Tree。

###### (4) 主键规则

> 为什么 InnoDB 表**必须有主键**？

InnoDB 设计的**数据文件**就是**依附于 B+Tree** 的，所有**数据都是依附在主键**上的，所以说**必须有主键**。如果自己建表的时候没有主键，那么会自动找一列可以充当唯一索引的当做主键，如果找不到，那么就会自动生成一个 rowId 当做主键。

> 为什么推荐使用**整型的自增**主键？

**主键**为啥推荐用**整型**？索引之前查找都是**基于比较**的，用整型是非常方便的。**别用 UUID 当主键**，占用空间也大，而且难以比较。为什么推荐**自增**？如果是自增的话，**直接顺序**放就行了，如果不是自增，当前面的结点**已经存满之后**，此时为了维护 B+Tree 的特性，会导致前面的结点进行分裂与平衡的变动，**影响性能**。

主键规则：**不要使用更新频繁的列**作为主键，不适用多列主键（相当于联合索引）。不**要使用 UUID, MD5, HASH** 字符串列作为主键（无法保证数据的**顺序增长**）。主键**建议使用自增 ID 值**。

##### 4. 总结

**MyISAM 的主键索引是非聚集索引，InnoDB 的主键索引是聚集索引。**多个索引字段建议创建联合索引。

- **MyISAM：**B+Tree **叶节点的 data 域**存放的是**数据记录的地址**。在索引检索的时候，首先按照 B+Tree 搜索算法搜索索引，如果指定的 Key 存在，则取出其 data 域的值，然后以 data 域的**值为地址**读取相应的数据记录。MyISAM 中**索引文件和数据文件是分离**的，这被称为“**非聚簇索引**”。
- **InnoDB：**其**数据文件本身就是索引文件**。其表数据文件本身就是按 B+Tree 组织的一个索引结构，树的叶节点 **data 域保存了完整的数据记录**。这个索引的 **key 是数据表的主键**，因此 InnoDB 表数据文件本身就是主索引。这被称为“**聚簇索引**（或聚集索引）”。而其余的索引都作为**辅助索引**，**辅助索引的 data 域存储相应记录主键的值**而不是地址，这也是和 MyISAM 不同的地方。**在根据主索引搜索时，直接找到 key 所在的节点即可取出数据；在根据辅助索引查找时，则需要先取出主键的值，再走一遍主索引。** **因此在设计表的时候，不建议使用过长的字段作为主键，也不建议使用非单调的字段作为主键，这样会造成主索引频繁分裂。** PS：整理自《Java工程师修炼之道》



#### 索引优化基础

先来个表：

```mysql
CREATE TABLE `employees` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `name` varchar(24) NOT NULL DEFAULT '' COMMENT '姓名',
    `age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄',
    `position` varchar(20) NOT NULL DEFAULT '' COMMENT '职位',
    `hire_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间', 
    PRIMARY KEY (`id`),
    # 创建联合索引
    KEY `idx_name_age_position` (`name`,`age`,`position`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 4 DEFAULT CHARSET = utf8 COMMENT = '员工记录表';

INSERT INTO employees(name,age,position,hire_time) VALUES('LiLei',22,'manager',NOW());
INSERT INTO employees(name,age,position,hire_time) VALUES('HanMeimei',23,'dev',NOW());
INSERT INTO employees(name,age,position,hire_time) VALUES('Lucy',23,'dev',NOW());
```

注意这里创建了**联合索引**：idx_name_age_position，联合了三个字段 **name，age 和 position**。

##### 1. 最左前缀法则

如果对**多列**字段建立了索引，要遵守最左前缀法则。指的是查询从索引的**最左前列开始并且不跳过索引中的列**。也就是要命中索引的话，**必须包含左边的字段，一个或者多个都可**。结合上面联合索引的组织图，可以看到联合索引是**从左往右挨着比较**的。如果差了左边的字段，那么其实也就是挨着比较，也就相当于走**全表**扫描了。

对于 **BLOB、TEXT 和 VARCHAR** 类型的列，必须使用==**前缀索引**==，只索引**开始的部分字符**。对于前缀长度的选取需要根据索引选择性来确定。

比如索引字段为 A（name），B（age），C（position）。则：

- 查询条件为 **A、AB、ABC** 时，**会走索引**。
- 查询条件为 **BC**，**不会走索引**。
- 查询条件为 **AC**，只会走字段 **A 的索引**。

案例 1：这里查询条件只含 B 和 C，**不会**走索引。

```mysql
EXPLAIN SELECT * FROM employees WHERE age = 22 AND position ='manager'; 
```

案例 2：这里查询条件只含 C，**不会**走索引。

```mysql
EXPLAIN SELECT * FROM employees WHERE position = 'manager';
```

案例 3：这里查询条件只含 A，就是最左前缀，**会**走索引。

```mysql
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei';  
```

案例 4：这里查询条件含 A 和 B（写的顺序倒过来也可），就是最左前缀，**会**走索引。

```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age = 22;
EXPLAIN SELECT * FROM employees WHERE age = 22 AND name= 'LiLei';
```

##### 2. 索引失效

###### (1) 失效实例

有一些情况会导致**索引失效**从而进行**全表扫描**。索引失效需要联系到 InnoDB 中的**联合索引组织方式**来思考。

> (1) 在索引列上做**任何操作，比如计算、函数、自动或手动类型转换等**

在进行查询时，索引列**不能是表达式的一部分**，也不要对其**施加函数**或者成为**函数的参数**，否则**无法**使用索引。这会造成无法命中索引。

有函数直接不走索引：

```mysql
EXPLAIN SELECT * FROM employees WHERE left(name, 3) = 'LiLei';
EXPLAIN select * from employees where date(hire_time) ='2018-09-30';
```

可以根据查询的需求**自己转换为范围查询**走索引。

```mysql
EXPLAIN select * from employees where hire_time >='2018-09-30 00:00:00' and
hire_time <='2018-09-30 23:59:59';
```

例如下面的查询不能使用 actor_id 列的索引：

```sql
SELECT actor_id FROM sakila.actor WHERE actor_id + 1 = 5;  
```

> (2) **范围查找之后**的索引失效

如果 MySQL 估计使用**全表扫秒比使用索引快**，则不适用索引。例如列 key 均匀分布在 1 和 100 之间，下面的查询使用索引就可能不走索引：select * from table_name where key>1 and key<90;。

下面的句子中第一句走三个索引，但是第二个 SQL 只走了两个索引，也就是 age 是范围查找，这里 age 后面的索引是失效了，也就是只会走 name 和 age 两个索引，position 失效了。

```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age = 22 AND position ='manager';
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age > 22 AND position ='manager';
```

> (3) 使用**不等于**（!= 或者<>）导致索引失效

因为索引一般是**比较相不相等**，比较不相等其实也就是**全表扫描**了。

```mysql
EXPLAIN SELECT * FROM employees WHERE name != 'LiLei';
```

> (4) **is null, is not null** 也无法使用索引  

与使用不等于失效的情况类似，所以建议数据表的字段设置为**NOT NULL**，没有值也搞一个**默认值**，这样**保证索引命中**。否则可能导致引擎放弃使用索引而进行**全表**扫描。

```mysql
EXPLAIN SELECT * FROM employees WHERE name is null;
```

> (5) **like 以通配符开头**（'$abc...'）导致索引失效

根据**最左前缀法则**，**% 在前面是没法走索引的**，直接就是全表扫描了。

```mysql
EXPLAIN SELECT * FROM employees WHERE name like '%Lei';
```

**% 在后面是可以走索引的。**这里是知道前缀的情况，**比对前缀**就行了。 

```mysql
EXPLAIN SELECT * FROM employees WHERE name like 'Lei%';
```

问题：解决 like '%字符串%' 索引不被使用的方法？  

a）使用**覆盖索引**，查询字段**必须是建立覆盖索引字段**。这里

```mysql
EXPLAIN SELECT name, age, position FROM employees WHERE name like '%Lei%';
```

b）如果不能使用覆盖索引则可能需要借助**搜索引擎**，如 ES。

> (6) **字符串不加单引号**索引失效 

如果列为**字符串**，则 where 条件中必须**将字符常量值加引号**，否则即使该列上存在索引，也不会被使用。下面第一句走索引，第二句不走索引。

```mysql
EXPLAIN SELECT * FROM employees WHERE name = '1000';
EXPLAIN SELECT * FROM employees WHERE name = 1000;
```

> (7) 使用**or**可能引起索引失效

如果**条件中有 or**，即使其中有条件带索引也不会使用。例如：select * from table_name where key1='a' or key2='b'; 如果在 key1 上有索引而在 key2 上没有索引，则该查询也不会走索引。

```mysql
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' or name = 'HanMeimei';
```

> (8) 使用**联合索引失效**

**联合索引**，如果索引列**不是联合索引的第一部分**，则不使用索引（即不符合最左前缀）。例如联合索引为(key1, key2)，则查询 select * from table_name where key2='b'; 将不会使用索引。

##### 3. Explain关键字

###### (1) 概述

使用 Explain 关键字可以**模拟优化器执行 SQL 语句**，分析你的查询语句或是结构的性能瓶颈**在 SELECT 语句之前增加  explain 关键字**，MySQL 会在查询上设置一个**标记**，执行查询会**返回执行计划的信息**，而不是执行这条 SQL。它可以分析查询与或是表结构的性能瓶颈。注意：如果  from 中包含子查询，仍会执行该子查询，将结果放入临时表中。

###### (2) explain属性

Explain 分析比较重要的属性有：

- **select_type**：select 查询的类型，主要是区别普通查询和联合查询、子查询之类的复杂查询。
- **type**：**联合查询所使用的类型**，type 显示的是**访问类型**，是**较为重要**的一个指标。结果值从好到坏依次是：**system > const > eq_ref > ref >fulltext > ref_or_null > index_merge > unique_subquery >index_subquery > range > index > ALL**。一般来说，得保证查询**至少达到 range 级别，最好能达到 ref**。type = const 表示通过**索引一次**就找到了；type = all 表示为**全表扫描**。

- **key**：显示 MySQL 实际决定使用的**键**。如果**没有索引被选择，键是 NULL**。key=primary 表示使用了主键；key=null 表示没用到索引。

- **possible_keys**：指出 MySQL 能使用哪个索引在该表中找到行。如果是空的，没有相关的索引。这时要提高性能，可通过检验 WHERE 子句，看是否引用某些字段，或者检查字段不是适合索引。

- **key_len**：显示 MySQL 决定使用的键长度。如果键是 NULL，长度就是 NULL。

- **ref**：显示哪个字段或常数与 key 一起被使用。
- **rows**：这个数表示 MySQL 要**遍历多少数据**才能找到，在 InnoDB 上是不准确的。
- **Extra**：如果是 **Only index**，这意味着信息只用索引树中的信息检索出的，这比扫描整个表要快。如果是 **where used**，就是使用上了 **where 限制**。如果是 **impossible where** 表示用不着 where，一般就是没查出来啥。

##### 4. Trace工具

对于上面的两种 name>'a' 和 name>'zzz' **范围查询**的执行结果，MySQL **最终**是否会选择走索引无法轻易判断的，可以使用 **trace 工具**来看看 MySQL 最终如何选择索引。开启 trace 工具会**影响 MySQL 性能**，所以只能**临时分析** SQL 使用，**用完之后立即关闭**。

开启 trace 工具。

```mysql
set session optimizer_trace="enabled=on",end_markers_in_json=on;    # 开启trace
```

执行一下面的语句。

```mysql
select * from employees where name > 'a' order by position;
```

**分析结果**存放到了下面的表中。

```mysql
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```

得到 trace 字段：

```json
{
"steps": [
{
"join_preparation": { ‐‐第一阶段：SQL准备阶段
                 "select#": 1,
                 "steps": [
                 {
                 "expanded_query": "/* select#1 */ select `employees`.`id` AS `id`,`employees`.`name` AS `name`,`employees`.`age` AS `age`,`employees`.`position` AS `position`,`employees`.`hire_time` AS `hire_time` from`employees` where (`employees`.`name` > 'a') order by `employees`.`position`"
}
]/* steps */
}/* join_preparation */},
{
"join_optimization": { ‐‐第二阶段：SQL优化阶段
"select#": 1,
"steps": [
{
"condition_processing": { ‐‐条件处理
"condition": "WHERE",
"original_condition": "(`employees`.`name` > 'a')",
"steps": [
{
"transformation": "equality_propagation",
"resulting_condition": "(`employees`.`name` > 'a')"
},
{
"transformation": "constant_propagation",
"resulting_condition": "(`employees`.`name` > 'a')"
},
{
"transformation": "trivial_condition_removal",
"resulting_condition": "(`employees`.`name` > 'a')"
}
]/* steps */
}/* condition_processing */
},
{
"substitute_generated_columns": {
}/* substitute_generated_columns */
},
{
"table_dependencies": [ ‐‐表依赖详情
{
"table": "`employees`",
"row_may_be_null": false,
"map_bit": 0,
"depends_on_map_bits": [
]/* depends_on_map_bits */
}
]/* table_dependencies */
},
{
"ref_optimizer_key_uses": [
]/* ref_optimizer_key_uses */
},
{
"rows_estimation": [ ‐‐预估表的访问成本
{
"table": "`employees`",
"range_analysis": {
"table_scan": { ‐‐全表扫描情况
"rows": 10123, ‐‐扫描行数
"cost": 2054.7 ‐‐查询成本
}/* table_scan */,70 "potential_range_indexes": [ ‐‐查询可能使用的索引
{
"index": "PRIMARY", ‐‐主键索引
"usable": false,
"cause": "not_applicable"
},
{
"index": "idx_name_age_position", ‐‐辅助索引
"usable": true,
"key_parts": [
"name",
"age",
"position",
"id"
]/* key_parts */
}
]/* potential_range_indexes */,
"setup_range_conditions": [
]/* setup_range_conditions */,
"group_index_range": {
"chosen": false,
"cause": "not_group_by_or_distinct"
}/* group_index_range */,
"analyzing_range_alternatives": { ‐‐分析各个索引使用成本
"range_scan_alternatives": [
{
"index": "idx_name_age_position",
"ranges": [
"a < name" ‐‐索引使用范围
]/* ranges */,
"index_dives_for_eq_ranges": true,
"rowid_ordered": false, ‐‐使用该索引获取的记录是否按照主键排序
"using_mrr": false,
"index_only": false, ‐‐是否使用覆盖索引
"rows": 5061, ‐‐索引扫描行数
"cost": 6074.2, ‐‐索引使用成本
"chosen": false, ‐‐是否选择该索引
"cause": "cost"
}
]/* range_scan_alternatives */,
"analyzing_roworder_intersect": {
"usable": false,
"cause": "too_few_roworder_scans"
}/* analyzing_roworder_intersect */
}/* analyzing_range_alternatives */
}/* range_analysis */
}
]/* rows_estimation */
},
{
"considered_execution_plans": [
{ "plan_prefix": [
]/* plan_prefix */,
"table": "`employees`",
"best_access_path": { ‐‐最优访问路径
"considered_access_paths": [ ‐‐最终选择的访问路径
{
"rows_to_scan": 10123,
"access_type": "scan", ‐‐访问类型：为scan，全表扫描
"resulting_rows": 10123,
"cost": 2052.6,
"chosen": true, ‐‐确定选择
"use_tmp_table": true
}
]/* considered_access_paths */
}/* best_access_path */,
"condition_filtering_pct": 100,
"rows_for_plan": 10123,
"cost_for_plan": 2052.6,
"sort_cost": 10123,
"new_cost_for_plan": 12176,
"chosen": true
}
]/* considered_execution_plans */
},
{
"attaching_conditions_to_tables": {
"original_condition": "(`employees`.`name` > 'a')",
"attached_conditions_computation": [
]/* attached_conditions_computation */,
"attached_conditions_summary": [
{
"table": "`employees`",
"attached": "(`employees`.`name` > 'a')"
}
]/* attached_conditions_summary */
}/* attaching_conditions_to_tables */
},
{
"clause_processing": {
"clause": "ORDER BY",
"original_clause": "`employees`.`position`",
"items": [
{
"item": "`employees`.`position`"
}
]/* items */,
"resulting_clause_is_simple": true,
"resulting_clause": "`employees`.`position`"
}/* clause_processing */
},
{
"reconsidering_access_paths_for_index_ordering": {
"clause": "ORDER BY", "steps": [
]/* steps */,
"index_order_summary": {
"table": "`employees`",
"index_provides_order": false,
"order_direction": "undefined",
"index": "unknown",
"plan_changed": false
}/* index_order_summary */
}/* reconsidering_access_paths_for_index_ordering */
},
{
"refine_plan": [
{
"table": "`employees`"
}
]/* refine_plan */
}
]/* steps */
}/* join_optimization */
},
{
"join_execution": { ‐‐第三阶段：SQL执行阶段
"select#": 1,
"steps": [
]/* steps */
}/* join_execution */
}
]/* steps */
}
```

上述的 SQL 语句中**全表扫描的成本低于索引扫描，所以最终选择全表扫描。**

trace 工具会**分析并预估**走索引和全表扫描的**成本**，进行对比，然后进行选择执行。



#### 索引优化建议

##### 1. 多列索引优化

在需要使用**多个列**作为条件进行查询时，使用**多列索引比使用多个单列索引性能更好**。例如下面的语句中，最好把 actor_id 和 film_id 设置为多列索引。

```sql
SELECT film_id, actor_ id FROM sakila.film_actor WHERE actor_id = 1 AND film_id = 1;
```

##### 2. 覆盖索引优化

索引包含**所有需要查询**的字段的值。具有以下优点：

- 索引通常远小于数据行的大小，只读取索引能大大减少数据访问量。
- 一些存储引擎（例如 MyISAM）在内存中只缓存索引，而数据依赖于操作系统来缓存。因此，只访问索引可以不使用系统调用（通常比较费时）。
- 对于 InnoDB 引擎，若**辅助索引**能够覆盖查询，则**无需访问主索引**。

减少 **select *** 语句的使用。

```mysql
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age = 23 AND
position ='manager';
```

优化一下使用覆盖索引：

```mysql
EXPLAIN SELECT name, age FROM employees WHERE name= 'LiLei' AND age = 23
AND position ='manager';  
```

##### 3. 索引列顺序优化

让**==选择性最强==的索引列放在前面**。索引的选择性是指：**不重复的索引值和记录总数的比值**。最大值**为 1**，此时**每个记录都有唯一的索引与其对应**。选择性越高，查询效率也越高。

例如下面显示的结果中 customer_id 的选择性比 staff_id 更高，因此最好把 customer_id 列放在多列索引的前面。

```sql
SELECT COUNT(DISTINCT staff_id)/COUNT(*) AS staff_id_selectivity,
COUNT(DISTINCT customer_id)/COUNT(*) AS customer_id_selectivity,
COUNT(*)
FROM payment;
```

```html
staff_id_selectivity: 0.0001
customer_id_selectivity: 0.0373
COUNT(*): 16049
```

##### 4. 范围查询优化

MySQL 内部优化器会**根据检索比例、表大小等多个因素整体评估是否使用索引**。如果使用范围查询，可能会因为**范围问题**使得优化器觉得全表扫描的效率还高，进而使得索引失效。具体的情况还得靠优化器根据实际场景自己分析。

**例子1**：给年龄添加**单值索引**。

```mysql
ALTER TABLE `employees` ADD INDEX `idx_age` (`age`) USING BTREE ;
```

下面的**范围查询**，实际上**没有走索引**。

```mysql
explain select * from employees where age >=1 and age <=2000;
```

这个例子，可能是由于单次数据量**查询过大**导致优化器最终选择不走索引。

优化方法：**可以将大的范围拆分成多个小范围**。比如改成下面的两个范围查询，可能就会走索引。

```mysql
explain select * from employees where age >=1 and age <=1000;
explain select * from employees where age >=1001 and age <=2000;
```

**例子2**：看下面的 SQL 语句，这里是没走索引的。

```mysql
EXPLAIN select * from employees where name > 'a';
```

如果用 name 索引需要遍历 name 字段**联合索引树**，然后还需要根据**遍历出来的主键值**去主键索引树里再去查出最终数据，相当于**遍历了两棵索引树**（一颗主键索引树、一颗联合索引树），成本比全表扫描还高，所以就不走索引了。

可以用**覆盖索引**优化，这样**只需要遍历 name 字段**的**联合索引树**就能拿到所有结果，如下：  

```mysql
EXPLAIN select name, age, position from employees where name > 'a';
```

再看下面的 SQL。这一条其实跟上面的类似，只是 name 的范围不同，这里 MySQL 根据数据量发现，这里数据量较少，于是就**走了索引**。这里就是引擎执行的**过程中**根据数据情况会进行优化。

```mysql
EXPLAIN select * from employees where name > 'zzz' ;
```

##### 5. 排序优化

###### (1) 概述

MySQL 支持两种方式的**排序 filesort 和 index**，Using index 是指 MySQL **扫描索引**本身完成排序。index 效率高，**filesort 相当于是加载数据**之后再进行**重新排序**，效率低。

Order by 满足两种情况会使用 **Using index**。

- order by 语句使用**索引最左前列**。
- 使用 where 子句与 order by 子句**条件列组合满足索引最左前列**。

所以尽量在**索引列上完成排序**，遵循索引建立（索引创建的顺序）时的**最左前缀法则**。如果 order by 的条件不在索引列上，就会产生 **Using filesort**。所以能用覆盖索引尽量用覆盖索引。

group by 与 order by 很类似，其实质是**先排序后分组**，遵照索引创建顺序的**最左前缀法则**。对于 group by 的优化如果不需要排序的可以**加上 order by null 禁止排序**。注意，**where 高于 having**，能写在 where 中的限定条件就不要去 having 限定了。  

###### (2) Using filesort

filesort **文件排序**方式，效率较低。filesort 排序方式分为：

- **单路排序**：是一次性取出满足条件行的**所有字段**，然后在 **sort buffer** 中进行排序；用 trace 工具可以看到sort_mode 信息里显示 <sort_key, additional_fields> 或者 <sort_key, packed_additional_fields>。  
- **双路排序**（又叫**回表**排序模式）：是首先根据相应的条件**取出相应的排序字段和可以直接定位行数据的行 ID**，然后在 **sort buffer** 中进行排序，排序完后需要**再次回表查询得到其它需要的字段**；用 trace 工具可以看到 sort_mode 信息里显示 <sort_key, rowid>。  

MySQL 通过比较系统变量 **max_length_for_sort_data**(默认 1024 字节) 的大小和需要查询的字段总大小来判断使用哪种排序模式。

- 如果 max_length_for_sort_data 比查询字段的总长度**大**，那么使用 **单路排序**模式；
- 如果 max_length_for_sort_data 比查询字段的总长度**小**，那么使用 **双路排序**模式。其实就是如果需要回表的数据行太多，那么开销就会很大。

如果**使用了 filesort**，那么 **trace** 分析结果中会有 **filesort_summary** 的信息。

```json
"filesort_summary": { 
    "rows": 10000, // 预计扫描行数
    "examined_rows": 10000, // 参数排序的行
    "number_of_tmp_files": 3, 
    "sort_buffer_size": 262056, // 排序缓存的大小
    "sort_mode": "<sort_key, packed_additional_fields>" // 排序方式，这里用的单路排序
}
```

number_of_tmp_files 表示使用**临时文件**的个数，这个值如果为 0 代表全部使用的 **sort_buffer 内存排序**，否则使用的**磁盘文件排序**。如果内存不够用，可能使用磁盘排序。例子如下：

```mysql
select * from employees where name = 'Tom' order by position;
```

我们先看**单路排序**的详细过程：

- 从索引 name 找到第一个满足 name = ‘Tom’ 条件的主键 id。
- 根据主键 id 取出**整行**，取出所有字段的值，存入 **sort_buffer** 中。
- 从索引 name 找到下一个满足 name = ‘Tom’ 条件的主键 id。
- 重复上述步骤直到不满足 name = ‘Tom’。
- 对 sort_buffer 中的数据按照字段 position 进行排序。
- 返回结果给客户端。

再看下**双路排序**的详细过程：

- 从索引 name 找到第一个满足 name = ‘Tom’ 的主键 **id。**
- 根据主键 id 取出整行，把排序字段 position 和主键 id 这两个字段放到 sort buffer 中。
- 从索引 name 取下一个满足 name = ‘Tom’ 记录的主键 id。
- 重复上述步骤直到不满足 name = ‘Tom’。
- 对 **sort_buffer 中的字段 position 和主键 id 按照字段 position 进行排序**。
- 遍历排序好的 id 和字段 position，按照 id 的值**回到原表**中取出 所有字段的值返回给客户端  。

其实对比两个排序模式，**单路排序**会把所有需要查询的字段**都放到 sort buffer** 中，而双路排序只会把**主键和需要排序的字段**放到 sort buffer 中进行排序，然后再通过主键回到原表查询需要的字段。

如果 MySQL **排序内存比较小**并且没有条件继续增加了，可以适当把 max_length_for_sort_data 配置小点，让优化器选择使用双路排序算法，可以在 sort_buffer 中一次排序更多的行，只是需要**再根据主键回到原表取数据**。

如果 MySQL **排序内存**有条件可以配置比较大，可以适当增大 max_length_for_sort_data 的值，让优化器优先选择全字段排序(单路排序)，把需要的字段放到 sort_buffer 中，这样排序后就会直接从内存里返回查询结果了。

所以，MySQL 通过 **max_length_for_sort_data** 这个参数来控制排序，在不同场景使用不同的排序模式，从而提升排序效率。

注意，如果**全部使用 sort_buffer 内存排序一般情况下效率会高于磁盘文件排序**，但不能因为这个就随便增大 sort_buffer (默认 1M)，MySQL 很多参数设置都是做过优化的，不要轻易调整。  

##### 6. 分页查询优化

业务系统经常需要分页功能：

```mysql
mysql> select * from employees limit 10000, 10;
```

表示从表 employees 中取出**从 10001 行开始的 10 行记录**。看似只查询了 10 条记录，实际这条 SQL 是**先读取 10010 条记录**，然后**抛弃**前 10000 条记录，然后读到后面 10 条想要的数据。因此要查询一张大表比较靠后的数据，执行效率是非常低的。常见的分页场景优化技巧。

###### (1) 根据非主键字段排序的分页查询

看一个根据**非主键字段排序**的分页查询，SQL 如下 :

```mysql
SELECT * FROM employees ORDER BY NAME LIMIT 90000,5;
EXPLAIN SELECT * FROM employees ORDER BY NAME LIMIT 90000,5;
```

发现**并没有使用 name 字段的索引**（key 字段对应的值为 null），具体原因是**扫描整个索引并查找到没索引**
**的行(**可能要遍历多个索引树)的成本比扫描全表的成本更高，所以优化器**放弃使用索引**。

这里优化关键是让**排序时返回的字段尽可能少**，所以可以**让排序和分页操作先查出主键**，然后根据主键查到对应的记录，SQL 改写如下：

```mysql
SELECT * FROM employees e INNER JOIN (SELECT id FROM employees ORDER BY NAME LIMIT 90000, 5) ed ON e.id = ed.id;
```

观察**执行计划**之后可以发现，原 SQL 使用的是 **filesort 排序**，而优化后的 SQL 使用的是**索引排序**。  

##### 7. Join关联查询优化

建个表：

```mysql
CREATE TABLE `t1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `a` int(11) DEFAULT NULL,
    `b` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_a` (`a`)) ENGINE=InnoDB AUTO_INCREMENT=10001 DEFAULT CHARSET=utf8;
create table t2 like t1;
```

互联网公司**少用 Join**，**Join 一般都是放到业务层**进行。MySQL 的**表关联**常见有两种算法：

- **Nested-Loop Join** 算法。
- **Block Nested-Loop Join** 算法。

###### (1) 嵌套循环连接 Nested-Loop Join(NLJ) 算法

**一次一行循环**地从第一张表（称为**驱动表**）中读取行，在这行数据中取到关联字段，根据关联字段在另一张表（**被驱动**
**表**）里取出满足条件的行，然后取出两张表的结果合集。  

```mysql
EXPLAIN SELECT * FROM t1 INNER JOIN t2 ON t1.a= t2.a;
```

**驱动表是 t2**，被驱动表是 t1。**先执行的就是驱动表**(执行计划结果的 id 如果一样则按从上到下顺序执行 SQL)；优
化器**一般会优先选择小表做驱动表**。所以使用 inner join 时，排在前面的表并不一定就是驱动表。使用了 NLJ 算法。一般 join 语句中，如果执行计划 Extra 中未出现 Using join buffer 则表示使用的 join 算法是 NLJ。  

上面 SQL 的大致流程如下：

- 从表 t2 中读取一行数据；
- 从第 1 步的数据中，取出关联字段 a，到表 t1 中查找；
- 取出表 t1 中满足条件的行，跟 t2 中获取到的结果合并，作为结果返回给客户端；
- 重复上面 3 步。

整个过程会读取 t2 表的**所有数据**(扫描 100 行)，然后遍历这每行数据中**字段 a 的值**，根据 t2 表中 a 的值索引扫描 t1 表
中的对应行(扫描 100 次 t1 表的索引，1 次扫描可以认为最终只扫描 t1 表一行完整数据，也就是总共 t1 表也扫描了 100
行)。因此整个过程扫描了 200 行。如果被驱动表的关联字段没索引，使用 NLJ 算法**性能会比较低**(下面有详细解释)，MySQL 会选择 Block Nested-Loop Join 算法。

###### (2) 基于块的嵌套循环连接Block Nested-Loop Join(BNL)算法

把驱动表的数据读入到 **join_buffer** 中，然后**扫描被驱动表**，把被驱动表每一行取出来跟 join_buffer 中的数据做对比。

```mysql
EXPLAIN select*from t1 inner join t2 on t1.b= t2.b;  
```

Extra 中 的 **Using join buffer** (Block Nested Loop) 说明该关联查询使用的是 **BNL 算法**。

上面 SQL 的大致流程如下：

- 把 t2 的所有数据放入到 join_buffer 中
- 把表 t1 中每一行取出来，跟 join_buffer 中的数据做对比
- 返回满足 join 条件的数据

整个过程对表 t1 和 t2 都做了**一次全表扫描**，因此扫描的总行数为10000(表 t1 的数据总量) + 100(表 t2 的数据总量) =
10100。并且 join_buffer 里的数据是**无序**的，因此对表 t1 中的每一行，都要做 **100 次判断**，所以内存中的判断次数是
100 * 10000= **100 万次**。    

被驱动表的关联字段**没索引**为什么要选择**使用 BNL 算法**而不使用 Nested-Loop Join 呢？
如果上面第二条 SQL 使用 Nested-Loop Join，那么扫描行数为 100 * 10000 = 100万次，这个是**磁盘扫描**。
很显然，用 BNL 磁盘扫描次数少很多，相比于磁盘扫描，BNL 的**内存计算**会快得多。
因此 MySQL 对于被驱动表的**关联字段没索引的关联查询，一般都会使用 BNL 算法。如果有索引一般选择 NLJ 算法，有索引的情况下 NLJ 算法比 BNL算法性能更高。**

###### (3) 关联查询优化建议

- 对关联字段加**索引**，让 MySQL 做 join 操作时尽量选择 **NLJ 算法**。

- 让**小表驱动大表**，写多表连接 SQL 时如果明确知道哪张表是小表可以用 **straight_join** 写法固定连接驱动方式，省去 MySQL 优化器自己判断的时间。straight_join 解释：straight_join 功能同 join 类似，但**能让左边的表来驱动右边的表**，能改表优化器对于联表查询的执行顺序。比如：select * from t2 straight_join t1 on t2.a = t1.a; 代表制定 MySQL 选 t2 表作为驱动表。  

##### 8. count(*)查询优化

对于下面的语句。

```mysql
mysql> EXPLAIN select count(1) from employees;
mysql> EXPLAIN select count(id) from employees;
mysql> EXPLAIN select count(name) from employees;
mysql> EXPLAIN select count(*) from employees;
```

四个 SQL 的**执行计划一样**，说明这四个 SQL  执行效率应该**差不多**，区别在于根据某个字段 count **会不会统计字段为 null 值的数据行**。

**常见优化方法：**

- 查询 MySQL **自己维护的总行数**。对于 MyISAM 存储引擎的表做不带 where 条件的 count 查询性能是很高的，因为 MyISAM 存储引擎的表的总行数会被 MySQL 存储在磁盘上，查询不需要计算。对于 InnoDB 存储引擎的表MySQL 不会存储表的总记录行数，查询 count 需要**实时计算**。
- 将总数维护到 **Redis 里**。插入或删除表数据行的时候同时维护 Redis 里的表总行数 key 的计数值(用 incr 或 decr 命令)，但是这种方式可能不准，很难保证表操作和 Redis 操作的事务一致性。  



#### 索引设计

##### 1. 什么情况下适合建立索引

经常用作**查询条件**的字段需要创建索引，经常需要**排序、分组和统计**的字段也需要建立索引，查询中与其他表**关联**的字段，外键关系建立索引。

考虑使用**索引覆盖**。对数据很少被更新的表，如果用户经常只查询其中的几个字段，可以考虑在这几个字段上建立索引，从而将表的扫描改变为索引的扫描。 

对于**非常小**的表、大部分情况下简单的全表扫描比建立索引更高效；对于**中到大型的表，索引就非常有效**；但是对于**特大型**的表，建立和维护索引的代价将会随之增长。这种情况下，需要用到一种技术可以**直接区分**出需要查询的一组数据，而不是一条记录一条记录地匹配，例如可以使用**分区技术**。

##### 2. 什么情况下不适合建立索引

表的记录太少，百万级以下的数据不需要创建索引。频繁修改（**增删改**）的表不需要创建索引。数据**重复**且分布平均的字段**不需要**创建索引，如 true, false 之类。WHERE 条件里**用不到**的字段不需要创建索引。

##### 3. 索引设计规范

> **限制每张表上的索引数量,建议单张表索引不超过5个**

索引并不是越多越好！索引可以提高效率同样可以降低效率。索引可以增加查询效率，但同样也会降低插入和更新的效率，甚至有些情况下会降低查询效率。因为 MySQL 优化器在选择如何优化查询时，会根据统一信息，对每一个可以用到的索引来进行评估，以生成出一个最好的执行计划，如果同时有很多个索引都可以用于查询，就会增加 MySQL 优化器生成执行计划的时间，同样会降低查询性能。

> **禁止给表中的每一列都建立单独的索引**

5.6 版本之前，一个 SQL 只能使用到一个表中的一个索引，5.6 以后，虽然有了合并索引的优化方式，但是还是远远没有使用一个**联合索引**的查询方式好。

> **每个Innodb表必须有个主键**

Innodb 是一种**索引组织表**：数据的存储的逻辑顺序和索引的顺序是相同的。每个表都可以有多个索引，但是表的存储顺序只能有一种。Innodb 是按照**主键索引的顺序**来组织表的。

> **常见索引列建议**

- 出现在 SELECT、UPDATE、DELETE 语句的 WHERE 从句中的列。
- 包含在 ORDER BY、GROUP BY、DISTINCT 中的字段。
- 并不要将符合 1 和 2 中的字段的列都建立一个索引， 通常将 1、2 中的字段建立联合索引效果更好。
- 多表 join 的关联列。

> **如何选择索引列的顺序**

建立索引的目的是：希望通过索引进行数据查找，减少随机 IO，增加查询性能 ，索引能过滤出越少的数据，则从磁盘中读入的数据也就越少。

- 区分度最高的放在联合索引的最左侧（区分度=列中不同值的数量/列的总行数）。
- 尽量把字段长度小的列放在联合索引的最左侧（因为字段长度越小，一页能存储的数据量越大，IO 性能也就越好）。
- 使用最频繁的列放到联合索引的左侧（这样可以比较少的建立一些索引）。

> **避免建立冗余索引和重复索引（增加了查询优化器生成执行计划的时间）**

- 重复索引示例：primary key(id)、index(id)、unique index(id)
- 冗余索引示例：index(a,b,c)、index(a,b)、index(a)

> **对于频繁的查询优先考虑使用覆盖索引**

**覆盖索引**：就是包含了所有查询字段 (where,select,ordery by,group by 包含的字段) 的索引。

**覆盖索引的好处：**

- **避免 Innodb 表进行索引的二次查询:** Innodb 是以聚集索引的顺序来存储的，对于 Innodb 来说，二级索引在叶子节点中所保存的是行的主键信息，如果是用二级索引查询数据的话，在查找到相应的键值后，还要通过主键进行二次查询才能获取我们真实所需要的数据。而在覆盖索引中，二级索引的键值中可以获取所有的数据，避免了对主键的二次查询 ，减少了 IO 操作，提升了查询效率。
- •**可以把随机 IO 变成顺序 IO 加快查询效率:** 由于覆盖索引是按键值的顺序存储的，对于 IO 密集型的范围查找来说，对比随机从磁盘读取每一行的数据 IO 要少的多，因此利用覆盖索引在访问时也可以把磁盘的随机读取的 IO 转变成索引查找的顺序 IO。

> **索引SET规范**

**尽量避免使用外键约束**

- 不建议使用外键约束（foreign key），但一定要在表与表之间的关联键上建立索引。
- 外键可用于保证数据的参照完整性，但建议在业务端实现。
- 外键会影响父表和子表的写操作从而降低性能





看看这个帖子：美团的：https://www.cnblogs.com/php-rearch/p/5034118.html