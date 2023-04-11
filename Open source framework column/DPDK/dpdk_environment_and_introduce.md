[toc]

# 零、从`0`开始配置`dpdk`环境的虚拟机

- **外部配置**

  - **配置网卡**

    - 初始网卡选择桥接模式作为`dpdk`运行的网卡，添加第二张网卡`NAT`模式用于`ssh`连接

    - **内存与`CPU`核数选择**
      - 内存选择`8g`以上，核数选择`8`核以上

  - **修改源文件让网卡支持多队列**

    - 修改虚拟机的`vmx`文件，修改`ethernet0.virtualDev = "e1000"`的值并添加一行

    ```bash
    ethernet0.virtualDev = "vmxnet3"
    ethernet0.wakeOnPcktRcv = "TRUE"
    ```

- **内部配置**

  - **配置网卡**

    - 在安装``ubunt_userver`时，存在多个网卡时需要选择默认网关网卡，这里选择用于`ssh`连接网卡`ens33`

    - `ens160`是多队列网卡，配置其静态`ip`地址，在`/etc/network/interface`追加下面行

      ```bash
      auto ens160
      iface ens160 inet static
      address 10.17.43.34
      netmask 255.255.0.0
      ```

      `ip`需要根据主机的`ip`地址的网段和掩码一致，因为这是桥接模式，第二章`nat`网卡直接使用`dhcp`自动获取`ip`网关`dns`服务器即可

      使用`route -n`可以查看默认网关

  - **修改系统启动参数**

    - 修改`etc/default/grub`文件

      ```bash
      GRUB_CMDLINE_LINUX_DEFAULT=""
      GRUB_CMDLINE_LINUX=""
      >>>>修改后
      GRUB_CMDLINE_LINUX_DEFAULT="quiet"
      GRUB_CMDLINE_LINUX="find_preseed=/preseed.cfg noprompt net.ifnames=0 biosdevname=0 default_hugepages=1G hugepagesz=2M hugepages=1024 isolCPUs=0-2"
      ```

      修改后`update-grub`后`reboot`会发现网卡名字变为了`eth*`，此时修改`/etc/network/interfaces`文件

      - `eth0`对应第一张也就是桥接模式的网卡，就替换`ens160`
      - `eth1`替换`ens33`

# 一、`dpdk`的编译`usertool/dpdk-setup.sh`

- 下载`dpdk-19.08.2`，在`dpdk-stable-19.08.2`文件夹下执行`./usertools/dpdk-setup.sh`

- 选择不带`app`的`x86_64-native-linux-gcc[39]`，因为它可以改源码

- 设置运行时环境变量

  ```bash
  export RTE_SDK=/home/ljq/share/module/dpdk-stable-19.08.2/
  ```

- 设置运行时环境的目标

  ```c++
  export RTE_TARGET=x86_64-native-linux-gcc
  ```

- `./usertools/dpdk-setup.sh`选择`[43] or [44]`插入`IGB_UIO`模块` or`插入`V`

- 选择`[45]`插入`KNI`模块

- 选择`[46] or [47]`巨页的配置

  - `46`代表采用`no numa`方式管理多内存编号，`47`代表采用`numa`方式管理多内存编号

    `numa`表示不是同一的编码方式，每个内存块都从索引`0`开始，`no-numa`表示同一编码方式，内存块之间的索引是连着的

  - 写`512`来设置`1g`的巨页

- 选择`[49] or [50]`

  - 把对应网卡的绑定设备绑定在哪个模块上`[49]->UIO`  `[50]->VFIO`

  - 两者选一个 我选`49`

    - 接着输入`PCI`地址来绑定`IGB UIO`

      找到`eth0`对应的行，输入其前面的数字字符串 我的是`0000:03:00.0`

      回车后会警告：

      `Warning: routing table indicates that interface 0000:03:00.0 is active. Not modifying
      OK`，意思是还在工作中 没有修改成功

      我们要给他`down`掉，就是不让对应的`NIC`接收数据，让`dpdk`去绑定这个网卡去截获，网卡的数据就不会往内核里面走了

      `ifconfig eth0 down`

      再次选择`49`，输入`0000:03:00.0`后绑定成功

      此时`ifconfig`是不存在`eth0`，也就是说明此时`eth0`对应的`NIC`不会再去接管他了，之后所有数据就是走到`dpdk`里面了

> 编译前需要安装的
>
> `sudo apt-get install libnuma-dev libpcap-dev make`
>
> 
>
> `dpdk`接管网卡的两种方式
>
> - 通用型`IO`
> - 虚拟`IO`
>
>
> 注意有可能45出会出现错误，此时使用`grep CONFIG_RTE_KNI /boot/config-$(uname -r)`查看`CONFIG_RTE_KNI`是否被设置为了`m`或者`y`，如果没有则需要手动添加进去

# 二、`dpdk`需要什么配置来支持

## 1.多队列网卡

`dpdk`要指定用哪一个队列来接收数据

## 2.巨页

# 三、解析接收网络数据的过程经历了什么

## 1.物理网卡

- 物理网卡接收（`vmware`的网络适配器就是一个网卡，是`vmware`模拟的）
- 物理网卡是光信号或者电信号与数字信号的相互转换

## 2.`NIC`

- 接收后每个网卡都会配对的有一个网络适配器（`NIC`）
  - 问题，`NIC`什么数据结构存？
  - 使用`ifconfig`就是查看`NIC`的数量以及相关信息
  - 网卡每来一帧数据`NIC`都会将它组织成`sk_buff`（其中有很多指针，指向数据包中的各个层的头部），发送给协议栈，协议栈就解析其各个头

## 3.内核协议栈

- `NIC`把数据抛给**内核协议栈**，协议栈解析其`sk_buff`中指向的各个头，注意这个协议栈是所有网卡共用的
  - 协议栈与`NIC`驱动的数据交互是怎样的
    - 网卡每来一帧数据`NIC`都会将它组织成`sk_buff`（其中有很多指针，指向数据包中的各个层的头部），发送给协议栈，协议栈就解析其各个头

## 4.标准接口层`Posix API`

- 标准接口层调用各种网络系统调用

## 5. 应用层

应用层就接收到了...

## 上述过程发生的拷贝

1. 网卡数据拷贝到`NIC`，组织出`sk_buff`

2. `APP`调用`recv..`将数据从内核态拷贝到用户态
3. 上述操作需要`CPU`的参与

# 四、`DPDK`介绍

## 基于上述接收网络数据流程`dpdk`做的事

- 把网卡通过一种**映射**的方式，把网卡接收数据的存储空间映射到用户态的内存空间中，

  这其中**没有拷贝**，是通过`DMA`的方式，直接网络适配器与硬盘驱动器与内存交互

- **`dpdk`主要做的事**：把网卡的数据快速转移到用户态内存里

## `dpdk`如何组织映射的数据

### `Huge Page`

- 正常的内存操作，一个页`4k`，假设一秒钟来了`1`个`G`的数据，那么`4k`的页划分的包就为`256x1024`，太多了，于是`dpdk`采用**巨页**

## `dpdk`优化了什么

对`thread`做`CPU`的亲缘性

- 某一线程只在同一个`CPU`上运行

## `KNI`机制

用`dpdk`只想处理一种协议，其他协议还是通过内核协议栈来处理

- `dpdk`提供了一种方式，将不需要的数据写入到内核协议栈，内核网络接口(`Kernel Network Interface<kni>`)

## `DPDK`的技术边界

`dpdk`处理数据时是跨过了`NIC`这一层，在`NIC`接收数据前就截获了数据，与`NIC`平级，改变了数据流程

### `DPDK`能做什么

1. 路由器、`SDN`网络定制开发
2. 网络协议栈
3. 防火墙，防`ddos`攻击
   1. 接收数据，检测异常数据
4. `VPN`
   1. 直接往原始数据中加入`vxline`或加上隧道技术再从网卡中发送出去

# 五、`DPDK`该怎么学

**环境搭建完后该做的事**

- `coding`，代码可控
  - 实现一个协议栈(`eth，ip，arp，icmp，tcp，udp`)
  - `posix api, epoll`的实现
- 协议栈是与dpdk相生相辅
- 行业的应用
  - 三层网络层的`vpp`
  - 二层传输层考虑`ovs`
  - 负载均衡`dpvs`
  - 发包工具`pktgen`
- 做`1-2`款上线的产品。。。

# 六、`coding`

## `rte_eal_init(argc, argv)`初始化`dpdk`应用

## `rte_pktmbuf_pool_create`创建

```c++
rte_pktmbuf_pool_create(
	const char * name,
    unsigned int n,  // 初始化mbufpoll池能放多少个mbuf
    unsigned int cache_size,  // 0
    uint16_t priv_size,  // 0
    unit16_t data_room_size,
    int socket_id  // 设置为一个全局的一个变量per_lcore_socket_id
);
// 返回一个rte_mempoll结构体
```

- 名字中的`mbuf`就是内核中的`sk_buf`
- `name`：`Packet Buffer` 池的名称，可以是任何字符串，建议使用有意义的名称。
- `n`：`Packet Buffer` 池中 `Packet Buffer` 的数量，建议设置为大于等于` 1024`。
- `cache_size`：`Packet Buffer `池中的本地缓存大小，可以根据使用场景进行调整，建议设置为` Packet Buffer` 数量的` 1/8 `左右。
- `priv_size`：`Packet Buffer `中每个 `Packet Buffer` 的私有数据大小，如果不需要私有数据，则可以设置为` 0`。
- `data_room_size`：`Packet Buffer `中数据区域的大小，即可以存储数据包的空间大小，建议设置为 `2048` 或更大。
- `socket_id`：用于分配内存的 `NUMA `节点编号，建议设置为套接字绑定的` CPU `所在的` NUMA `节点编号。

## 对网口的配置

 在`TCP/IP`网络中，当一个网络设备接收到来自其他设备的数据包时，就会触发`RX`操作，将数据包解析并交给上层协议或应用程序 

**tx与rx是什么分别用在什么地方**

`txqueue`主要用于管理应用程序待发送的数据包，当应用程序需要向网络中发送数据时，首先将其封装成一个数据包，并添加到`txqueue`中。

`rxqueue`主要用于管理已经从网络中接收到的数据包，当网络中有数据包到达时，`DPDK`会将其添加到`rxqueue`中，应用程序可以从`rxqueue`中取出数据包并进行进一步的处理。 

```C++
	// setup
	uint16_t nb_rx_queues = 1;
	uint16_t nb_tx_queues = 0;
	const struct rte_eth_conf port_conf_default = {
		.rxmode = {.max_rx_pkt_len = RTE_ETHER_MAX_LEN }
	};
	
	rte_eth_dev_configure(gDpdkPortId, 
                          nb_rx_queues, 
                          nb_tx_queues, 
                          &port_conf_default); 

	// 设置第gDpdkPortId个绑定网口的信息，使用了索引为0对应的rx队列，设置了rx队列的大小为128，使用mbuf_pool来存储接收队列缓存
	rte_eth_rx_queue_setup(gDpdkPortId,  // 设置第gDpdkPortId个绑定网口的信息
                           0,   		 // 使用了索引为0对应的rx队列
                           128,			 // 设置了rx队列的大小为128
                           rte_eth_dev_socket_id(gDpdkPortId), 
                           NULL, 
                           mbuf_pool);	 // 使用mbuf_pool来存储接收队列缓存
	// 
	rte_eth_dev_start(gDpdkPortId);

	// disable
	//rte_eth_promiscuous_enable(gDpdkPortId); //

```

- `rte_eth_dev_configure`**API**介绍

  ```c++
  ret_eth_dev_configure(
  	unit16_t port_id,  //网口 nic 的id是什么
      unit16_t nb_rx_q,  // rx队列 ,对应索引  0表示使用第一个
      unit16_t nb_tx_q,  
      const struct rte_eth_conf* dev_conf  // 配置信息  每个包的长度
  );
  ```

  - `port_id`：以太网设备的端口 ID。
  - `nb_rx_queue`：用于接收数据包的 RX 队列数量。
  - `nb_tx_queue`：用于发送数据包的 TX 队列数量。
  - `eth_conf`：一个结构体，包含了以太网设备的配置信息，如 `CRC `校验、硬件 `RSS `散列、速率控制，以及杂项配置等。

- `rte_eth_rx_queue_setup` `API`介绍

  ```c++
  int rte_eth_rx_queue_setup(
      uint16_t port_id,      // 要绑定的网口索引号 0代表使用第一个网口
      uint16_t rx_queue_id,  // 要绑定的队列的索引，0代表绑定第一个rx队列
  	uint16_t nb_rx_desc,   // 指定队列可以装多少mbuf
      unsigned int socket_id,
  	const struct rte_eth_rxconf *rx_conf,
  	struct rte_mempool *mb_pool
  );
  ```

  - `port_id`：以太网设备的端口 ID。
  - `rx_queue_id`：将要创建的接收队列的队列号。
  - `nb_rx_desc`：用于网卡接收队列的缓存区描述符数量，可以简单理解为接收队列的缓存容量。（是网卡上面的`rx_queue_id`对应`id`的接收队列的大小，前面`mbuf_pool`内存池的大小就是用来接收这个队列中的节点，所以这个内存池的大小肯定要比`rx`队列大小大）
  - `socket_id`：用于分配和管理内存资源的` NUMA` 节点` ID`，一般使用 `rte_socket_id()` 函数获取。
  - `rx_conf`：用于指定接收队列的配置信息，如 `RSS` 散列、哈希过滤器和 `Ptype` 解析等特性。如果不需要使用这些特性，可以将该参数设置为 `NULL`。
  - `mb_pool`：用于存储接收队列缓存的内存池指针。

## 接受数据

```c++
    while(1){
        //..后面的代码都在这个里面
    }
```

### 使用`API`接受数据

```c++
		struct rte_mbuf *mbufs[MBUF_SIZE];  // 32
		unsigned num_recvd = rte_eth_rx_burst(
            gDpdkPortId,   // 绑定的网口
            0,             // rx队列索引
            mbufs, 
            MBUF_SIZE  // 设置的32
        );
		if (num_recvd > MBUF_SIZE) {
			rte_exit(EXIT_FAILURE, "rte_eth_rx_burst Error\n");
		}
```

- **`rte_eth_rx_burst`**

  ```c++
  uint16_t rte_eth_rx_burst(uint16_t port_id, 
                            uint16_t queue_id,
                            struct rte_mbuf **rx_pkts,  // 传出参数
                            const uint16_t nb_pkts
                         	  );
  // 不需要考虑释放  在内存池中直接将对应id取出来直接用
  ```

  - `port_id`：以太网设备的端口 `ID`。
  - `queue_id`：用于接收数据包的 `RX` 队列` ID`。
  - `rx_pkts`：指向 `rte_mbuf` 结构体指针的数组，用于存储接收到的数据包。
  - `nb_pkts`：要读取的最大数据包数量。

### 循环遍历解析所有接受到的包

```c++
unsigned i = 0;
for (i = 0;i < num_recvd;i ++) {
    // 宏函数
    struct rte_ether_hdr *ehdr = rte_pktmbuf_mtod(
        mbufs[i], 
        struct rte_ether_hdr *
    );
    if (ehdr->ether_type != rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)) {
        continue;
    }

    struct rte_ipv4_hdr *iphdr = rte_pktmbuf_mtod_offset(
        mbufs[i], 
        struct rte_ipv4_hdr *, 
        sizeof(struct rte_ether_hdr)
    );

    if (iphdr->next_proto_id == IPPROTO_UDP) {

        struct rte_udp_hdr* udphdr = (struct rte_udp_hdr*)(iphdr + 1);  // 加上IP头得到udp头
        uint16_t length = udphdr->dgram_len;
        *((char*) udphdr + length - 1) = '\0';
        printf("udp: %s\n",  (char*)(udphdr+1));
    }
}
```

- `rte_pktmbuf_mtod` 是一个宏，而不是一个函数。`在 DPDK` 的 `rte_mbuf.h` 文件中，可以看到 `rte_pktmbuf_mtod` 的定义如下：

  ```c
  #define rte_pktmbuf_mtod(m, t)  ((t)((char *)((m)->buf_addr) + (m)->data_off))
  ```

  这个宏的实现使用了强制类型转换，其目的是将缓冲区中数据的地址转换为用户指定的数据类型 `t` 的指针，从而方便用户访问和处理接收到的数据包。需要注意的是，在使用 `rte_pktmbuf_mtod` 宏时，应当确保传入的参数合法，并且传入的数据类型和数据包的格式相符，否则可能会导致程序错误。



> 设置`arp`静态映射
>
> 用arp静态映射 就可以用网络调试工具调试了

```c++

arp -s 10.20.181.170 00-0C-29-AF-77-5C

arp -d 10.20.181.170

# The ARP entry addition failed: Access is denied

netsh i i show in  # 查看网络接口的索引号

netsh -c i i add neighbors 18 10.20.181.170 00-0C-29-AF-77-5C

netsh -c "i i" delete neighbors 1
```



每次重启后需要从设置运行时环境变量开始