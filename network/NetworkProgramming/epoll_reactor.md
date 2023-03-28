[toc]

# `epoll`理解

例子：快递员（服务器端`server`）负责寄快递与送快递，她需要到住户（客户端文件描述符`client`）那里去问，有没有快递要记或收（`IO`），这样太麻烦了，于是就在楼下建了一个蜂巢（`epoll`），快递员不需要再上门取件了。快递员只会查看有住户请求（`IO`）的蜂巢柜子，不管这个请求是寄快递还是收快递。

- `epoll_create`：相当于建立了蜂巢
- `epoll_ctl`：相当于注册一个能够使用蜂巢（能够被`epoll`检测到`io`）的用户或者删除用户

- `epoll_wait`：相当于快递员去风潮里面接收请求，可以是寄快递也可以是取快递，具体操作要看用户是请求了什么
  - 其中`timeout`参数指的是多久去取一次，`-1`则代表一直再快递柜旁边守着等待用户请求。
  - `events`相当于快递员去蜂巢里面去快递需要用一个袋子装
  - `length`代表需要多大的袋子装

# `select`与`epoll`的对比

- `epoll`的优点
  - 不许要循环遍历所有的`fd`，`select`会循环所有的`fd`{这个所有的`fd`并不是`rfds`（`fd_set`）的大小，其置为`1`的最大的索引数}，她不管里面有没有`io`，都需要我们自己去访问一遍，并不会主动告知我们
  - 每一次只会取就绪的集合（里面`fd`一定是有自己的`io`）
  - 做到了异步解耦，每个**`io`的处理**与**被处理**被解耦了（**`fd`就绪**与**`io`的事件触发**）

# **`epoll_create`为什么只需要传入一个`>0`的数**

查看源码 ` epoll_create`只需要传一个>0的数，为什么要有这个参数的存在，最早是形容一次性就绪事件的最大数量，现在使用链表实现故无需指定大小



# 为什么需要先添加`listenfd`进`epoll`中去

相当与收寄快递你要先注册蜂巢，这样你才能够对她发出请求。

（`epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);`  ）,

这一步后内核中就有一个`fd`了，就是蜂巢门口站了一个人，主要负责住户的注册蜂巢。

# 服务器`7x24`运行

靠的就是`epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);`执行之后的`while`循环，其中可以是`while 1`、`do while`、 其他`flag`条件

# 为什么下列代码执行后在客户端连接后会陷入无尽循环

```c++
// ...
epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);  

while(1){
    epoll_wait(epfd, events, EVENTS_LENGTH, -1); 
    printf("-----------------------\n");
}
// ...
```

- 没有`accept`，`listenfd`中的`io`一直无法清除，导致`listenfd`上一直有`io`，`epollwait`就会一直返回

# 服务器区分两类`io`，一类是监听的`io`，一类是客户端的`io`

- 服务端对这两类的`io`处理方式是不同的
- 监听的`io`通过`accept`来处理
- 客户端`io`通过`recv/send`来处理

# 没有处理客户端`io`会怎样?

```c++
// ... 
while(1){
    int nready = epoll_wait(epfd, events, EVENT_LENGTH, 1000);
    printf("nready = %d\n", nready);
    
    for(int i = 0; i < nready; i++){
        int clientfd = events[i].data.fd;
        if(listenfd == clientfd){  // 处理监听io
            //..
       		int connfd = accept(...);
            //...
            epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev);
        }else{ // 处理客户端io
            // 无操作
        }
    }
}
```

上述没有对客户端的`io`处理，所以后续客户端断开连接的时候`nready`一直刷`1`，再断开一个就一直刷`2`，因为没有对客户端断开进行处理，`epoll_wait`就会一直检测到`clientfd`有`io`从而一直返回给服务端

# 水平触发与边缘触发

- 水平：在`recv`时设置的`buffer_length`小于**发送数据长度**的情况，会循环的去接收，这里的循环是因为`epoll`检测为水平触发，只要`fd`上还有数据就一直触发
- 边缘：`epoll_ctl`设置设置`fd`事件为`epollet`，`fd`上的`io`只会在客户端发送数据的时候触发一次，不管后续有没有将其读完，都不会触发了，所以说如果使用`epollet`，则接收数据需要循环的将其读完

# `listenfd`用水平触发还是边缘触发

- 两者都可以用
- 一般代码都是水平触发，边缘触发时处理监听的`fd`需要先设置`listenfd`非阻塞（为了让accept在必要时返回`-1`），在处理`listenfd`时加一个`while(1){// 处理}`，然后当`accept`返回`-1`时`break`掉

# `EPOLLOUT`事件有什么用

- 判断`fd`可以写，是在`send`之前判断`IO`可写(`send`是往`send`的缓冲区中发，也可以理解是往协议栈里面发，内核中什么时候读客户端不会管)
- 不判断`EPOLLOUT`不利于业务的解析、不利于`fd`的独立、不利于连接池的连接

# `epoll_wait`检测到的`io`超过了`events`的最大大小时会如何处理

- 例如有`1000`个客户端同时有`io`，同时被加进了就绪队列，此时`epollwait`会从就绪队列中取出`128`（假设设置的`events`的大小是`128`），之后`epollwait`不会阻塞继续取出就绪队列`128`个出来

# 引入`reactor`反应堆模式s

## 什么是`reactor`?

一个服务器有很多`io`，我们将他们递交给`epoll`内核管理，一旦`io`上面有事件（可读或者可写），就触发事件（可读或者可写）对应的回调函数处理业务。每个`fd`都有一个对应的结构体，里面保存了一些必要信息（回调函数，独立的读写缓冲区）

## 为什么要有`reactor`

`reactor`在`epoll`的基础上面增加了什么，有了哪些好处

- `epoll`：他是对`io`的管理
- `reactor`：是对事件的管理，不同事件对应于不同的回调函数
- 由于`sock_item`封装，对未处理完的事件，放到一个独立（独立于其他的`fd`（客户端））的`buffer`里面
  - 好处？
    - 比如实现了 一个`httpserver`，我们的`recv`只能收`1024`，但是`get`请求有`2000`个字符，每次接收都是放在`fd`自己的`buffer`里面，不会被其他的`fd`的输入

