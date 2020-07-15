[TOC]

### RESTful API设计

#### 概述

##### 1. 概述

Representational State Transfer，翻译是 "表现层状态转移"，通俗解释就是 **“资源”在网络传输中以某种“表现形式”进行“状态转移”** 。 **面向资源**是 REST 最明显的特征，对于同一个资源的一组不同的操作。资源是服务器上一个可命名的抽象概念，资源是以名词为核心来组织的，首先关注的是**名词**。REST 要求必须通过统一的接口来对资源执行各种操作。对于每个资源只能执行一组有限的操作（7 个 HTTP 方法：GET/POST/PUT/DELETE/PATCH/HEAD/OPTIONS ）。

RESTful API 可以让我们看到 **URL + HTTP 方法**就知道这个 URL 是干什么的，看到了 **HTTP 状态码**（status code）就知道请求**结果**如何。

```
GET    /classes：列出所有班级
POST   /classes：新建一个班级
```

REST 是所有 Web 应用都应该遵守的架构设计指导原则。 

总结一下什么是 **RESTful 架构**：

1. 每一个 **URI 代表一种资源**；
2. 客户端和服务器之间通过某种**表现形式**比如 json，xml，image，txt 等**传递这种资源**；
3. 客户端通过特定的 HTTP 动词，对服务器端**资源进行操作**，实现"**表现层状态转化**"。

##### 2. 基本术语

- **资源（Resource）**：可以把**真实的对象数据**称为**资源**。一个资源既可以是一个集合，也可以是单个个体。比如班级 classes 是代表一个集合形式的资源，而特定的 class 代表单个个体资源。**每一种资源都有特定的 URI（统一资源定位符）与之对应**，如果需要获取这个资源，访问这个 URI 就可以了，比如获取特定的班级：/class/12。
- **表现形式（Representational）**："资源" 是一种**信息实体**，它可以有**多种外在表现形式**。把"资源"具体**呈现**出来的形式比如 **json，xml，image，txt** 等叫做它的"表现层/表现形式"。
- **状态转移（State Transfer）**：通俗说 REST 中的状态转移更多地描述的**服务器端资源的状态**，比如通过增删改查（通过 HTTP 动词实现）**引起资源状态的改变**。

#### REST接口规范

##### 1. 动作

- **GET** ：请求从服务器**获取**特定资源。举个例子：`GET /classes`（获取所有班级）。
- **POST** ：在服务器上**创建**一个新的资源。举个例子：`POST /classes`（创建班级）。
- **PUT** ：**更新**服务器上的资源（客户端提供更新后的整个资源）。举个例子：`PUT /classes/12`（更新编号为 12 的班级）。
- **DELETE** ：从服务器**删除**特定的资源。举个例子：`DELETE /classes/12`（删除编号为 12 的班级）。
- **PATCH** ：更新服务器上的资源（客户端提供更改的属性，可以看做作是部分更新），使用的**比较少**，这里就不举例子了。

##### 2. 路径(接口命名)

路径又称"**终点**"（endpoint），表示 API 的具体网址。实际开发中常见的规范如下：

- **网址中不能有动词，只能有名词，API 中的名词也应该使用复数**。 因为 REST 中的资源往往和数据库中的表对应，而数据库中的表都是同种记录的"集合"（collection）。如果 API 调用**并不涉及资源**（如计算，翻译等操作）的话，**可以用动词**。 比如：GET /calculate?param1=11&param2=33。
- 不用大写字母，建议**用中杠 - 不用下杠 \_**。比如邀请码写成 invitation-code 而不是 ~~invitation_code~~。

**接口尽量使用名词，禁止使用动词。** 下面是一些例子：

```react
GET    /classes：列出所有班级
POST   /classes：新建一个班级
GET    /classes/classId：获取某个指定班级的信息
PUT    /classes/classId：更新某个指定班级的信息（一般倾向整体更新）
PATCH  /classes/classId：更新某个指定班级的信息（一般倾向部分更新）
DELETE /classes/classId：删除某个班级
GET    /classes/classId/teachers：列出某个指定班级的所有老师的信息
GET    /classes/classId/students：列出某个指定班级的所有学生的信息
DELETE /classes/classId/teachers/ID：删除某个指定班级下的指定的老师的信息
```

**反例：**

```
/getAllclasses
/createNewclass
/deleteAllActiveclasses
```

应该理清**资源的层次结构**，比如业务针对的范围是学校，那么学校会是**一级**资源：/schools，老师： /schools/teachers，学生：/schools/students 就是**二级**资源。

##### 3. 过滤信息(Filtering)

如果在查询的时候需要**添加特定条件**的话，建议使用 **URL 参数**的形式。比如要查询 state 状态为 active 并且 name 为 guidegege 的班级：

```http
GET /classes?state=active&name=guidegege
```

比如要实现**分页查询**：

```http
GET /classes?page=1&size=10 // 指定第1页，每页10个数据
```

##### 4. HTTP状态码(Status Codes)

常见 HTTP 状态码如下：

| 2xx 成功 |   3xx 重定向   |  4xx 客户端错误  | 5xx 服务器错误 |
| :------: | :------------: | :--------------: | :------------: |
| 200 成功 | 301 永久重定向 |   400 错误请求   | 500 服务器错误 |
| 201 创建 | 304 资源未修改 |    401 未授权    |  502 网关错误  |
|          |                |   403 禁止访问   |  504 网关超时  |
|          |                |    404 未找到    |                |
|          |                | 405 请求方法不对 |                |







