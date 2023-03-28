[toc]

# `reactor`的实现

## 为什么需要`sock_item`这一数据结构

- 每一个`fd`都有一个自己的`sock_item`

- `fd`有自己的`rbuffer`、`wbuffer`、`wlength`(发送数据的长度，已经发送了多少)、`rlength`、`events`、`callback`（触发事件对应的回调函数）
- 每实现一个连接，其`clientfd`都会对应的有一个`sock_item`

## `reactor`结构体

- 保存了都会用到的数据例如`epfd`（`epoll_create`的返回值）
- 保存了`sock_item`的存储形式

## 关于`sock_item`的存储形式

- 实现一：固定长度的数组

  ```c++
  struct reactor {
  	int epfd;
  	struct sock_item* items;
  };
  ```

  直接为`items`分配固定长度的空间，利用`fd`作为索引来初始化和查找对应的`sock_item`实现简单，但是如果连接数超过了其数组长度就会出现段错误，索引越界。

- 实现二：链表实现

  ```c++
  struct eventblock{
      struct sock_item *items;// 长度是1024的数组
      struct eventblock* next;
  };
  
  struct reactor {
  	int epfd;
      int blkcnt;
  	struct eventblock* evblk;
  };
  ```

  - `reactor`结构体有一个用来标识链表块数的变量
  - 该链表可以动态增加长度
  - 来一个`fd`，则`fd/1024`代表链表的某一节点，`fd%1024`代表某一节点的某一具体点

## 测试高并发时出现第一个问题

连接数到达65535时出现请求地址不足

### 引入五元组概念

通过（`source ip, source port, destination ip,  destination port, protocal`）可以确定一个连接，在测试中有四个参数是确定的，只有一个`sorce port`也就是源客户端地址在改变，从`0`到`65535`，用完了就无法请求了

- 解决端口不够用

  - 绑定不同的客户端`ip`地址，用多个进程去启动
  - 绑定不同的服务端的`ip`地址（服务端有多个网卡）
  - 绑定不同的服务端的端口（服务端开放多个端口来监听）

- 使用绑定不同的服务端的端口来解决，服务端创建大量的监听的`fd`并且添加进`epoll`中，用一个数组来存储`fd`，在`epoll_wait`后使用自建函数`is_listened`，来看是否是有客户端连接进来了

  ```c++
  int is_listenfd(int * fds,int connfd){
      int i = 0;
      for(i = 0; i < PORT_COUNT; i++){
          if(fds[i] == connfd){
              return 1;
          }
      }
      return 0;
  }
  ```

# 什么是并发？

- 服务器同时能够承载客户端连接的数量

## 一个能够承载百万并发的服务器需要进行的测试

- 首先需要测试网络连接过`100w`，每秒接入量
- 然后是测试每个业务的`qbs`
- 断开连接的时候 测试
- 每一秒能够传输的网络数据包

## 真正做到`100w`连接是依靠什么

- 只要你使用到了`epoll`，那么其服务器并发量则一定会到达`100w`

- 面试时如何说：

  面试官问：做的web服务器并发量是多少

  回：（网络连接数怎么测的，数据量是多少）`100w`的连接数量，我测过，用三台客户端加一台服务器，并发的连接梁能够到达百万，其他的`qbs`需要分开测

  回：（针对业务，测的什么业务）《以问卷调查为例》我测了一个登录，然后测了`post`请求，没有问题

  回：（针对于断开连接）对于三个客户端，每一个跑`34`万，同时三个客户端同时`ctrl+c`同时宕掉，此时服务器不会崩溃

# 网络数据包如何测：

- 使用`iperf`

# 网络连接的并发量与单线程多线程没有关系

- 多线程解决的是每秒接入量（多个线程`accept`）与业务处理能力



> 所有的网络框架并发能够达到多少都是因为底层使用的`epoll`，跟框架本身没有关系