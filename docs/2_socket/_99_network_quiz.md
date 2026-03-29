---
layout: post
title: Socket 与网络编程要点（精简版）
categories: Socket
toc:
  sidebar: right
---

## 概览：主要问题与主题
本页面按主题整理常见疑问与最佳实践：IO 模型、阻塞/非阻塞、send/recv 行为、非阻塞 connect、IO 复用 vs 异步 IO、粘包/半包与协议设计、收发的“正确姿势”、服务端防护与调度。

---

## 1. IO 模型与阻塞/非阻塞
- 常见 IO 模型：阻塞 I/O、非阻塞 I/O、I/O 复用（select/poll/epoll）、异步 I/O（AIO）。  
- 阻塞：调用会挂起线程直到完成或超时；非阻塞：调用会立即返回（失败时返回 EWOULDBLOCK/EAGAIN）。  
- 同步 vs 异步：同步由调用方等待完成；异步由内核或框架完成后通知（回调/事件）。  
- IO 复用通常是同步模型：select/poll/epoll 阻塞等待事件发生，应用在事件就绪后进行读写。异步 IO 则由内核完成数据拷贝并通知应用。

---

## 2. select / poll / epoll 的区别（要点）
- select：fd 集合限制、每次复制 fdset、O(N) 扫描。  
- poll：无 fdset 位图限制，但仍 O(N) 扫描事件数组。  
- epoll（Linux）：基于事件就绪通知，适合大量连接；支持 LT（水平触发）与 ET（边缘触发）。  
- LT: 只要条件满足持续触发；ET: 仅在状态变化时触发，使用 ET 时必须一次性读/写尽可能多的数据，否则可能丢失通知。

---

## 3. 阻塞/非阻塞 socket 与 send/recv 行为（要点）
- send 将数据从用户空间拷贝到内核发送缓冲区（并不保证立刻发出）；recv 将内核缓冲区数据拷贝到用户缓冲区。  
- 阻塞模式：当内核缓冲区已满（发送）或无数据（接收）时，send/recv 会阻塞。  
- 非阻塞模式：若不可操作，send/recv 立即返回 -1，并置 errno 为 EWOULDBLOCK（或 EAGAIN）；EINTR 表示被信号中断应重试。  
- 返回值判断要严谨：n > 0 可能 < 期望长度，应循环写直到发送完或缓存剩余时登记到待发送缓冲区。

示例（设置非阻塞的简要方式）：
```c
// Linux: fcntl 示例
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

Windows 使用 ioctlsocket(..., FIONBIO, &arg)。

---

## 4. 非阻塞 connect 的正确做法（Linux 要点）
- 设置 socket 为非阻塞后调用 connect，若返回 -1 且 errno==EINPROGRESS，表示连接进行中。  
- 使用 select/poll/epoll 等等待可写（或异常），但在 Linux 上可写并不必然表示成功，需通过 getsockopt(SO_ERROR) 检查实际错误码（0 表示成功）。  
- Windows 下可用 select/WsaEvent/WSAAsyncSelect 等机制。  

注意：创建时直接用 SOCK_NONBLOCK 或 accept4 能减少一次 fcntl。

---

## 5. IO 复用 vs 异步 IO（精炼）
- IO 复用：单/少量线程轮询内核事件表来处理大量连接（节省线程上下文开销）。  
- 异步 IO：内核负责实际 I/O 操作并在完成时通知（更低延迟的并发模型，但实现复杂度和可移植性较高）。  
- Reactor（事件分发，应用执行 I/O）与 Proactor（内核完成 I/O，应用处理完成事件）区分：epoll 常用作 Reactor。

---

## 6. 粘包与半包（TCP 是字节流）与解决方案
问题来源：TCP 的字节流特性，发送方的多次 send 可能在接收端形成任意分割或合并。

常见解决方案：
1. 固定包长（simple，但灵活性差）。  
2. 特殊分隔符（如 CRLF，需转义处理）。  
3. 包头 + 包体（常用）：包头包含包体长度（或整个包长度），接收端先读包头再按长度读包体。  

解析要点：  
- 接收缓冲区应支持“peek”或先检查长度再取数据；  
- 对 bodysize 做上下限校验（防止内存耗尽攻击）；  
- 使用循环在缓冲区中处理可能的多个包（while 循环直到缓冲区不足以构成完整包）。

---

## 7. 收发数据的正确姿势（实践要点）
- 读（接收）流程：在可读事件触发后循环 recv，直到返回 EWOULDBLOCK（ET 模式需读尽）或无更多数据；将数据放入接收缓冲并尝试解包处理。  
- 写（发送）流程：先直接尝试 send；若全部发送成功不注册写事件；若部分发送或 EWOULDBLOCK，将剩余数据放入发送缓冲并注册可写事件；可写事件触发时继续发送并在发送完后取消写事件。  
- 在 epoll-ET 下读写需一次性读尽/写尽，否则可能丢失事件通知。

---

## 8. 服务端防护与资源管理
- 控制每个连接的发送缓冲上限，超过则认为该连接异常并关闭（防止内存耗尽）。  
- 对长期驻留在发送缓冲的数据设置超时检测，超过阈值关闭连接并回收资源。  
- 明确 fd ownership（哪个线程负责 close），避免并发 close 导致竞态。  
- accept 负载分摊：可用 SO_REUSEPORT、多线程/进程 accept（注意竞争与分配策略）。

---

## 9. 协议设计与跨语言兼容
- 推荐使用包头+包体格式，包头包含长度与必要元信息（版本、压缩、校验等）。  
- 注意字节序（网络字节序为大端）：发送端建议使用 htons/htonl，接收端使用 ntohs/ntohl。跨语言解析时务必统一字节序和字段顺序。  
- 常用框架（如 Netty）提供多种 FrameDecoder，有助于减少样板解包代码；自实现时建议封装 BinaryRead/Write 流接口。

---

## 10. 面试/复习建议（简短）
- 能简述 select/poll/epoll 的区别及各自适用场景。  
- 熟练说明 TCP 三次握手、四次挥手及 TIME_WAIT 的目的。  
- 能写出基于“包头+包体”的解包伪码，说明如何处理粘包/半包。  
- 理解非阻塞 connect 的跨平台差异与正确检查方法（Linux 的 getsockopt(SO_ERROR)）。  
- 了解 epoll LT/ET 的语义差异并给出相应的读写实现要点。

## 1. IO 模型与阻塞/非阻塞
- 常见 IO 模型：阻塞 I/O、非阻塞 I/O、I/O 复用（select/poll/epoll）、异步 I/O（AIO）。
- 阻塞 vs 非阻塞：阻塞调用会挂起线程等待结果；非阻塞立即返回，需要轮询或事件驱动。
- 同步 vs 异步：同步调用由调用方等待完成；异步由内核或框架在完成时通知或回调。

## 2. select / poll / epoll 的区别
- select：描述符数受限，复制 fdset，O(N) 扫描。
- poll：无描述符位图限制，但仍需 O(N) 扫描事件列表。
- epoll：Linux 下高效方案，基于事件通知，适合大量连接；支持两种模式：水平触发(LT) 和 边缘触发(ET)。

要点：epoll 在大连接数时更高效，但代码复杂度（ET）更高，需注意事件消费语义。

## 3. epoll 工作模式（LT / ET）
- LT（水平触发）：只要条件存在，事件会反复通知。
- ET（边缘触发）：只在状态变化时通知，需一次性读尽数据以免丢失通知。

## 4. 常见并发与同步问题
- pthread_cond_signal vs pthread_cond_broadcast：signal 唤醒一个等待线程，broadcast 唤醒全部等待线程。
- 多线程资源同步：常用 Mutex/Cond/Semaphore/RWLock，注意死锁场景（锁顺序、不释放锁等）。

## 5. TCP 基础（握手/挥手/状态）
- 三次握手（建立连接）：SYN → SYN+ACK → ACK，解决序号同步与防止已滞留连接误认问题。
- 两次握手问题：无法确认双方序号与接收能力，容易产生半连接或重复连接问题。
- 四次挥手（关闭连接）：每方向独立关闭（FIN/ACK），出现 TIME_WAIT 的原因是确保最后 ACK 能到达并消除迟到报文、防止端口重用引起旧连接混淆。
- 如果最后 ACK 丢失，发送者会重传 FIN/ACK，TIME_WAIT 确保能处理重传。
- TIME_WAIT 必要性：防止旧重复报文影响新连接并允许可靠重传。

## 6. TCP 可靠性与流控/拥塞控制
- 可靠性机制：序列号、确认号（ACK）、重传、校验和（checksum）、滑动窗口（流量控制）、拥塞控制（慢启动、拥塞避免、快重传、快恢复）。
- 粘包/半包：TCP 是字节流，需在应用层设计协议（固定头+length、分隔符、消息边界）来解决。常见方案：前置长度字段、定界符、消息帧解析器、或使用高层协议（HTTP）。

## 7. 其他常见问题与设计点
- fd 的负载均衡：多线程/多进程分发、accept 负载均衡（SO_REUSEPORT）、事件分组处理。
- 子线程回收 fd：在设计上明确 ownership，确定哪个线程负责 close，避免竞态并使用同步。
- 如何判断完整 HTTP 请求：解析 HTTP 首部（Content-Length 或 Transfer-Encoding: chunked），依据协议判断是否完整。
- Reactor vs Proactor：
  - Reactor：事件分发，应用读写操作；内核告知事件发生。
  - Proactor：内核完成 I/O 并返回结果，应用仅处理完成事件（常见于真正的异步 I/O 实现）。
- IO 复用 vs 多线程/多进程：IO 复用用少线程处理多连接，节省线程创建/切换开销；多线程是每连接独占线程，模型简单但开销大。

## 8. 面试复习建议（简短）
- 熟记 select/poll/epoll 的优缺点与使用场景，能画出事件流和处理逻辑（LT/ET）。
- 熟悉 TCP 三次握手与四次挥手的报文序列与状态变化，理解 TIME_WAIT 的目的与后果。
- 能用代码/伪码描述如何处理粘包（前置长度、循环读取直至满足长度）。
- 理解常见并发原语（mutex/cond/sem）及常见死锁成因，能给出避免策略。
- 能解析 HTTP 报文判断完整性（Content-Length / chunked / connection close）。
