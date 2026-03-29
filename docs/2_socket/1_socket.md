---
layout: post
title: （一）网络编程那些事儿：socket
categories: 计算机网络
related_posts: True
tags: network
toc:
  sidebar: right
---

## （一）网络编程那些事儿：socket

### 1. socket 概念

- 什么是 socket？
  Socket（套接字）是计算机网络中用于实现不同主机之间的通信的一种技术。它提供了一种在网络上发送和接收数据的方式。在 TCP/IP 网络模型中，Socket 是应用层和传输层之间的接口。

- 对 socket 的理解可以从四个角度出发：

  1. **Socket 创建**：选择相关协议，如 IP 协议和 TCP/UDP 等。
  2. **Socket 连接**：设置两端的连接地址，如本机的端口号，目标端的 IP 地址-端口号等等。
  3. **Socket 通信**：一组用于通信的 api，收发数据。
  4. **Socket 的状态**：Socket 在其生命周期中会经历多种状态，如 CLOSED、LISTEN、SYN_SENT、SYN_RECEIVED、ESTABLISHED 等。了解这些状态对于理解网络通信的过程非常重要。
  5. **Socket 的设置**：如阻塞和无阻塞设置等

### 2. TCP-socket

#### 2.1 tcp-服务端流程

```c++
   int sockfd = socket(AF_INET, SOCK_STREAM, 0);
   bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
   listen(sockfd, 5);
   int client_sock = accept(sockfd, (struct sockaddr*)&client_addr, &addr_len);
   recv(client_sock, buffer, sizeof(buffer), 0);
   send(client_sock, response, strlen(response), 0);
   close(client_sock);
   close(sockfd);
```

#### 2.2 tcp-客户端流程

```c++
   int sockfd = socket(AF_INET, SOCK_STREAM, 0);
   connect(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
   send(sockfd, message, strlen(message), 0);
   recv(sockfd, buffer, sizeof(buffer), 0);
   close(sockfd);
```

总结

- 服务端需要先启动并监听，等待客户端连接。
- 客户端主动连接服务端，建立通信后双方可以通过 `send` 和 `recv` 进行数据交换。
- 通信完成后，双方需要关闭套接字释放资源。

### 3. UDP-socket

以下是客户端和服务端使用 Socket 建立 **UDP 通信** 的基本流程：

#### 3.1 udp-服务端流程

```cpp
  int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
  bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
  recvfrom(sockfd, buffer, sizeof(buffer), 0, (struct sockaddr*)&client_addr, &addr_len);
  sendto(sockfd, response, strlen(response), 0, (struct sockaddr*)&client_addr, addr_len);
  close(sockfd);
```

#### 3.2 客户端流程

```cpp
  int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
  inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr);
  sendto(sockfd, message, strlen(message), 0, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
  recvfrom(sockfd, buffer, sizeof(buffer), 0, (struct sockaddr*)&from_addr, &addr_len);
  close(sockfd);
```

- **UDP 是无连接的协议**，不需要像 TCP 那样建立连接。
- 服务端通过 `recvfrom` 接收数据，并通过 `sendto` 发送数据。
- 客户端通过 `sendto` 发送数据，并通过 `recvfrom` 接收数据。
- 通信完成后，双方需要关闭套接字释放资源。

### 4. API-usage

#### 4.1 socket 创建

```cpp
extern int socket (int __domain, int __type, int __protocol) __THROW;
```

- `__domain`：
  - IP 地址类型
  - AF_INET 表示使用 IPv4，如果使用 IPv6 请使用 AF_INET6。
- `__type`：
  - 数据传输方式
  - SOCK_STREAM 表示流格式、面向连接，多用于 TCP。
  - SOCK_DGRAM 表示数据报格式、无连接，多用于 UDP。
- `__protocol`：
  - 协议，0 表示根据前面的两个参数自动推导协议类型。
  - 设置为 IPPROTO_TCP 和 IPPTOTO_UDP，分别表示 TCP 和 UDP。

#### 4.2 IP 地址 - sockaddr

- ip 地址

  ```cpp
  struct sockaddr_in
  {
    sa_family_t sin_family;
    in_port_t sin_port;
    struct in_addr sin_addr;
  };
  struct sockaddr_in serv_addr;
  bzero(&serv_addr, sizeof(serv_addr));
  serv_addr.sin_family = AF_INET;
  serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
  serv_addr.sin_port = htons(8888);
  ```

  - `sin_family`:
    - 地址族
    - AF_INET 表示使用 IPv4，如果使用 IPv6 请使用 AF_INET6。
  - `sin_port`:
    - 端口号
    - 使用网络字节序（需要通过 htons() 转换）
  - `sin_addr`:
    - IP 地址，使用网络字节序（可以通过 inet_addr() 设置）

#### 4.3 绑定 ip 地址 - connect

```cpp
extern int connect (int __fd, __CONST_SOCKADDR_ARG __addr, socklen_t __len);
```

#### 4.4 监听 - listen

```cpp
// 最后我们需要使用`listen`函数监听这个socket端口，这个函数的第二个参数是listen函数的最大监听队列长度，系统建议的最大值`SOMAXCONN`被定义为128。
listen(sockFd, SOMAXCONN);
```

#### 4.5 同意连接 - accept

#### 4.6 通信 - recv/write/send/read

- send/recv(read/write)返回值大于 0、等于 0、小于 0 的区别
  - recv
    - 阻塞与非阻塞 recv 返回值没有区分，都是 <0：出错，=0：连接关闭，>0 接收到数据大小
    - 特别：非阻塞模式下返回 值 <0 时并且(errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN)的情况 下认为连接是正常的，继续接收。
    - 只是阻塞模式下 recv 会阻塞着接收数据，非阻塞模式下如果没有数据会返回，不会阻塞着读，因此需要 循环读取
  - write：
    - 阻塞与非阻塞 write 返回值没有区分，都是 <0 出错，=0 连接关闭，>0 发送数据大小
    - 特别：非阻塞模式下返回值 <0 时并且 (errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN)的情况下认为连接是正常的， 继续发送
    - 只是阻塞模式下 write 会阻塞着发送数据，非阻塞模式下如果暂时无法发送数据会返回，不会阻塞着 write，因此需要循环发送。
  - read：
    - 阻塞与非阻塞 read 返回值没有区分，都是 <0：出错，=0：连接关闭，>0 接收到数据大小
    - 特别：非阻塞模式下返回 值 <0 时并且(errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN)的情况 下认为连接是正常的，继续接收。
    - 只是阻塞模式下 read 会阻塞着接收数据，非阻塞模式下如果没有数据会返回，不会阻塞着读，因此需要 循环读取
  - send：
    - 阻塞与非阻塞 send 返回值没有区分，都是 <0：出错，**=0：连接关闭**，>0 发送数据大小。 还没发出去，连接就断开了，有可能返回 0
    - 特别：非阻塞模式下返回值 <0 时并且 (errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN)的情况下认为连接是正常的， 继续发送
    - 只是阻塞模式下 send 会阻塞着发送数据，非阻塞模式下如果暂时无法发送数据会返回，不会阻塞着 send，因此需要循环发送

### 5. 进阶 api

fcntl: 设置套接字为非阻塞模式。
setsockopt 和 getsockopt: 设置或获取套接字选项。

### 99. quiz

#### 1 socket 相关的状态位有哪些？

- 客户端调用 connect 发送 SYN 包，客户端协议栈收到 ACK 之后，使得应用程序从 connect 调用返回，表示客户端到服务器端的单
  向连接建立成功

#### 2. `accept`发生在什么阶段？

三次握手之后，tcp 连接会加入到 accept 队列。accept()会从队列中取一个连接返回，若队列为空，则阻塞。

#### 3.信号 SIGPIPE 与 EPIPE 错误码

- 在 linux 下写 socket 的程序的时候，**如果服务器尝试 send 到一个 disconnected socket 上，就会让底层抛出一个 SIGPIPE 信号**。 这个信号的缺省处理方法是退出进程，大多数时候这都不是我们期望的。也就是说，**当服务器繁忙，没有及时处理客户端断开连接的事件，就有可能出现在连接断开之后继续发送数据的情况，如果对方断开而本地继续写入的话，就会造成服务器进程意外退出**
- 根据信号的默认处理规则 SIGPIPE 信号的默认执行动作是 terminate(终止、退出)，所以 client 会退出。若不想客户端退出可以把 SIGPIPE 设为 SIG_IGN 如：signal(SIGPIPE, SIG_IGN); 这时 SIGPIPE 交给了系统处理。 服务器采用了 fork 的话，要收集垃圾进程，防止僵尸进程的产生，可以这样处理： signal(SIGCHLD,SIG_IGN); 交给系统 init 去回收。 这里子进程就不会产生僵尸进程了

#### 4. **如何检测对端已经关闭 socket**

- 根据 errno 和 recv 结果进行判断
  - 在 UNIX/LINUX 下，**非阻塞模式 SOCKET 可以采用 recv+MSG_PEEK 的方式进行判断**，其中 MSG_PEEK 保证了仅仅进行状态判断，而不影响数据接收
  - 对于主动关闭的 SOCKET, recv 返回-1，而且 errno 被置为 9（#define EBADF 9 // Bad file number ）或 104 (#define ECONNRESET 104 // Connection reset by peer)
  - 对于被动关闭的 SOCKET，recv 返回 0，而且 errno 被置为 11（#define EWOULDBLOCK EAGAIN // Operation would block ）**对正常的 SOCKET, 如果有接收数据，则返回>0, 否则返回-1，而且 errno 被置为 11**（#define EWOULDBLOCK EAGAIN // Operation would block ）因此对于简单的状态判断(不过多考虑异常情况)：recv 返回>0， 正常

#### 5. udp 调用 connect 有什么作用？

- 因为 UDP 可以是一对一，多对一，一对多，或者多对多的通信，所以每次调用 sendto()/recvfrom()时都必须指定目标 IP 和端口号。通过调用 connect()建立一个端到端的连接，就可以和 TCP 一样使用 send()/recv()传递数据，而不需要每次都指定目标 IP 和端口号。但是它和 TCP 不同的是它没有三次握手的过程
- 可以通过在已建立连接的 UDP 套接字上，调用 connect()实现指定新的 IP 地址和端口号以及断开连接
- **处理非阻塞 connect 的步骤**（重点）：

  1. 创建 socket，返回套接口描述符；
  2. 调用 fcntl 把套接口描述符设置成非阻塞；
  3. 调用 connect 开始建立连接；
  4. 判断连接是否成功建立

     1. 如果 connect 返回 0，表示连接成功（服务器和客户端在同一台机器上时就有可能发生这种情况）
     2. 否则，需要通过 io 多路复用(select/poll/epoll)来监听该 socket 的可写事件来判断连接是否建立成功：当 select/poll/epoll 检测出该 socket 可写，还需要通过调用 getsockopt 来得到套接口上待处理的错误（SO_ERROR）。**如果连接建立成功，这个错误值将是 0**；**如果建立连接时遇到错误，则这个值是连接错误所对应的 errno 值**（比如：ECONNREFUSED，ETIMEDOUT 等）

#### 6. accept 返回 EMFILE 的处理方法

- SO_ERROR 选项

  ```c
  #include <sys/socket.h>

  int getsockopt(int sockfd, int level, int option, void *optval, socklen_t optlen);
  ```

  - 当一个 socket 发生错误的时候，将使用一个名为 SO_ERROR 的变量记录对应的错误代码，这又叫做 pending error，SO_ERROR 为 0 时表示没有错误发生。一般来说，有 2 种方式通知进程有 socket 错误发生：

    1. 进程阻塞在 select 中，有错误发生时，select 将返回，并将发生错误的 socket 标记为可读写
    2. 如果进程使用信号驱动的 I/O，将会有一个 SIGIO 产生并发往对应进程

  - 此时，进程可以通过 SO_ERROR 取得具体的错误代码。getsockopt 返回后，\*optval 指向的区域将存储错误代码，而 so_error 被设置为 0

  - 当 SO_ERROR 不为 0 时，如果进程对 socket 进行 read 操作，若此时接收缓存中没有数据可读，则 read 返回-1，且 errno 设置为 SO_ERROR，SO_ERROR 置为 0，否则将返回缓存中的数据而不是返回错误；如果进行 write 操作，将返回-1，errno 置为 SO_ERROR，SO_ERROR 清 0

  注意，这是一个只可以获取，不可以设置的选项。

#### 3. 说明 socket 网络编程有哪些系统调用?其中 close 是一次就能直接关闭的吗,半关闭状态是怎么产生的?

    socket()    创建套接字
    bind()      绑定本机端口
    connect()   建立连接     (TCP三次握手在调用这个函数时进行)
    listen()    监听端口
    accept()    接受连接
    recv(), read(), recvfrom()  数据接收
    send(), write(), sendto()   数据发送
    close(), shutdown() 关闭套接字

使用 close()时,只有当套接字的引用计数为 0 的时候才会终止连接,而用 shutdown()就可以直接关闭连接

#### 4. 同一个 IP 同一个端口可以同时建立 tcp 和 udp 的连接吗

可以,同一个端口虽然 udp 和 tcp 的端口数字是一样的,但实质他们是不同的端口,所以是没有影响的,从底层实质分析,对于每一个连接内核维护了一个五元组,包含了源 ip,目的 ip/源端口目的端口/以及传输协议,在这里尽管前 4 项都一样,但是传输协议是不一样的,所以内核会认为是 2 个不同的连接,在 ip 层就会进行开始分流,tcp 的走 tcp,udp 走 udp.
