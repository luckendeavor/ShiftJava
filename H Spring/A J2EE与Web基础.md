[TOC]

### J2EE

Spring 之前还是了解一下之前的老东西，而且 Servlet 也是非常重要的。

#### JSP

##### 1. 概述

JSP 是一种 Servlet，但是与 HttpServlet 的工作方式**不太一样**。HttpServlet 是先由**源代码编译**为 **class 文件**后部署到服务器下，为**先编译后部署**。而 JSP 则是**先部署后编译**。JSP 会在客户端第一次请求 JSP 文件时被编译为 HttpJspPage 类（Servlet 接口的一个子类）。该类会被服务器临时存放在服务器工作目录里面。 

由于 JSP 只会在客户端**第一次请求的时候被编译** ，因此第一次请求 JSP 时会感觉比较**慢**，之后就会快很多。如果把服务器保存的 class 文件删除，服务器也会重新编译 JSP。

开发 Web 程序时经常需要修改 JSP。Tomcat 能够自动检测到 JSP 程序的改动。如果检测到 JSP 源代码发生了改动。Tomcat 会在下次客户端请求 JSP 时重新编译 JSP，而不需要重启 Tomcat。这种自动检测功能是默认开启的，检测改动会消耗少量的时间，在部署 Web 应用的时候可以在 web.xml 中将它关掉。

参考：《Javaweb 整合开发王者归来》P97

##### 2. 四种作用域

JSP 中的四种作用域包括 page、request、session 和 application，具体来说：

- **page**：代表与一个页面相关的对象和属性。
- **request**：代表与 Web 客户端发出的一个请求相关的对象和属性。一个请求可能跨越多个页面，涉及多个 Web 组件；需要在页面显示的临时数据可以置于此作用域。
- **session**：代表与某个用户与服务器建立的一次会话相关的对象和属性。跟某个用户相关的数据应该放在用户自己的session中。
- **application**：代表与整个 Web 应用程序相关的对象和属性，它实质上是跨越整个 Web 应用程序，包括多个页面、请求和会话的一个全局作用域。

##### 3. JSP内置对象

JSP 有 9 个内置对象：

- request：封装客户端的请求，其中包含来自 GET 或 POST 请求的参数；
- response：封装服务器对客户端的响应；
- pageContext：通过该对象可以获取其他对象；
- session：封装用户会话的对象；
- application：封装服务器运行环境的对象；
- out：输出服务器响应的输出流对象；
- config：Web 应用的配置对象；
- page：JSP 页面本身（相当于 Java 程序中的 this）；
- exception：封装页面抛出异常的对象。



#### Servlet

##### 1. 概述

Servlet 是在**服务器**上运行的小程序。一个 servlet 就是一个 Java 类，并且可以通过 "请求—响应" 编程模式来访问的这个驻留在服务器内存里的 servlet 程序。  

类的继承关系如下：

<img src="assets/1535532891036.png" style="zoom:56%;" />

Servlet 三种**实现方式**：

- 实现 javax.servlet.**Servlet** 接口。

- 继承 javax.servlet.**GenericServlet** 类。
- 继承 javax.servlet.http.**HttpServlet** 类。通常会去继承 HttpServlet 类来完成 Servlet。

在 JavaWeb 程序中，Servlet 主要负责**接收用户请求 HttpServletRequest**，在 **doGet()，doPost()** 中做相应的处理，并将回应 **HttpServletResponse** 反馈给用户。Servlet 可以设置初始化参数，供 Servlet 内部使用。一个 Servlet 类只会有**一个实例**，在它初始化时调用 **init**() 方法，销毁时调用 **destroy**() 方法。Servlet 需要在 **web.xml** 中配置，一个 Servlet 可以设置多个 URL 访问。**Servlet 不是线程安全**，因此要谨慎使用类变量。

##### 2. Servlet的优点

优点如下：

- 只需要启动一个操作系统进程以及加载**一个 JVM**，大大降低了系统的开销。如果多个请求需要做同样处理的时候，这时候只需要**加载一个类**，这也大大降低了开销。
- 所有**动态加载**的类可以实现对网络协议以及请求解码的**共享**，大大降低了工作量。
- Servlet 能直接和 Web 服务器交互，而普通的 CGI 程序不能。Servlet 还能在各个程序之间共享数据，使数据库连接池之类的功能很容易实现。

##### 3. Servlet接口与生命周期

Servlet 接口定义了 5 个方法，其中**前三个方法与 Servlet 生命周期相关**：

```java
void init(ServletConfig config) throws ServletException;
void service(ServletRequest req, ServletResponse resp) throws ServletException, java.io.IOException;
void destroy();
java.lang.String getServletInfo();
ServletConfig getServletConfig();
```

Servlet 生命周期如下：

- Web 容器加载 Servlet 并将其实例化后，Servlet 生命周期开始，容器运行其 **init() 方法**进行 Servlet 的**初始化**。
- **请求到达**时调用 Servlet 的 **service**() 方法，service() 方法会根据需要调用与请求对应的 **doGet 或 doPost** 等方法。
- 当服务器**关闭**或项目被卸载时服务器会将 Servlet 实例销毁，此时会调用 Servlet 的 **destroy**() 方法。

init 方法和 destroy 方法**只会执行一次**，service 方法客户端每次请求 Servlet 都会执行。Servlet 中有时会用到一些需要初始化与销毁的资源，因此可以把初始化资源的代码放入 init 方法中，销毁资源的代码放入 destroy 方法中，这样就不需要每次处理客户端的请求都要初始化与销毁资源。

- Servlet 类由自己编写，但**对象由服务器**来创建，并由服务器来调用相应的方法。　
- 服务器**启动**时 ( web.xml 中配置 load-on-startup=1，默认为 0 ) 或者**第一次请求该 servlet** 时，就会**初始化**一个 Servlet 对象，也就是**会执行初始化方法 init**(ServletConfig conf)。
- 该 servlet 对象去处理所有**客户端请求**，在 **service**(ServletRequest req，ServletResponse res) 方法中执行。
- 最后服务器**关闭**时，才会销毁这个 servlet 对象，执行 **destroy**() 方法。

<img src="assets/1535535812505.png" style="zoom:42%;" />

##### 4. 线程安全性

**Servlet 不是线程安全的，多线程并发的读写会导致数据不同步的问题。** 

解决的办法是**尽量不要定义 name 属性**，而是要把 name 变量分别定义在 doGet() 和 doPost() 方法内。虽然使用synchronized(name){} 语句块可以解决问题，但是会造成线程的等待。 注意：**多线程的并发的读写 Servlet 类属性会导致数据不同步**。但是如果只是并发地读取属性而不写入，则不存在数据不同步的问题。因此 Servlet 里的**只读属性**最好定义为 final 类型的。

##### 5. Tomcat和Servlet的联系

Tomcat 是 Web 应用服务器，是一个 Servlet/JSP **容器**。Tomcat 作为 **Servlet 容器**，负责处理客户请求，把请求传送给 Servlet，并将 Servlet 的**响应传送回给客户**。而 Servlet 是一种运行在支持 Java 语言的服务器上的组件。Servlet 最常见的用途是扩展 Java Web 服务器功能，提供非常安全的，可移植的，易于使用的 CGI 替代品。

从 HTTP 协议中的请求和响应可以得知，浏览器发出的请求是一个**请求文本**，而浏览器接收到的也应该是一个响应文本。但是在下面这个图中，并不知道是如何转变的，只知道浏览器发送过来的请求也就是 request，我们响应回去的就用 response。忽略了其中的细节，现在就来探究一下。

<img src="assets/servlet-tomcat.png" alt="servlet-tomcat" style="zoom:45%;" />

(1) Tomcat 将 HTTP 请求文本接收**并解析**，然后封装成 **HttpServletRequest** 类型的 **request 对象**，所有的 HTTP **头数据**读可以通过 request 对象调用对应的方法查询到。

(2) Tomcat 同时会要**响应的信息封装为 HttpServletResponse 类型的 response 对象**，通过设置 response 属性就可以控制要输出到浏览器的内容，然后将 response 交给 Tomcat，Tomcat 就会将其变成响应文本的格式发送给浏览器。

Java Servlet API 是 **Servlet 容器(Tomcat) 和 servlet 之间的接口**，它定义了 serlvet 的各种方法，还定义了 Servlet 容器传送给 Servlet 的对象类，其中最重要的就是 **ServletRequest** 和 **ServletResponse**。所以在编写 servlet 时，需要实现 Servlet 接口，按照其规范进行操作。

##### 6. Tomcat装载Servlet的三种情况

(1) Servlet 容器**启动时**自动装载某些 Servlet，实现它只需要在 web.xml 文件中的 `<servlet></servlet>` 之间添加以下代码：

```xml
<load-on-startup>1</load-on-startup>
```

其中，数字越小表示优先级越高。例如在 web.xml 中设置 TestServlet2 的优先级为 1，而 TestServlet1 的优先级为 2，启动和关闭 Tomcat：优先级高的先启动也先关闭。　　

(2) 客户端首次向某个 Servlet **发送请求**。

(3) Servlet 类被**修改**后，Tomcat 容器会重新装载 Servlet。



#### 转发(Forward)和重定向(Redirect)的区别

**转发是服务器行为，重定向是客户端行为。**

**转发（Forward）** 通过 **RequestDispatcher** 对象的 forward（HttpServletRequest request,HttpServletResponse response）方法实现的。RequestDispatcher 可以通过 HttpServletRequest 的 getRequestDispatcher() 方法获得。例如下面的代码就是跳转到 login_success.jsp 页面。

```java
request.getRequestDispatcher("login_success.jsp").forward(request, response);
```

**重定向（Redirect）** 是利用**服务器返回的状态码**来实现的。客户端浏览器请求服务器的时候，服务器会返回一个状态码。服务器通过 **HttpServletResponse** 的 `setStatus(int status)` 方法**设置状态码**。如果服务器返回 301 或者 302，则浏览器会到**新的网址**重新请求该资源。

- **地址栏显示**：forward 是服务器请求资源，服务器直接访问目标地址的 URL，把那个 URL 的响应内容读取过来，然后把这些内容再发给浏览器。浏览器根本**不知道**服务器发送的内容从哪里来的，所以它的地址栏还是原来的地址。 redirect 是服务端根据逻辑，发送一个状态码，告诉浏览器重新去请求那个地址，所以**地址栏显示的是新的 URL**。

- **数据共享**：forward 转发页面和转发到的页面可以**共享** request 里面的数据。redirect 不能共享数据。

- **使用场景**：forward一般用于**用户登陆**的时候，根据角色转发到相应的模块。 redirect 一般用于用户注销登陆时返回主页面和跳转到其它的网站等。
- **效率**：forward 效率高，redirect 效率低。



#### 会话跟踪技术

##### 1. 使用Cookie

向客户端发送 Cookie。

```java
Cookie c = new Cookie("name","value"); // 创建Cookie 
c.setMaxAge(60*60*24);   // 设置最大时效，此处设置的最大时效为一天
response.addCookie(c);   // 把Cookie放入到HTTP响应中
```

从客户端读取 Cookie。

```java
String name = "name"; 
Cookie[]cookies = request.getCookies(); 
if(cookies != null){ 
    for(int i = 0; i< cookies.length; i++){ 
        Cookie cookie = cookies[i]; 
        if(name.equals(cookis.getName())) 
            //something is here. 
            //you can get the value 
            cookie.getValue(); 
    }
}
```

**优点:** 数据可以持久保存，不需要服务器资源，简单，基于文本的 Key-Value。

**缺点:** 大小受到限制，用户可以禁用 Cookie 功能，由于保存在本地，有一定的安全风险。

##### 2. URL重写

在 URL 中添加**用户会话的信息**作为**请求的参数**，或者将唯一的**会话 ID 添加到 URL 结尾**以标识一个会话。

**优点：** 在 Cookie 被禁用的时候依然可以使用。

**缺点：** 必须对网站的 URL 进行编码，所有页面必须动态生成，不能用预先记录下来的 URL 进行访问。

##### 3. 隐藏的表单域

```
<input type="hidden" name ="session" value="..."/>
```

**优点：** Cookie 被禁时可以使用。

**缺点：** 所有页面必须是表单提交之后的结果。

##### 4. HttpSession

在所有会话跟踪技术中，HttpSession 对象是最强大也是功能最多的。当一个用户第一次访问某个网站时会自动创建 **HttpSession**，每个用户可以访问其自己的 HttpSession。可以通过 HttpServletRequest 对象的 getSession 方法获得HttpSession，通过 HttpSession 的 **setAttribute** 方法可以将一个**值**放在 HttpSession 中，通过调用 HttpSession 对象的 **getAttribute** 方法，同时传入属性名就可以获取保存在 HttpSession 中的对象。

与上面三种方式不同的是，HttpSession 放在服务器的**内存**中，因此不要将过大的对象放在里面，即使目前的 Servlet 容器可以在内存将满时将 HttpSession 中的对象移到其他存储设备中，但是这样会影响性能。