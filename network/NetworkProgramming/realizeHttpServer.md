# 你的`web`服务器需说什么

- 测试你的网络并发量

# 什么是开发

基于已有的框架做成自动的处理，然后测试

# 当一个`io`有事件的时候，回调函数做什么

- 例如客户端发来`hello`，调用`read_cb`（`callback`）
  - 其中包含
    - 读数据（`recv`读到`rbuffer`）
    - 解析数据（`parser()`）
    - 判断有没有读完（重新设置事件（`event_register()`））
      - 没读完，下一次继续读
      - 读完了，下一次就判断可写（注册`epollout`）
- 加入客户端回复发送“我好 我很好！”，调用`write_cb()`
  - 其中包含
    - 将要发送的数据拷贝到`wbuffer`中（重要，不能在`read`读到`rbuffer`后直接`memcpy`到`wbuffer`中）
    - 调用`write()`
    - 判断是否写完
      - 没写完，继续写
      - 写完，下一次判断读

> `event_register(fd, event, callback)`：为`fd`注册事件和设置对应的回调函数

# 为什么要设置为回调不设置为全局函数

每一个`fd`的`io`要对应一个自己的回调函数，并且`epoll`监听的`io`不止网络`io`，还可以是磁盘`io`，定时器，所以每个`fd`都与之对应的有自己的回调函数

# 实现`http`的服务器

- `recv`到`http`请求后立即解析第一行获取文件位置与请求方法设置到`sock_item`里面

- 设置`reactor`结构体中`fd`对应`sock_item`的回调函数为`send_cb`，并设置对应事件为`EPOLLOUT`

- 接着判断到`fd`可写后执行`sendcb`回调函数，在`send`之前进行打包，

  根据`fd`对应的`sock_item`的`method`来判断执行什么打包操作，

  例如`get method`，就进入`getmethod`，将返回信息打包到`fd`对应的`sock_item`的`wbuffer`和`wlength`中，之后就开始`send`然后利用`sendfile`发送**数据**，重新注册`fd`事件`epollin`，然后设置`recvcb`回调
