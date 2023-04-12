[toc]

# 一、网络`IO`的职责

## 操作`IO`

### `IO`的操作方式

- 阻塞`IO`
- 非阻塞`IO`

通过`fcntl`修改`fd`是阻塞还是非阻塞，之后所有使用这个`fd`的系统调用都会表现为非阻塞

### **阻塞与非阻塞`IO`的具体差别：**

数据在未就绪的处理

#### 阻塞`IO`在系统调用中的流程

**`read`阻塞`IO`调用流程**：先去检测内核缓冲区有无数据，如果没有数据则阻塞等待，当客户端给我们发送数据了，然后我们的网卡驱动往里面内核缓冲区填充数据，然后阻塞的系统调用才会把阻塞去掉，然后就拷贝，然后系统调用就返回

**`write`阻塞`IO`调用流程**：先去检测内核缓冲区是否可以写数据（是否是满的），如果是满的，则就阻塞，一直等到网络协议栈把内核缓冲区数据发送出去了之后他有剩余空间了才会停止阻塞，将数据写进内核缓冲区，返回实际写入内核空间的字节数

**`accept`阻塞`IO`调用流程：**检查全连接队列中是否有节点，如果没有则一直阻塞等待，如果有则取出返回分配的客户端`fd`

#### 非阻塞`IO`在系统调用中的流程

不管内核缓冲区是否可写可读，都直接返回。在必要情况会设置`errno`

### 网络编程系统调用具备检测和操作的功能

#### `accept`：

- 检测：全连接是否还有未处理的连接信息
- 操作：从全连接队列中取出连接去获取并生成客户的fd以及ip端口信息

#### `read`：

- 检测：内核缓冲区是或否有数据
- 操作：将内核缓冲区数据拷贝到用户态缓冲区来

#### `write`：

- 检测：内核缓冲区是否可以写数据（满了吗）
- 操作：将用户态缓冲区拷贝到内核态缓冲区

# 二、系统调用在调用非阻塞`IO`的具体处理

## `connect`

- 多次调用`connect`，其**返回值**和**设置的`errno`**可能是不一样的，当`errno`为`EISC-ONN`时，就告诉我们连接已经建立成功了，如果是第一次调用`connect`的时候会返回`EINPROGRESS`，告诉我们正在建立连接

## `accept`

- 如果全连接队列中为空（客户端没有与服务器进行连接），那么`accept`会返回`-1`，并且`errno`会设置为`EWOULDBLOCK`，告诉我们全连接队列为空

## `read`

- `read`调用查看`errno`：
  - `read = -1 && errno = EWOULDBLOCK`：代表读缓冲区还没有数据
  - `read = -1 && errno = EINTR`：代表可以读的时候被中断了

## `write`

- `write`调用查看`errno`：
  - 同上  不过是代表缓冲区满了

# 三、网络中连接断开的情况

## 主动关闭

- `shutdown`关闭一端
- `close`是关闭两个端（首先是回收资源，然后读写端相应的会关闭）
- **客户端**的**写端**关联**服务端**的**读端**，**服务端**的**读端**关联**客户端**的**写端**
  - 所以客户端调用`shutdown`关闭读端时，服务端对应的写端就会关闭（需要自己去判断）

## 被动关闭

- **`read` 返回 0**：告诉我们连接断开了，更准确的说是服务端的读端关闭了
- **`write`返回`-1 `&& `errno = EPIPE`**：告诉我们服务端的写端关闭了

- 为什么要区分读端关闭还是写端关闭
  - 因为有的服务器框架需要支持半关闭状态，
    - 如果是读端关闭了，服务器还可以将这个连接未发送出去的数据调用`write`发送出去，对应四次回收中对端发送`fin`包之前的时间段。

# 四、分离检测与操作

`accept`亲自检测全连接队列中是否有节点，而是使用`io`多路复技术来检测`io`是否就绪，保证当执行到`accept`时一定是`io`就绪的状态

并且`io`多路复用可以同时检测**多个**`io`的就绪状态

## 以`epoll`为例介绍`io`多路复用

### 建立连接

- `io`检测**接收连接**的处理流程

  - `socket`
  - `bind&listen`
  - 监听读事件（`epollin`）
    - `epoll`如何监听的！`epoll`会监听`listenfd`的读事件，当我们连接监听成功的时候，**全连接队列上会多一个节点，这个节点就会发送一个信号**（`EPOLLIN`）来告诉`epoll`（也可以是`select、poll`），就会触发读事件，就说明接收连接的`io`已经就绪，于是后续就调用`accept`，而现在`accept`则只会进行操作而不会进行检测，因为`io`多路复用已经做好了检测

- `io`检测**主动连接**的处理流程

  - 服务器作为客户端 连接一些服务比如`mysql`

  - 监听写事件（`epollout`）

    - 服务器调用`conncet`发送`SYN`包，此时状态是`EINPROGRESS`，如何由`io`多路复用来检测这个连接是否建立成功，`io`多路复用会检测**三次握手**的最后一次握手是否发送，**当三次握手的最后一次握手发出时会发送一个信号**，告诉`epoll`写事件触发了（所以当`epoll`监听到写事件触发的时候就说明我们的连接建立成功了）

    - 下面是一个`epoll`负责`connect`的检测的小`demo`

      ```c++
      // 设置sockfd非阻塞
      connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr));
      
      epollfd = epoll_create(EPOLL_SIZE);  // 10
      
      event.events = EPOLLOUT;
      event.data.fd = sockfd;
      ret = epoll_ctl(epollfd, EPOLL_CTL_ADD, sockfd, &event);
      
      epoll_wait(epollfd, events, EPOLL_SIZE, -1);  // -1代表阻塞
      
      if (events[0].events & EPOLLERR || events[0].events & EPOLLHUP) {
          printf("connect failed\n");
          exit(EXIT_FAILURE);
      }
      
      else if (events[0].events & EPOLLOUT) {
          printf("connect success\n");
      
          // 发送一条消息
          ret = write(sockfd, send_buf, strlen(send_buf));
          if (ret == -1) {
              perror("write");
              exit(EXIT_FAILURE);
          }
      }
      ```

### 连接断开

通过判断`event[i].events`来对客户单断开连接进行检测

- `EPOLLRDHUP`：表示服务器读端关闭
- `EPOLLHUP`：表示服务器读写端都关闭

### 消息到达

- 客户端`fd`触发`EPOLLIN`（检测），然后调用`read`（操作）

### 消息发送

- 客户                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  端`fd`触发`EPOLLOUT`（检测），然后调用`write/send`（操作）

## `epoll`详解

`epoll_create`会创建一个红黑树和一个双端队列

`epoll_ctl`是对红黑树增删改，同时会跟网卡驱动建立回调关系，响应事件触发时会调用回调函数，这个回调函数会将触发的事件拷贝到`rdlist`双向链表中；

调用`epoll_wait`将会把`rdlist`中就绪事件拷贝到用户态中（拷贝出来后会清空内核态就绪队列的事件）

### 介绍`epoll_wait`的四个参数接口

- 参数

  - `epoll`文件描述符

  - 用户空间的数组，用来接收内核空间中就绪队列的东西

  - 预期要拷贝多少事件

  - 定时（以毫秒为单位）

    - `epoll`是一种同步的`io`，没有阻塞与非阻塞之说，但是可以通过`timeout`来控制函数的阻塞，

      设置为`-1`则会表现为阻塞（就绪队列没有事件就一直阻塞），

      设置为`0`会表现为非阻塞（就绪队列）

- 返回

  - 返回实际取出来多少事件

# 五、引入`Reactor`

是将对`io`的操作转化为对事件的处理

## 什么是`reactor`?

一个服务器有很多`io`，我们将他们递交给`epoll`内核管理，一旦`io`上面有事件（可读或者可写），就触发事件（可读或者可写）对应的回调函数处理业务。每个`fd`都有一个对应的结构体，里面保存了一些必要信息（回调函数，独立的读写缓冲区）

## 为什么要有`reactor`

`reactor`在`epoll`的基础上面增加了什么，有了哪些好处

- `epoll`：他是对`io`的管理
- `reactor`：是对事件的管理，不同事件对应于不同的回调函数
- 由于`sock_item`封装，对未处理完的事件，放到一个独立（独立于其他的`fd`（客户端））的`buffer`里面
  - 好处？
    - 比如实现了 一个`httpserver`，我们的`recv`只能收`1024`，但是`get`请求有`2000`个字符，每次接收都是放在`fd`自己的`buffer`里面，不会被其他的`fd`的输入所影响

## `reactor`为什么搭配非阻塞？

为什么`io`多路复用帮我们检测好了`io`就绪，为什么还要使用非阻塞`io`，考虑一下三种情况

- 多线程环境

  - 多个线程使用同一`epoll`监听同一`fd`

    当这个`fd`上有`io`的时候，就绪队列的节点会向每一个线程的`epoll`通知这里有新的连接建立。

    如果使用阻塞的`io`，则当一个线程的`epoll_wait`将它取出来后其他线程的`epoll_wait`就会阻塞在那儿

    **惊群！：**有一个海王，有很多个女朋友，每一个都叫老婆，有一次他同时碰到了这些女朋友，他交了医生老婆。结果他们都答应了

- 边缘触发情况下

  - 调用`epollwait`，在客户端`fd`发生`io`事件的时候取出`rdlist`中的对应节点，后续不管`readbuf`还有没有数据都不会自动的把该`fd`放入`rdlist`中（而`lt`是会在`readbuf`还有数据时自动的放入`rdlist`，再让`epollwait`触发）

    在边缘触发下，`readbuf`必须一次性将数据读空，不然会出现类似`tcp`的粘包，如果`fd`设置为阻塞，则`readbuf`在已经读空的情况下不会返回而是继续阻塞，所以要设置为非阻塞来判断`read`读完

- `select bug`

  - 当某个socket接收缓冲区有新数据**分节**到达，然后select报告这个socket描述符可读，但随后，协议栈检查到这个新分节检验和错误后丢弃了这个分节，这时候调用read无数据可读，如果socket没有设置非阻塞，那么read就会阻塞当前线程

# 六、`reactor`在中间件的运用

## 1、`redis`

使用单`reactor`

- 具体环境

  - `kv`、数据结构、内存数据库

    通过`key`来查`value`，`value`可以支持种数据结构，这些数据的操作都是在内存上操作

  - 命令处理是单线程的

- `redis`为什么要使用单`reactor`

  - 只能采用单reactor，因为核心业务逻辑是单线程的。
  - 去操作数据结构的时候很简单，操作命令时间复杂度比较低

- `redis`是怎么处理的`reactor`

  大致流程

  - 创建l`istenfd`
  - `bind listen`
  - `listenfd`注册读时间
  - 读事件触发回调`accept`
  - `clientfd`注册读事件
  - 读事件触发

- `redis`针对`reactor`做了哪些优化

  - 开启多线程，将读数据和解协议绑定在一起放在另一个线程，将组包和写数据放在另一个线程处理，中间的业务逻辑还是放在主线程操作

## 2、`memcache`

多`reactor`

- 针对reactor做了哪些优化

主线程中有一个专门用于接收连接的`reactor`，用多少个连接也就有多少个子`reactor`，来一个连接创建一个线程，线程与线程的交互是使用`pipe`来交流的

## 3、`nginx`

多`reactor`

多进程，每个进程有自己的`epfd`，监听同一`listenfd`，用户层处理惊群是为了可以自定义负载均衡