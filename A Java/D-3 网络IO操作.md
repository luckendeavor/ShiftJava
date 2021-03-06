[TOC]

### 网络IO

IO 部分主要放到了 Netty 部分，参考后面。



### 网络操作

Java 中的网络支持：

- **InetAddress**：用于表示网络上的硬件资源，即 IP 地址；
- **URL**：统一资源定位符；
- **Sockets**：使用 TCP 协议实现网络通信；
- **Datagram**：使用 UDP 协议实现网络通信。

#### InetAddress

没有公有的构造函数，只能通过**静态方法**来创建实例。

```java
InetAddress.getByName(String host);
InetAddress.getByAddress(byte[] address);
```

#### URL

可以直接从 URL 中读取**字节流**数据。

```java
public static void main(String[] args) throws IOException {
    URL url = new URL("http://www.baidu.com");
    /* 字节流 */
    InputStream is = url.openStream();
    /* 转换成字符流 */
    InputStreamReader isr = new InputStreamReader(is, "utf-8");
    /* 装饰成缓存流具有缓冲功能 */
    BufferedReader br = new BufferedReader(isr);
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
    br.close();
}
```

#### Sockets

- **ServerSocket**：服务器端类
- **Socket**：客户端类
- 服务器和客户端通过 InputStream 和 OutputStream 进行**输入输出**。下面的图是 BIO 模型。

<img src="assets/1563439394302.png" alt="1563439394302" style="zoom:70%;" />

#### Datagram类

- DatagramSocket：通信类
- DatagramPacket：数据报类





