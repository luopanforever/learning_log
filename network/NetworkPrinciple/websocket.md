# 什么是`websocket`

为了使得`webserver`能够做长连接

使得了服务器可以主动推送数据给客户端



# **网络编程重点**

`socket` ，`recv/send`，`epoll`，网络`io`，`reactor`，`libevent/libev` 。`1`对`1`,`1`对`n`，`udp`，异步请求，多进程多线程

具体应用`->`

- `redis`
- `nginx`
- `memchche`

# 网络原理重点

**重点**：**`tcp`原理，`udp`原理**，`http`原理，

非重点：`ip`协议/`eth`以太网协议/`arp`/`icmp `

这一部分了解即可  代码层不会遇到

# **应用`websocket`的例子**

- 第三方扫码登录

  - 业务：一个浏览器，请求到了`csdn`的`server`（`csdn.net`），登录的时候出现二维码扫码登录，

    于是拿手机微信扫一下，会上传数据到微信的服务器上，

    微信的服务器会开放一个开发者平台，就可以申请一个账号，账号可以设置一个回调函数，于是就由腾讯的开发者账号回调到`csdn`的`server`上。`csdn`

    收到信息后会**主动**推送登录页面的信息（这一步的实现就是使用的`websocket`）

- 实时数据更新

# 如何区别`websocket`与`http`请求报文

- 首先判断`connection`的`value`值，再判断`upgrade`是否是`websocket`

# **`websocket`握手的流程**

- 一来一回，没有三次

- 发什么？回什么？

## 发来什么

一般是`web`端发来一个请求报文里面包含着`sec-websocket-key`

需要提取该`sec-websocket-key`值用来后续回包时候封装`sec-websocket-accept`

## 回什么

利用发来的`sec-websocket-key`进行**处理**后得到`sec-websocket-accept`放入响应头回给发送端。

### 如何处理`sec-websocket-key`为`sec-websocket-accept`

**前置**：所有使用`websocket`协议都公用一个`GUID`

**操作**：

- 拿出`SEC-WebSocket-key`的`key`与`GUID`连接起来
- `server`对合成的字符串做一个`hash`转换（`SHA-1()`）,返回一个值
- 再对这个值做`base64`编码（`base64-encoded()`）

- 然后把他的返回值作为`Sec-WebSocket-Accept`键的值发给客户端

客户端同样会将上述`key`重新根据上述计算一遍看是否与`key`相等，如果相等则握手成功

**注意：**

加了`base64`编码后需要进行如下编译

`gcc -o server myserver.c -lssl -lcrypto`

# 状态机

利用`enum`定义状态，`websocket`有三个状态，握手状态，数据传输状态，`end`状态

初始状态为握手状态，执行完握手状态对应的代码后就调整状态为通信状态，下一次来判断的时候就会进入通信状态

# 基于`reactor`实现`websocket`协议

- 在`recv`后用`ws_request`进行解包
- 在`send`前用`ws_response`进行组包

## **`ws_request`协议解析**

使用状态机判断`websocket`此时处于什么状态，执行相应的解析

### 握手状态

利用`readline`函数读取`sec-websocket-key`后经过处理得到`accept`后封装进`sock_item`的`sec_accept`字段，循环读取的停止条件是没有遇到连续的`\r\n`

### 传输状态

进入该状态的前提是之前已经用`recv`接收到客户端的请求头信息，这个请求头信息被保存在`sock_item`的`buffer`字段（最好的保存方式是创建一个`request`的结构体，内部包含解析好的资源）

然后解析`buffer`里面的内容，在这之前需要了解数据包的格式



#### `websocket`数据包的格式

```c++
Bit    0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     |1|1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +-+-+-+-+-------+-+-------------+-------------------------------+
```

其中含义如下：

- `FIN`: `1bit`，表示是否是消息的最后一帧（`1`：是；`0`：否）。
- `RSV1、RSV2、RSV3`: 各占`1 bit`，预留给未来扩展使用。
- `opcode`: `4 bits`，表示数据类型。 `0x0`：表示附加数据帧；`0x1`：表示文本数据帧；`0x2`：表示二进制数据帧；`0x8`：表示连接关闭；`0x9`：表示心跳检测请求；`0xA`：表示心跳检测响应。
- `MASK`: `1 bit`，表示`Payload data`是否经过掩码处理。客户端发往服务器的数据必须要进行掩码处理，而服务器发往客户端的数据则不需要。
- `Payload len`: `7 bits`，表示`Payload data`部分的长度。如果`Payload len`为`0-125`，则该值为实际长度； 如果`Payload len`为`126`，则其后的`2`个字节表示实际长度； 如果`Payload len`为`127`，则其后的`8`个字节表示实际长度。
- `Extended payload length`: `16 bits`或`64 bits`，如果`Payload len`等于`126`，该部分的值为`Payload data`的实际长度；如果`Payload len`等于`127`，该部分的值为`Payload data`的实际长度。
- `Masking-key`: `4 bytes`，用于对`Payload data`进行掩码处理。只有客户端发往服务器的数据才需要经过掩码处理，并且必须随机生成。
- `Payload Data`: 实际的载荷数据。长度由`Payload len、Extended payload length`以及`MASK`三者共同决定。

### 继续接着探究传输状态

定义一个结构体，利用小端形式，构造出`websocket`头的前`2`个字节(一共`8`位)

```c++
struct ws_ophdr{
	unsigned char opcode:4,
				  rsv3:1,
				  rsv2:1,
				  rsv1:1,
				  fin:1;

	unsigned char pl_len:7,
				  mask:1;
};
```

直接将请求头的`buffer`进行强转，获取头部信息判断是否是掩码，然后进行明文处理或者是密文处理，密文处理的话则需要进行`websocket`自己规定的解码方式来解密

## 在`send`前用`ws_response`进行组包

将解析好的`sec-websocket-accept`放入回包中发送即可