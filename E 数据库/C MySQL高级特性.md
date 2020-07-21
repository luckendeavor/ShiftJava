[TOC]

### MySQL高级特性

#### MySQL函数

**调用函数：**

```mysql
SELECT 函数名（实参列表）
```

##### 1. 创建函数

```mysql
CREATE FUNCTION 函数名(参数名 参数类型,...) RETURNS 返回类型
BEGIN
	函数体
END
```

##### 2. MySQL函数内置

各个 DBMS 的函数都是不相同的，因此**不可移植**。这些内置函数的使用方法详见**基础部分**。

###### (1) 聚集函数

|    函 数    |        说 明         |
| :---------: | :------------------: |
|  **AVG()**  | 返回**某列的平均值** |
| **COUNT()** |  返回**某列的行数**  |
|  **MAX()**  | 返回**某列的最大值** |
|  **MIN()**  | 返回**某列的最小值** |
|  **SUM()**  |  返回**某列值之和**  |

**AVG**() 会忽略 **NULL** 行。

使用 **DISTINCT** 可以让汇总函数值汇总不同的值。

```sql
SELECT AVG(DISTINCT col1) AS avg_col FROM mytable;
```

###### (2) 文本处理函数

|   函数   |      说明      |   函数    |      说明      |
| :------: | :------------: | :-------: | :------------: |
|  LEFT()  |   左边的字符   |  RIGHT()  |   右边的字符   |
| LOWER()  | 转换为小写字符 |  UPPER()  | 转换为大写字符 |
| LTRIM()  | 去除左边的空格 |  RTRIM()  | 去除右边的空格 |
| LENGTH() |      长度      | SOUNDEX() |  转换为语音值  |

其中  **SOUNDEX()**  可以将一个字符串转换为描述其语音表示的字母数字模式。

```sql
SELECT * FROM mytable WHERE SOUNDEX(col1) = SOUNDEX('apple');
```

###### (3) 日期和时间处理


- **日期格式**：YYYY-MM-DD。
- **时间格式**：HH:MM:SS。

|     函 数     |            说 明             |    函数     |              说明              |
| :-----------: | :--------------------------: | :---------: | :----------------------------: |
|   AddDate()   |   增加一个日期（天、周等）   | DayOfWeek() | 对于一个日期，返回对应的星期几 |
|   AddTime()   |   增加一个时间（时、分等）   |   Hour()    |     返回一个时间的小时部分     |
|   CurDate()   |         返回当前日期         |  Minute()   |     返回一个时间的分钟部分     |
|   CurTime()   |         返回当前时间         |   Month()   |     返回一个日期的月份部分     |
|    Date()     |    返回日期时间的日期部分    |    Now()    |       返回当前日期和时间       |
|  DateDiff()   |       计算两个日期之差       |  Second()   |      返回一个时间的秒部分      |
|  Date_Add()   |    高度灵活的日期运算函数    |   Time()    |   返回一个日期时间的时间部分   |
| Date_Format() | 返回一个格式化的日期或时间串 |   Year()    |     返回一个日期的年份部分     |
|     Day()     |    返回一个日期的天数部分    |             |                                |

```mysql
mysql> SELECT NOW();  # 2018-4-14 20:25:11
```

###### (4) 数值处理函数

|    函数    |    说明    | 函数  |  说明  |
| :--------: | :--------: | :---: | :----: |
|   SIN()    |    正弦    | COS() |  余弦  |
|   TAN()    |    正切    | ABS() | 绝对值 |
|   SQRT()   |   平方根   | MOD() |  余数  |
|   EXP()    |    指数    | PI()  | 圆周率 |
| **RAND**() | **随机数** |       |        |



#### 存储过程

##### 1. 概述

**存储过程**就是一组经过**预先编译**的 SQL 语句的集合。

一般来说，存储过程实现的功能要复杂一点，而函数的实现的功能针对性比较强。存储过程，功能强大，可以执行包括修改表等一系列数据库操作；用户定义函数不能用于执行一组修改全局数据库状态的操作。

存储过程可以看成是对一系列 SQL 操作的**批处理**。使用存储过程的好处：代码封装，保证了一定的安全性；代码复用；由于是预先编译，因此具有很高的性能。但是阿里巴巴的开发规范规定**别用**存储过程。

命令行中创建存储过程**==需要自定义分隔符==**，因为命令行是以 **; 为结束符**，而存储过程中也包含了**分号**，因此会错误把这部分分号当成是结束符，造成语法错误。

存储过程包含 **in、out 和 inout** 三种参数。

给变量赋值都需要用 **SELECT INTO** 语句，且每次只能给**一个变量**赋值，不支持集合的操作。

##### 2. 创建存储过程

语法：

```mysql
CREATE PROCEDURE 存储过程名(IN|OUT|INOUT 参数名 参数类型, ...)
BEGIN
	存储过程体
END;
```

类似于方法：

```mysql
修饰符 返回类型 方法名(参数类型 参数名, ...){
	方法体;
}
```

注意：

```mysql
# 1.需要设置新的结束标记
DELIMITER 新的结束标记;
示例：
DELIMITER $

CREATE PROCEDURE 存储过程名(IN|OUT|INOUT 参数名  参数类型,...)
BEGIN
	sql语句1;
	sql语句2;
END $ 

# 2.存储过程体中可以有多条SQL语句，如果仅仅一条SQL语句，则可以省略BEGIN END

# 3.参数前面的符号的意思
IN: 该参数只能作为输入（该参数不能做返回值）
OUT: 该参数只能作为输出（该参数只能做返回值）
INOUT: 既能做输入又能做输出
```

完整示例：

```mysql
delimiter //		# 自定义分隔符

create procedure myprocedure(out ret int)
    begin
        declare y int;
        select sum(col1)
        from mytable
        into y;
        select y*y into ret;
    end //

delimiter ;			# 换回分隔符
```

##### 3. 调用存储过程

```mysql
CALL 存储过程名(实参列表);
```

```mysql
call myprocedure(@ret);
select @ret;
```

##### 4. 游标

在存储过程中使用**游标**可以对**一个结果集**进行**移动遍历**。游标主要用于**交互式应用**，其中用户需要对数据集中的任意行进行浏览和修改。使用游标的四个步骤：

1. **声明游标**，这个过程**没有**实际检索出数据；
2. **打开游标**；
3. **取出数据**；
4. **关闭游标**；

```sql
delimiter //
create procedure myprocedure(out ret int)
    begin
        declare done boolean default 0;
        declare mycursor cursor for
        select col1 from mytable;
        # 定义了一个continue handler，当sqlstate '02000' 这个条件出现时，会执行set done = 1
        declare continue handler for sqlstate '02000' set done = 1;
        open mycursor;
        repeat
            fetch mycursor into ret;
            select ret;
        until done end repeat;
        close mycursor;
    end //
 delimiter ;
```



#### 视图

##### 1. 概述

视图是一张**虚拟的表**，它是通过表**动态生成**的数据。

视图和表的区别：两者使用方式**完全相同**，不过表需要占用物理空间，而视图**不占用物理空间**，仅仅保存的是 SQL 逻辑。

使用视图可以简化复杂的 SQL 操作，比如复杂的连接，且只使用实际表的一**部分数据**，视图可以通过只给用户**访问视图的权限**，保证数据的安全性。

##### 2. 视图的基本操作

**查询语句**放在 AS 后面。

```mysql
# 创建视图
CREATE VIEW 视图名 AS 查询语句;
# 删除视图
DROP VIEW test1, test2;
# 查看视图结构
DESC test1;
SHOW CREATE VIEW test1;
```

##### 3. 视图的增删改查

```mysql
# 1.查看视图的数据 ★
SELECT * FROM my_v4;
SELECT * FROM my_v1 WHERE last_name = 'Partners';
# 2.插入视图的数据
INSERT INTO my_v4(last_name, department_id) VALUES('虚竹',90);
# 3.修改视图的数据
UPDATE my_v4 SET last_name = '梦姑' WHERE last_ame = '虚竹';
# 4.删除视图的数据
DELETE FROM my_v4;
```

##### 4. 视图的作用

- 测试表：user 有 id，name，age，sex 字段。

- 测试表：goods 有 id，name，price 字段。

- 测试表：ug 有 id，userid，goodsid 字段。

**作用一：** 提高了**重用性**，就像一个函数。如果要**频繁获取** user 的 name 和 goods 的 name。就可以使用以下 SQL 语言。示例：

 ```mysql
SELECT a.name AS username, b.name AS goodsname FROM USER AS a, goods AS b, ug AS c WHERE a.id = c.userid AND c.goodsid = b.id;
 ```

这里利用上述的语句创建视图 **other**。示例：

 ```mysql
CREATE VIEW other AS SELECT a.name AS username, b.name AS goodsname FROM USER AS a, goods AS b, ug AS c WHERE a.id = c.userid AND c.goodsid = b.id;
 ```

创建好视图后就可以**通过视图获取** user 的 name 和 goods 的 name。示例：

 ```mysql
select * from other;
 ```

**作用二：**对数据库**重构**，却不影响程序的运行。有两个 user 相关的表的结构如下：

- 测试表：usera 有 id，name，age 字段。
- 测试表：userb 有 id，name，sex 字段。

现在想在结果上合并两个表到 user 中，但是物理上又不合并，这里就可以**创建视图**。以下 SQL 语句创建视图 user：

```mysql
create view user as select a.name, a.age, b.sex from usera as a, userb as b where a.name = b.name;
```

以上假设 name 都是**唯一**的。此时使用 SQL 语句：select * from user; 也不会报错。这就实现了**更改数据库结构**，不更改脚本程序的功能了。

**作用三：** 提高了安全性能。可以对**不同的用户，设定不同的视图**。例如：某用户只能获取 user 表的 name 和 age 数据，不能获取 sex 数据。则可以这样创建视图。示例如下：

 ```mysql
create view other as select a.name, a.age from user as a;
 ```

这样的话，使用 SQL 语句：select * from other; 最多就只能获取 name 和 age 的数据，其他的数据就获取不了了。

**作用四：** 让数据更加清晰。想要什么样的数据，就创建什么样的视图。

##### 5. 某些视图不能更新

如果**视图中数据**是来自于**一个表**时，**修改视图中的数据**，表数据会**更新**。而且**修改表中数据**时，对应视图也会**更新**。但是如果视图数据来源于**两个表**时，修改视图数据时会报错，**无法**修改。



#### 触发器

##### 1. 概述

**触发器**是与表有关的数据库对象，在**满足定义条件时触发**，并**执行触发器中定义的语句**集合。触发器的这种特性可以**协助**应用在数据库端确保数据的完整性。

比如有两个表【用户表】和【日志表】，当一个用户被创建的时候，就需要在日志表中插入创建的 log 日志，如果在不使用触发器就需要**编写程序语言**逻辑才能实现，但如果定义一个触发器就可以实现在用户表中插入一条数据之后在日志表中插入一条日志信息。当然触发器并不是只能进行**插入**操作，还能执行**修改，删除。**

##### 2. 创建触发器

创建**触发器的语法**如下：

```mysql
CREATE TRIGGER trigger_name trigger_time trigger_event ON tb_name FOR EACH ROW;
# trigger_stmt
# trigger_name：触发器的名称
# tirgger_time：触发时机，s为BEFORE或者AFTER
# trigger_event：触发事件，为INSERT、DELETE或者UPDATE
# tb_name：表示建立触发器的表明，就是在哪张表上建立触发器
# trigger_stmt：触发器的程序体，可以是一条SQL语句或者是用BEGIN和END包含的多条语句
```

```mysql
# 所以可以说MySQL根据触发时机创建以下六种触发器：
BEFORE INSERT, BEFORE DELETE, BEFORE UPDATE
AFTER INSERT, AFTER DELETE, AFTER UPDATE
```

其中，触发器名参数指要创建的触发器的名字。BEFORE 和 AFTER 参数指定了触发**执行的时间**，在事件**之前**或是**之后**。FOR EACH ROW 表示**任何一条记录**上的操作满足触发事件都会触发该触发器。

**创建有多个==执行语句==的触发器**：

```mysql
CREATE TRIGGER 触发器名 BEFORE|AFTER 触发事件
ON 表名 FOR EACH ROW
BEGIN
    执行语句列表
END;
```

其中，**BEGIN 与 END** 之间的执行语句列表参数表示需要执行的**多个**语句，不同语句用**分号**隔开。

一般情况下，MySQL 默认是以 **;** 作为结束执行语句，与触发器中需要的**分行**起冲突，为解决此问题可用 **DELIMITER**，如：DELIMITER ||，可以将结束符号变成 **||**， 当触发器创建完成后，可以用 **DELIMITER ;** 来将结束符号变成 **;** 。这与**存储过程的使用类似**。

```mysql
mysql> DELIMITER ||		# 修改默认的结束符号
mysql> CREATE TRIGGER demo BEFORE DELETE
    -> ON users FOR EACH ROW		# 每一行都观测
    -> BEGIN
    -> INSERT INTO logs VALUES(NOW());	# BEGIN与END中间是触发时执行的逻辑
    -> INSERT INTO logs VALUES(NOW());
    -> END
    -> ||
Query OK, 0 rows affected (0.06 sec)

mysql> DELIMITER ;	# 还原默认的结束符号
```

上面的语句中，开头将结束符号定义为 ||，中间定义一个触发器，一旦有满足条件的**删除操作**。就会执行 BEGIN 和 END 中的语句，接着**使用 || 结束**。最后使用DELIMITER ; 将结束符号**还原**。

##### 3. 其他

触发器会在**某个表**执行以下语句时而**自动执行**：**==DELETE、INSERT、UPDATE==** 等操作。触发器必须指定在语句执行**之前**还是**之后**自动执行，之前执行使用 **BEFORE** 关键字，之后执行使用 **AFTER** 关键字。**BEFORE 用于数据验证和净化，AFTER 用于审计跟踪**，将修改记录到另外一张表中。

INSERT 触发器包含一个名为 **NEW 的虚拟表**。

```sql
CREATE TRIGGER mytrigger AFTER INSERT ON mytable
FOR EACH ROW SELECT NEW.col into @result;

SELECT @result; -- 获取结果
```

**DELETE** 触发器包含一个名为 **OLD** 的虚拟表，并且是**只读**的。

**UPDATE** 触发器包含一个名为 **NEW** 和一个名为 OLD 的虚拟表，其中 **NEW 是可以被修改**的，而 OLD 是**只读的**。

MySQL 不允许在触发器中使用 CALL 语句，也就是**不能调用存储过程**。







