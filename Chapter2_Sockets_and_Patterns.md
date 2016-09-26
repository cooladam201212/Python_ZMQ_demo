# Sockets and Patterns

第一章中，简单举例了ZeroMQ的一些主要的模式：请求-应答、订阅-发布和管道(pipline)模式，在本章中，我们将要亲身学习如何在真实的程序中使用这些工具。

本章摘要：
- 如何创建和使用ZeroMQ sockets
- 如何在sockets上发送和接收信息
- 如何利用ZeroMQ的异步IO模型构建自己的程序
- 如何在单线程中处理多个sockets
- 如何处理中断信号（类似Ctrl+C）
- 如何优雅地关闭ZeroMQ程序
- 如何检查ZeroMQ是否内存泄漏
- 如何收发多份信息
- 如何在网络上转发信息
- 如何构建一个简单的消息队列broker
- 如何用ZeroMQ构建多线程程序
- 如何利用ZeroMQ在多线程中传递信号
- 如何利用ZeroMQ协调网络节点
- 如何在pub-sub模式中创建和使用消息信封
- 利用HWM(high-water mark)防止内存溢出


## The Socket API

ZeroMQ提供了一个熟悉的基于socket的API，隐藏了一堆的消息处理引擎。Sockets已经成为网络编程的事实标准，ZeroMQ借鉴sockets提出了一些新的概念。

类似于BSD sockets，Sockets生命周期分为四个阶段:

- 创建和销毁sockets，一起构成了sockets生命周期的循环（zmq_socket(), zmq_close()）
- 设置或读取选项参数配置sockets（zmq_setsockopt(), zmq_getsockopt()）
- 创建ZeroMQ连接，将sockets加入网络拓扑中（zmq_bind(), zmq_connect()）
- 接收和发送消息（zmq_msg_send(), zmq_msg_recv()）

> Note: sockets是void类型的指针，messages是结构体，在C语言中，消息的传参是传地址，例如`zmq_msg_send`和`zmq_msg_recv`，记住：“在ZeroMQ中，所有的sockets归你所有，而消息归代码所有”。

创建、销毁和配置sockets可以在任意期待的对象上操作，但是切记，ZeroMQ是异步的、弹性的，这对如何将sockets插入到网络拓扑中和后期如何使用它都一定影响。


## Plugging Sockets into the Topology

如果要在两个节点之间建立连接，可以在一个节点上调用zmq_bind()，并在另一个节点上调用zmq_connect()，根据经验，调用zmq_bind()的节点被称做服务端，设置一个可达的网络地址；调用zmq_connect()的节点被称做客户端，并不暴露网络地址。这个过程称做“将socket绑定到端点”和“连接socket到端点”，端点就是网络地址。

ZeroMQ和传统的TCP连接有所不同，主要的差异有以下几点：

- 支持多种传输方式（inproc, ipc, tcp, pgm, or epgm），分别参考：[zmq_inproc()](http://api.zeromq.org/3-2:zmq_inproc)， [zmq_ipc()](http://api.zeromq.org/3-2:zmq_ipc)， [zmq_pgm()](http://api.zeromq.org/3-2:zmq_pgm)， [zmq_epgm()](http://api.zeromq.org/3-2:zmq_epgm)
- 一个socket可以有多个上行和下行连接
- 不提供zmq_accept()函数，当sockets绑定到端点后自动接受连接状态
- 网络连接本身隐藏在后端，ZeroMQ实现了断开自动重连
- 代码程序不能直接操作这些连接，他们被封装在socket之下

许多体系遵循某种类型的客户端/服务端模式，服务端组件大部分是静态的，而客户端组件大部分是动态的，有时需要解决的问题是服务端对客户端可见，但不一定反之亦然。所以在大部分情况下，角色是很明显的。在一些不常见的网络架构中，也取决于你使用的sockets的类型，后续讨论。

在传统的网络模式中，必须要在客户端启动之前先启动服务端，而在ZMQ中，即使服务端还没有调用zmq_bind()函数，只要客户端调用了zmq_connect()函数，这个连接就已经存在了，如果服务端在消息队列溢出或客户端阻塞之前启动，那么消息就开始在ZMQ中传输。

一个服务端节点可以使用一个socket绑定多个端点，即可以接收来自多个端点的连接。

```
zmq_bind (socket, "tcp://*:5555");
zmq_bind (socket, "tcp://*:9999");
zmq_bind (socket, "inproc://somename");
```

大部分传输模式中，不能两次绑定同一端点，不像例子中的UDP。但是在IPC传输模式中，允许在两个进程中绑定同一个端点，作为进程崩溃后恢复。

虽然ZeroMQ在服务端和客户端的定义上有所保留，但是一般的设计模式认为，服务端是作为拓扑图静态部分绑定到一个或多个端点上；客户端就是作为动态部分绑定到这些端点上。

Sockets也有类型，类型定义了sockets的语义，这些类型的组合定义了消息的流入和流出的规则。

可以采用不同的方式连接sockets,构成了ZeroMQ消息队列系统的基本能力，在这个基础上，才有后续讲到的proxies，但本质上，用ZeroMQ定义网络架构就像孩子拼凑建筑玩具一样的。


## Sending and Receiving Messages

使用zmq_msg_send()和zmq_msg_recv()函数发送和接收消息。ZeroMQ的IO模型和传统的TCP模型有很大不同。

### Figure 9 - TCP sockets are 1 to 1

![](https://github.com/imatix/zguide/raw/master/images/fig9.png)

对比一下ZeroMQ sockets和TCP sockets在处理数据时的不同点：

- ZeroMQ sockets就像UDP一样传输数据，而不是像TCP以字节流的方式。ZeroMQ消息是一个指定数据长度的数据，这种优化能提高性能但有点棘手。
- ZeroMQ在后台线程中处理IO操作，即：不管程序多忙，消息都能从本地input队列接收，并从output队列发出。
- ZeroMQ根据socket类型，内置支持一对多的特性。

zmq_send()函数并没有真正的从socket发送消息到连接中，而是将消息入队列中，IO线程可以异步的将消息发出。在异常抛出之前不会阻塞，所以在zmq_send()返回之前，消息并不一定已经发送了。


## Unicast Transports

ZeroMQ提供一系列的单播传输（inproc, ipc, tcp）和多播（epgm, pgm），在理解fan-out会使一对多的单播不可能之前，不要轻易使用他。

- tcp：大部分情况下，使用的是断开的TCP连接，因为在ZeroMQ的tcp传输中，并没有要求在连接之前端点预先存在，客户端和服务端可以在后续的任意时间连接上来，对应用程序是透明的。

- ipc：与tcp类似，也是断开连接式的，但是有一个限制条件：在windows下无法工作。方便起见，端点采用“.ipc”作为扩展名，避免和其他文件出现潜在的冲突。在linux系统中，使用ipc端点之前必须要有访问权限到指定目录，否则，在不同用户ID之间运行的进程无法共享它，同时必须确认所有的进程有访问该文件的权限，例如在同一工作目录下运行。

- inproc: 一个连接的信号传输，比tcp和ipc都要快很多，与前两种对比有一个特殊的限制：任何客户端确认连接之前，服务器必须确认一个绑定。这个可能在未来的某个版本会修复。


## ZeroMQ is Not a Neutral Carrier

新手常见的问题是，如何用ZeroMQ创建一个HTTP服务器，答案是，如果你会用常规的sockets承载http请求和应答，那么你同样也可以用ZeroMQ创建一个，而且更快更好。

ZeroMQ不是一个中性的承载者，他强加一个框架到使用的传输协议中，这个框架和现存的协议并不兼容，并且更趋向于使用自己的框架。下面对比HTTP请求和ZeroMQ请求，都是基于TCP/IP。

### Figure 10 - HTTP on the Wire

![](https://github.com/imatix/zguide/raw/master/images/fig10.png)

该HTTP请求使用CR-LF作为简单的框架定界符，然而ZeroMQ使用长度定界符。所以可以用ZeroMQ写一个类似的HTTP协议，例如使用request-reply模式的socket实现。但他并不是HTTP。

### Figure 11 - ZeroMQ on the Wire

![](https://github.com/imatix/zguide/raw/master/images/fig11.png)

从v3.3开始，ZeroMQ有一个`ZMQ_ROUTER_RAW`选项允许不用ZeroMQ框架读写数据。可以用它来读写正确的HTTP请求和应答，Hardeep Singh共享了这个修改，才能通过通过他的ZeroMQ程序连接到telnet服务端，在作者写文档的时候，这个还是试验版，可能在下一个补丁版本会推出，这也展示了ZeroMQ如何演变并解决新的问题。


## I/O Threads

我们说ZeroMQ在后台线程中处理IO，仅一个IO线程足以应付所有的情况，除非极端的情况下。当创建一个新的context时，即开启一个新的IO线程。通用的准则是，*允许每秒千兆字节一个线程*。增加IO线程数，通过在创建sockets之前调用zmq_ctx_set()函数。

```
int io_threads = 4;
void *context = zmq_ctx_new ();
zmq_ctx_set (context, ZMQ_IO_THREADS, io_threads);
assert (zmq_ctx_get (context, ZMQ_IO_THREADS) == io_threads);
```

我们见证过一个socket可以同时处理数十，甚至数千个连接，这有一个根本性的影响就是你如何写这个程序。传统的网络程序中，可能一个或者一个线程处理每一个连接，一个进程或者一个线程处理一个socket，ZeroMQ将使你放弃整个架构而进入一个单进程，并在需要的情况下做扩展。

如果只是用ZeroMQ做线程内通信，也可以设置IO线程为零，这不是一个显著的优化，更多的是好奇。


## Messaging Patterns

下面介绍基于ZeroMQ socket API消息模式，对于有过企业级消息系统或者UDP背景知识的来说，他们依稀有些相似，但是对于新人来说还是很吃惊的，他们已经习惯了TCP模式的一对一的模式。

简要回顾一下ZeroMQ能为你做什么：能快速传递消息到节点（包括进程，线程或节点）；ZeroMQ为程序提供了一个简单的socket API；能自动重连；能在发送端和接收端实现消息队列；能限制守护进程队列防止内存溢出；能处理socket错误；运行后台IO线程；在多节点之间无须加锁技术，不存在锁、等待、段错误或死锁。

> ZeroMQ的模式都是硬编码的，可能在后期的版本会提供用户自定义模式。

ZeroMQ的模式都是依据类型成对的sockets，也就是说，如果要理解ZeroMQ的模式，就需要理解socket类型和他们如何工作。

内置的ZeroMQ核心模式有下面几种：

- 请求-应答（Request-reply）：连接一组客户端和一组服务端，适用于分布式系统的远程过程调用
- 发布-订阅（Pub-sub）：连接一组订阅者和一组发布者，适用于数据分发系统
- 管道（Pipeline）：连接fan-out/fan-in模式的节点，可能有多个步骤和循环，适用于任务下发和结果收集
- 对独家（Exclusive pair）：只是连接两个sockets，适用于连接进程中两个线程，不能和正常的sockets对搞混淆。

以下是合法的sockets组合：

- PUB and SUB
- REQ and REP
- REQ and ROUTER (take care, REQ inserts an extra null frame)
- DEALER and REP (take care, REP assumes a null frame)
- DEALER and ROUTER
- DEALER and DEALER
- ROUTER and ROUTER
- PUSH and PULL
- PAIR and PAIR

后期可能还会看到有XPUB和XSUB对，任何其他的组合都有可能得到意外的结果，在未来的版本也有可能会报错。你能做的就是通过代码桥接这组合，从一个socket读入，再写入下一个socket中。


## High-Level Messaging Patterns

一些没有加入到核心库的模式，但是已经在ZeroMQ社区中，例如 Majordomo pattern，也就是第四章要讲到的request-replay模式，在Majordomo的github中。


## Working with Messages

核心的libzmq库提供了两个API分别处理发送和接收消息，zmq_send()和zmq_recv()，但是zmq_recv()在处理随意大小的消息时不是很好，不论提供多大的buffer size都会截断消息。结合zmq_msg_t结构体，提供另外一个更丰富更复杂的API：

- 初始化消息：zmq_msg_init(), zmq_msg_init_size(), zmq_msg_init_data().
- 发送和接收消息： zmq_msg_send(), zmq_msg_recv().
- 释放消息： zmq_msg_close().
- 接触消息体： zmq_msg_data(), zmq_msg_size(), zmq_msg_more().
- 处理消息属性： zmq_msg_get(), zmq_msg_set().
- 消息操作： zmq_msg_copy(), zmq_msg_move().

ZeroMQ消息是除0外任意大小的块，可以使用协议缓存、msgpack或json等任意程序需要的格式做格式化。选择可移植的数据是明智的，但也要根据实际考虑。

从内存的角度理解，ZeroMQ消息是zmq_msg_t格式的结构体，下面列出了使用C操作的规则：

- You create and pass around zmq_msg_t objects, not blocks of data.

- 读取消息过程：先调用 zmq_msg_init()创建一个空消息，然后将他传给zmq_msg_recv()函数。

- 创建一条新消息，调用zmq_msg_init_size()创建一条消息，同时分配一些大小的数据块，在用memcpy填充数据，最后将消息传给zmq_msg_send()。

- 释放消息（不是销毁），调用 zmq_msg_close()，这个只是丢弃引用，最终ZMQ会销毁这个消息。

- 调用zmq_msg_data()访问消息内容；调用zmq_msg_size()得到消息体数据的大小。

- 在弄清楚全部之前，别调用这几个函数：zmq_msg_move(), zmq_msg_copy(), or zmq_msg_init_data() 

- 传消息给zmq_msg_send()函数之后，ZMQ会清掉消息，设置大小为0，不能重复发送同一消息，发送后的消息不能被访问。

- 这些规则不适应zmq_send()和zmq_recv()，因为他们接收的参数是字节流，而不是消息结构体。


如果想一个消息发送多次，可以用zmq_msg_init()先初始化，然后用zmq_msg_copy()复制一个，实际上只是复制了一个引用，这样才能两次发送同一个消息（如果有多个复制，也可以发送多次），在最后一次消息发送被发送或者被关闭时，消息会被销毁。

ZeroMQ也支持多部分的消息，这在真实的程序中会经常见到，例如[Chapter 3 - Advanced Request-Reply Patterns](http://zguide.zeromq.org/page:all#advanced-request-reply)

Frames是一些用来控制ZeroMQ消息基本有线格式，也就是长度限制的数据，写过TCP编程的你会理解“究竟能从网络socket中读取多少数据”的问题答案。

一个简单的例子定义了ZeroMQ如何在TCP中读写frames，[protocol called ZMTP ](http://rfc.zeromq.org/spec:15)

在底层ZeroMQ API和引用手册上，关于消息frames有些模糊，做如下解释：

- 一个message可以分为多个parts
- 这些parts也称做frames
- 每一个parts也是一个zmq_msg_t对象
- 独立接收和发送每一个part，在底层API
- 高层API提供封装整个多parts消息

其他必知的部分：

- 可以发送长度为零的消息，例如从一个线程发送信号到另一个线程
- ZeroMQ保证发送message的全部的parts，或者是完全不发送。
- ZeroMQ并不会立即发送message（single or multipart），而是在后面的某个时间。因此multiparts消息必须全部载入内存。
- massage（single or multiparts）必须载入内存，如果想发送随意大小文件，必须将他分片成single-part messages，使用multiparts将会提升内存使用
- 在完成message接收必须调用zmq_msg_close()，当scope关闭时，在没有自动销毁对象的语言环境中。在调用发送message后无须调用该函数。

需要重复强调的是，不要使用 zmq_msg_init_data()函数，这个零拷贝函数保证会给你带来麻烦，在考虑节约毫秒级别的时间之前，还有很长的路要走。

丰富的API可能会带来很多麻烦，这些函数都是经过优化的，而不是简单的，在仔细阅读完帮助文档之前，肯定会遇到不少的麻烦


## Handling Multiple Sockets

到目前为止，所有的循环都是：
1. 等待socket上的message
2. 处理message
3. 重复

从同一个endpoints读取的最简单的方法是，使一个socket连接所有的enspoint, 并从fan-in类型的socket获取message。这在远端endpoint都是同一种模式的情况下是合法的，但是连接PULL socket到PUB socket是错误的。

用zmq_poll可以从多个socket读取，更好的办法是将zmq_poll()包装进入一个框架，把他变成一个很好的事件驱动器。

这里展示一个非阻塞读socket

这种做法的代价是在第一条消息到达之前的额外的时间延迟，和后续的几毫秒的时间等待。

【代码示例】
- chapter1/wuserver.py
- chapter1/wuserver_push.py
- chapter2/msreader.py
- chapter2/mspoller.py

结构体定义：

```
typedef struct {
    void *socket;       //  ZeroMQ socket to poll on
    int fd;             //  OR, native file handle to poll on
    short events;       //  Events to poll on
    short revents;      //  Events returned after poll
} zmq_pollitem_t;
```

## Multipart Messages

multipar message允许构造超出几个frame的message， 实际应用中很多用到了multipart message，包括打包地址信息到message中，或者是简单的序列化。

这里讲到如何盲目的安全的在任意应用程序中读写multipart messages，并且转发而不用检查他们。

每一个part都是一个zmq_msg item，例如，如果要发送一个由5个parts组成的message，那么就必须构造，发送和销毁5个zmq_msg items。

- 下面解释了如何发送multipart message：

```
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
…
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
…
zmq_msg_send (&message, socket, 0);
```

- 下面是如何接收并处理multipart message:

```
while (1) {
    zmq_msg_t message;
    zmq_msg_init (&message);
    zmq_msg_recv (&message, socket, 0);
    //  Process the message frame
    …
    zmq_msg_close (&message);
    if (!zmq_msg_more (&message))
        break;      //  Last message frame
}
```

关于multipart的一些注意事项：

- 发送multipart message时，当最后一个part被发送时，第一个part还在线上。
- 如果使用zmq_poll()，那么在收到第一个part的时候，最后一个已经到线上了。
- 对于message的所有part，要么全收，要么都不收。
- 每一个part项都是一个zmq_msg item。
- 不论是否检查property，都将收到message的全部parts。
- 在发送时， ZeroMQ将message全部载入内存，直到最后一个最后一个接收完毕，然后再将他们发出。
- 已经部分发出的message无法撤销，除非销毁socket.


## Intermediaries and Proxies

类似于消息行业的中介，协调处理两头的工作， 在ZeroMQ中，被称做 proxies, queues, forwarders, device, or brokers, 取决于上下文。


## The Dynamic Discovery Problem

在设计大型分布式架构的遇到的问题就是自动发现，如果节点不稳定时，显得尤其困难，这类问题称做“动态发现问题”

解决的方案有多种，最简单粗暴的办法就是网络架构硬编码，也就是说，如果新增节点，则需要重新修改配置。

### Figure 12 - Small-Scale Pub-Sub Network

![](https://github.com/imatix/zguide/raw/master/images/fig12.png)

事实上，这样的架构的发展会变得日益脆弱和笨重。这样连接单个publisher和上百个subscriber是没有问题的，只要在每个subscriber配置publisher的endpoint很简单，此时，publisher是静态的，而subscriber是动态的。如果要将每个subscriber连接到每个publisher，问题就变得很复杂了。

### Figure 13 - Pub-Sub Network with a Proxy

![](https://github.com/imatix/zguide/raw/master/images/fig13.png)

解决上面的难题最简单的办法就是添加一个中介，他作为一个静态的节点供所有的节点去连接，在传统的消息模型中，他是作为一个消息broker，在ZeroMQ中不存在broker这样的角色，但是可以这样很简单的构建一个中介。

你可能会考虑，如果在网络架构变得更加复杂时，是不是可以将所有的应用连接到一个broker上？作为一个初学者，这可能是一个公平的妥协，但是，如果只是简单的应用这种星型的拓扑，而不考虑性能，可能会正常工作，但是broker是一个很贪婪的东西，他会变得更加的复杂，更加状态化，终于成为一个问题。

更好的方法是把他作为一个无状态的消息switch，更恰当的比喻是一个TCP代理，除去其他额外的角色，这个例子添加一个sub-pub proxy来解决动态发现问题。在网络的中心设置一个proxy，这个proxy打开一个XSUB socket和一个XPUB socket，并分别绑定到可访问的IP地址和端口，这样所有的其他进程都可以连接到这个proxy，而不是这些进程之间相互连接，这样的话，要添加更多的subscribers和publishers就变得简单多了。


### Figure 14 - Extended Pub-Sub

![](https://github.com/imatix/zguide/raw/master/images/fig14.png)

这里用到了XPUB和XSUB作为subscribers到publishers之间的转发，XPUB/XSUB与SUB/PUB类似，只是暴露了一些额外其他的消息。proxy从XPUB读取消息并写入XSUB，完成将这些订阅消息从发布侧转发到订阅侧。这是XSUB/XPUB的主要用法。


## Shared Queue (DEALER and ROUTER sockets)

前面的hello/world例子只是实现了单个客户端和单个服务端之间的通讯，在实际应用中，更多的是多个服务端跟多个客户端通讯，此时服务端必须是无状态的，所有的状态要么在request中或者是在共享存储中--例如数据库。

### Figure 15 - Request Distribution

![](https://github.com/imatix/zguide/raw/master/images/fig15.png)

多客户端跟多服务端的连接有两种方式：蛮力的办法就是使每个客户端连接到每个服务端，如上图所示，每个客户端的请求被平均分发到各个服务端中。

这种设计的特点就是添加客户端很方便。但是如果要添加更多的服务端，则需要通知客户端服务拓扑的更改，此时就需要重新配置和重启所有的客户端。

改进的方法可以是添加一个broker，一端连着客户端，一端连着服务端，用zmq_poll()函数来管理这两个socket的活动，然而实际上并不管理任何明确的队列，ZeroMQ会在这些socket之间自动处理。

如果使用REQ和REP，他们必须要有严格的同步，即客户端发送请求，等待服务端返回，服务端收到请求，返回给客户端，客户端再接收返回。如果任意方出现故障，都会导致双方的报错。

由于broker是非阻塞的，才能用zmq_poll()函数去等待双方的socket活动，所以不能用REQ和REP模式。

### Figure 16 - Extended Request-Reply

![](https://github.com/imatix/zguide/raw/master/images/fig16.png)

有了DEALER和ROUTER两种socket，就能实现非阻塞的request/response，这里用DEALER和ROUTER来作为中介扩展REQ-REP。

这个模型中，REQ和ROUTER通信，REP和DEALER通信，在ROUTER和DEALER之间，用代码实现从一个socket拉取数据并推入另一个socket。

request-reply broker绑定了两个端点，一端供客户端连接，一端供服务端连接。

【rrclient.py rrbroker.py rrworker.py】

【代码示例】

- chapter2/rrclient.py
- chapter2/rrbroker.py
- chapter2/rrworker.py


### Figure 17 - Request-Reply Broker

![](https://github.com/imatix/zguide/raw/master/images/fig17.png)

这种采用request-reply结构的broker使客户端/服务端的架构更容易扩展，因为clients和workers之间相互不可见，唯一静态的节点就是中间的broker。


## ZeroMQ's Built-In Proxy Function

前面的例子的rrbroker中，主循环的实现才有pub-sub转发和共享队列的实现，在ZeroMQ中，提供了简单的函数`zmq_proxy()`:

```
zmq_proxy (frontend, backend, capture);
```

【代码示例】

- chapter2/rrclient.py
- chapter2/msgqueue.py
- chapter2/rrworker.py


## Transport Bridging

如何用其他网络或消息通讯技术连接ZeroMQ网络？

### Figure 18 - Pub-Sub Forwarder Proxy

![](https://github.com/imatix/zguide/raw/master/images/fig18.png)

最简单的方法是创建一个桥，这个桥负责在两个socket之间做翻译，从一种协议转义成另一种协议，ZeroMQ中桥接的常见问题是网络和端点的桥接。

上图在一个publisher和多个subscribers之间简历一个proxy，前向socket(SUB)连接的是内部网络，而后向socket(PUB)连接的是外部网络，通过前向socket订阅到天气服务，并重新发布数据到后端的socket。

【代码示例】

- chapter2/wuproxy.py


看起来很简单的proxy，关键部分是前向和后向sockets连接的是两个不同的网络，这个模型可以用来连接组播网络和tcp publisher。


## Handling Errors and ETERM

ZeroMQ的错误处理哲学是：快速失败和弹性的混合。尽可能的脆弱的内部错误和尽可能健壮的抵御外部攻击和错误，举例来说，活的细胞如果检测到内部错误会自我毁灭，也会通过一切手段来抵御来自外部的攻击。

在ZeroMQ的代码中，断言对代码的健壮性是至关重要的。如果分不清内部或外部的错误，那么这个设计是有值得加固缺陷的。在C/C++中，断言就是立即终止程序并报错，其他语言中可能是抛出异常和终止。

当ZeroMQ检测到外部错误时，少数情况下会丢弃消息，如果没有明显的策略来处理这类错误的情况下。

大部分的C代码例子中没有对错误处理的，`实际应用的代码中必须要有对每一个ZeroMQ请求的错误处理`，其他语言可能已经实现了，但是如果用C，则必须按照如下规则来：

- 创建函数如果报错，返回NULL
- 处理数据的函数可能会返回字节数，-1表示出错
- 其他函数返回0表示成功，-1表示出错或失败
- 错误码在zmq_errno()函数中
- 错误描述记录函数是zmq_strerror()

举例：

```
void *context = zmq_ctx_new ();
assert (context);
void *socket = zmq_socket (context, ZMQ_REP);
assert (socket);
int rc = zmq_bind (socket, "tcp://*:5555");
if (rc == -1) {
    printf ("E: bind failed: %s\n", strerror (errno));
    return -1;
}
```

两个主要的异常可以作为非致命错误处理：

- 如果通过ZMQ_DONTWAIT选项接收消息，如果没有等待的数据时，ZeroMQ会返回-1，并设置errno为EAGAIN

- 当一个线程调用zmq_ctx_destroy()，其他线程还在继续阻塞，zmq_ctx_destroy()的调用会关闭context，并且其他阻塞的线程都会返回-1，errno设置为ETERM.

在C/C++中，优化代码时，断言可以被完全移除，所以不要在包装整个ZeroMQ的过程中调用assert()犯错

### Figure 19 - Parallel Pipeline with Kill Signaling

![](https://github.com/imatix/zguide/raw/master/images/fig19.png)

上图展示了如何干净的关闭。如何连接sink和workers，PULL/PUSH sockets是单向连接的，当然，也可以切换到其他类型的socket，或者多种类型混用，后面的例子用sub-pub来实现发送kill信号到workers：

- sink创建一个PUB类型的socket到一个新的endpoint
- workers绑定输入到endpoint
- 当sink检测到批量的结束时，发送kill信号到PUB socket
- 当worker收到这个kill消息，自动退出

sink添加的代码如下：

```
void *controller = zmq_socket (context, ZMQ_PUB);
zmq_bind (controller, "tcp://*:5559");
…
//  Send kill signal to workers
s_send (controller, "KILL");
```

【代码示例】

- chapter1/taskevent.py
- chapter2/taskworker2.py
- chapter2/tasksink2.py



## Handling Interrupt Signals

现实的应用程序需要在收到中断信号（Ctrl+C）时完整的退出，默认情况下，只是简单的杀掉进程，消息不会被清空，文件也没有完整的关闭。

【代码示例】

- chapter1/hwclient.py
- chapter2/interrupt.py


这个程序提供一个s_catch_signals()函数捕捉Ctrl-C (SIGINT) 和 SIGTERM信号，当有该信号到达时，立即设置全局变量s_interrupted。

ZeroMQ处理中断顺序：

- 如果线程阻塞期，在收到终止信号，会直接返回EINTR
- 在收到中断信号，s_recv()返回NULL

```
s_catch_signals ();
client = zmq_socket (...);
while (!s_interrupted) {
    char *message = s_recv (client);
    if (!message)
        break;          //  Ctrl-C used
}
zmq_close (client);
```


## Detecting Memory Leaks

如果是用C/C++开发，需要程序员手动负责内存管理，那么可以用valgrind来检测内存泄漏：

- 安装valgrind

```
sudo apt-get install valgrind
```

- 默认情况下，ZeroMQ会导致valgrind抛出很多警告，关掉这些警告可以创建一个vg.supp的文件：

```
{
   <socketcall_sendto>
   Memcheck:Param
   socketcall.sendto(msg)
   fun:send
   ...
}
{
   <socketcall_sendto>
   Memcheck:Param
   socketcall.send(msg)
   fun:send
   ...
}
```

- 添加参数--DDEBUG编译程序，确保valgrind能够检测到具体哪里内存泄漏了

- 最后，执行valgrind：

```
valgrind --tool=memcheck --leak-check=full --suppressions=vg.supp someprog
```

修复所有错误之后，将得到如下信息：

```
==30536== ERROR SUMMARY: 0 errors from 0 contexts...
```


## Multithreading with ZeroMQ

ZeroMQ可能是最适合写多线程，只需要在传统socket编程的基础上稍作修改。完成一个完美的多线程通讯，无须互斥，加锁或任何其他形式的内部通信，只要在ZeroMQ sockets之间传输messages。我们说的完美是指，很容易编写和理解，跨语言，跨操作系统，在任意颗CPU的机器上都能无状态的，没有递减的指针返回。


在写ZeroMQ多线程程序时，注意以下几条：

- 多线程之间不会共享数据，唯一共享的是context，而且是线程安全的
- 远离传统的并发机制，例如：互斥、临界区、信号量等，这些都不是ZeroMQ应用所提倡的
- 创建一个ZeroMQ上下文来启动进程，传递给每个用inproc sockets连接的线程。
- 在自己的上下文下，用超然的线程来模拟独立的任务，通过TCP来连接，可以在不改变大批代码的基础上，将他移植到独立的进程中
- 线程之间所有的交互通过ZeroMQ message
- 不要在多个线程之间共享sockets，ZeroMQ的sockets不是线程安全的。虽然将一个socket从一个线程迁移到另一个线程是可以做到的，远程线程之间共享socket是跟语言绑定的，需要做一些魔法类似于sockets之间的垃圾收集。

如果需要在程序中运行多个proxy，一个常见的错误就是在每个线程运行一个连接前端和后端的proxy， 这种模式在一开始可能没有问题，但是在生产上肯定会带来随机的bug，记住一条：除非在创建socket的线程中，否则不要使用或者关闭他们。

ZMQ使用操作系统原生的线程，而不是所谓的绿色线程，这样的好处就是无须再去学习其他API了，而且这些线程对操作系统的映射是干净的，也就是说可以用Intel的ThreadChecker工具来查看程序正在执行情况；坏处就是，这种原生的多线程不方便携带，对于某些操作系统可能还需要调整。

举例说明：前面的hello-world的server都是单线程的，这样在低并发的情况下是可以满足的，但是当10000个客户端同时请求时，可能就无法胜任了，所以在现实的服务端会启动多个线程。

当然，我们可以使用启动多个worker进程的方式来实现，但是启动一个进程总比启动多个进程要来的方便且易于管理。而且，作为线程启动的worker，所占用的带宽会比较少，延迟也会较低。 

![](https://github.com/imatix/zguide/raw/master/images/fig20.png)

【代码示例】

- chapter1/mt_hwclient.py
- chapter2/mtserver.py


分析例子代码的执行：

- 服务端启动一组worker线程，每个worker创建一个REP套接字，并处理收到的请求。worker线程就像是一个单线程的服务，唯一的区别是使用了inproc而非tcp协议，以及绑定-连接的方向调换了。

- 服务端创建ROUTER套接字用以和client通信，因此提供了一个TCP协议的外部接口。
- 服务端创建DEALER套接字用以和worker通信，使用了内部接口（inproc）。
- 服务端启动了QUEUE内部装置，连接两个端点上的套接字。QUEUE装置会将收到的请求分发给连接上的worker，并将应答路由给请求方。

消息的流向是这样的：REQ-ROUTER-queue-DEALER-REP。


## Signaling Between Threads (PAIR Sockets)

在处理ZMQ的多线程间的通讯时，可能会想到用信号量或互斥，但是在ZMQ中唯一能使用的是ZMQ消息。

### Figure 21 - The Relay Race

![](https://github.com/imatix/zguide/raw/master/images/fig21.png)

这是一个传统模式的ZMQ多线程通讯：

- 两个线程之间共享context，通过inproc通信
- 父线程创建一个socket，并绑定到inproc:// endpoint， 然后开启子线程，并将context传给他
- 子线程创建第二个socket，并连接到inproc:// endpoint， 然后发送信号通知父线程已经准备就绪

例子中的这种模式是结构紧密的，当然也可以每个线程独享一个context通过inpro通信，这样就编程了松耦合。

之所以选择PAIR类型的socket，是因为其他类型的socket可能会带来副作用而影响信号：

- 如果选择PUSH-PULL模式，因为这种模式是将消息轮训发送到所有的client，那么如果不小心启动了两个client，就会丢弃一半的信号。PAIR的优势就是能拒绝多个连接。

- 如果选择DEALER-ROUTER， ROUTER会将消息包装为一个信封，会将一个空消息包装成一个multipart消息，这种情况下，如果不在乎消息数据或不检查消息内容合法性，并且不再二次读取sockets，这并不影响该功能。但是如果发送的是一条有意义的消息，那么会发现ROUTER发送的是一条错误的消息； DEALER同样会分发消息，带来和PUSH一样的效果。

- 如果选择PUB-SUB， 这确实可以精准的将消息发送，并且PUB并不会像PUSH或DEALER那样发送，然而这需要将SUB配置成一个空的订阅者，这个是比较麻烦的。



## Node Coordination

如果需要协调网络中的多个节点， PAIR sockets不再那么好用了，因为线程和节点的策略是不相同的，主要是节点是动态的，而线程是静态的，PAIR sockets无法实现在节点掉线后自动重连。

### Figure 22 - Pub-Sub Synchronization

![](https://github.com/imatix/zguide/raw/master/images/fig22.png)

线程和节点的第二个重大不同就是，线程个数是确定的，而节点的个数可能多很多，举例来说：


【代码示例】

- chapter2/syncsub.py
- chapter2/syncpub.py

分析代码如何执行：

- publisher预先知道有多少个subscriber
- publisher启动并等待subscriber来连接，每一个subscriber订阅频道并通过另一个socket通知publisher准备就绪
- 当publisher确认所有订阅者就绪后，开始发布消息。


## Zero-Copy

所谓的“零拷贝”是指：ZMQ的api提供直接从程序的缓存收发数据， 不存在拷贝过程，这一点能提升程序的性能。

在频繁处理上千KB数据时，才用到这个特性。在小字节低频率使用他就会使得程序过于复杂，没有实际的好处。通看所有的优化，大都是在熟悉他的功能之后。

用`zmq_msg_init_data()`函数创建一个message，指向的是使用malloc()函数或其他函数分配的数据块，然后将他传递给`zmq_msg_send()`，当创建message时，也指定了释放数据块的回调函数，该函数在数据发送完毕执行。举例：

```
void my_free (void *data, void *hint) {
    free (data);
}
//  Send message from buffer, which we allocate and ZeroMQ will free for us
zmq_msg_t message;
zmq_msg_init_data (&message, buffer, 1000, my_free, NULL);
zmq_msg_send (&message, socket, 0);
```

> NOTE: 无须手动调用zmq_msg_close()，因为libzmq库会在消息发送完毕后自动调用。

在接收数据无法做到零拷贝：ZMQ会将收到的消息放入一块内存区域供你读取，但不会将消息写入程序指定的内存区域。

ZMQ的多段消息能够很好地支持零拷贝。在传统消息系统中，你需要将不同缓存中的内容保存到同一个缓存中，然后才能发送。但ZMQ会将来自不同内存区域的内容作为消息的一个帧进行发送。而且在ZMQ内部，一条消息会作为一个整体进行收发，因而非常高效。


## Pub-Sub Message Envelopes（发布 - 订阅消息信封）

在发布-订阅模式中，我们可以将key拆分成一个单独的消息帧，也就是信封。key和data是分开的。

### Figure 23 - Pub-Sub Envelope with Separate Key

![](https://github.com/imatix/zguide/raw/master/images/fig23.png)

订阅者是做前向匹配，使用信封技术就能保证匹配不会超越信封的边界。


【代码示例】

- chapter2/psenvsub.py
- chapter2/psenvpub.py

这个例子中，订阅者通过过滤拒绝或接收multipart message。发布者也可以是multiple publishers，此时订阅者可以根据发布者地址来过滤。

### Figure 24 - Pub-Sub Envelope with Sender Address

![](https://github.com/imatix/zguide/raw/master/images/fig24.png)


## High-Water Marks（高潮线）

考虑一种情况，A高速的发送消息给B，而B接收消息后需要做处理。突然B变得负载非常高，短期内无法及时处理消息，如果A此时还是继续快速的往B发送消息，此时A的网卡缓存，内存，CPU都会瞬间暴涨，最后系统崩溃。

一种解决方案是流量控制，在B处理不过来的时候，告诉A需要做流量控制了。单纯靠流量控制也是不够的，就像传输层无法通知应用层停止消息的发送，可以设置缓存阀值，当达到阀值时，采取一些有效的方法，例如丢弃消息或等待发送。

ZMQ使用HWM (high-water mark)来定义内部pipes能力，每一个进出socket的连接都有自己的pipes，一些（PUB, PUSH）只有发送缓存，一些（SUB, PULL, REQ, REP）只有接收缓存，一些（DEALER, ROUTER, PAIR）既有发送也有接收缓存。

ZeroMQ v2.x的HWM默认是无限大小，ZeroMQ v3.x默认是1000，当socket达到HWM阀值时，根据消息类型开始丢弃数据或阻塞： PUB and ROUTER会开始丢弃数据，其他类型则会开始阻塞，在inproc类型中，发送者和接收者共用同一个buffer,此时实际的HWM是双方HWM之和


## Missing Message Problem Solver（解决信件丢失）

实际应用中经常遇到的一个问题是：丢失期待接收的消息，下图遍历常见的几种情况：

### Figure 25 - Missing Message Problem Solver

![](https://github.com/imatix/zguide/raw/master/images/fig25.png)


总结一下：

- SUB类型的socket，调用zmq_setsockopt()函数定义ZMQ_SUBSCRIBE，如果订阅“”空串，那么将收到所有消息。

- 如果在PUB socket开始发送数据之后再启动SUB socket，在连接建立之前都会有数据丢失，解决的方法就是先启动SUB socket.

- 即使SUB-PUB同步了，还是有可能会丢失数据，因为内部的queue并不是一定要等到外部的连接建立后才开始建立。如果可以调转SUB和PUB的方向，SUB端调用bind，PUB端调用connect，可能得到意想不到的结果。

- 如果是REP-REQ类型的sockets,如果不是严格的send/recv/send/recv顺序，ZMQ会报错，而且看起来像是消息丢失了，所以需要在ZMQ调用是检查错误。

- 对于PUSH sockets， 第一个PULL socket连接抓到的消息是不公平的，消息准确公平的轮询只是在所有的PULL socket都成功连接之后，可能需要耗费几毫秒，可以考虑用ROUTER/DEALER替代PUSH-PULL，或者负载均衡模式。

- 多线程之间共享socket会带来一些意外的crash。

- 如果使用inproc，确保所有的sockets是共用同一个context。inproc不像tcp是断开传输，所以必须先bind再connect。

- ROUTER sockets，如果使用畸形的身份帧很容易意外的丢失数据。可以考虑在 ROUTER sockets 设置ZMQ_ROUTER_MANDATORY， 当然要记得在每一个send调用之后检查返回码。

- 如果实在不知道问题出在哪里，只有简化架构调试，或者像社区需求帮助。
