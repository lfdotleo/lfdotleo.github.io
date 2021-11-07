---
title: "IO 模型详解"
date: 2021-10-28T20:07:49+08:00
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: dotleo
tags:
- IO
- Linux
- NIO
- Java
categories:
- Java

---

## 系统调用

当系统运行时，第一个启动的程序就是 `kernel` ，它的作用就是各个硬件工作的调度器。为了保证硬件操作规范及 kernel 的稳定性等原因，Linux 将分为了内核空间和用户空间，普通的应用程序是运行在用户空间，无法访问内核空间中的指令等，当然也就无法直接操作硬件。当进程需要访问硬件设备（比如读取磁盘文件、接受网络数据等）时，必须由用户空间（或者叫用户态、用户模式）切换到内核空间，这是通过 **系统调用** 实现。strace 可以跟踪到一个进程产生的系统调用，包括参数、返回值及执行消耗时间等。

<!--more-->

关于 strace 的使用及参数，可参考 [6. strace 跟踪进程中的系统调用 — Linux Tools Quick Tutorial](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/strace.html)，这里就不过多介绍了，我们主要了解一个可以详细打印系统调用的运行参数。

`strace -ff -o <filename> command`: 该命令及参数可以将该 `command`运行后的 fork 出的所有线程的跟踪结果输出到相应的 `filename.pid` 中，`pid` 是进程号及 fork 出的线程编号。

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println("hello, world");
    }
}
```

1. 先跑起来再说，先新建一个 `Demo.java` 源文件。
1. 然后运行`mkdir javatest`创建一个文件夹，方便我们将所有线程的跟踪结果输出到一起方便查看。
1. 执行`strace -ff -o javatest/demo java Demo`
1. 此时 `javatest` 件夹中就出现了以 `demo.pid` 命名的若干文件。
1. 我们找到最大的一个，打开后搜索 `hello, world` ，就能看到它有一个系统调用 `write(1, "hello, world", 12)            = 12`，即系统调用了 `write()` ，其中第一个参数 1 表示标准输出流，即通过系统调用将 "hello,world" 输出到标准输出流中。

到这里，我们就可以使用 strace 分析一个程序的系统调用了。

## 学会查看文件描述符

> [内核](https://baike.baidu.com/item/内核/108410)（kernel）利用文件描述符（file descriptor）来访问文件。文件描述符是 [非负整数](https://baike.baidu.com/item/非负整数/2951833)。打开现存文件或新建文件时，内核会返回一个文件描述符。读写文件也需要使用文件描述符来指定待读写的文件。

Linux 的 `/proc/$pid/fd` 目录下存放了此进程所有打开的 fd。当然，有些可能不是本进程自己打开的，如通过 fork() 从父进程继承而来的。如果我们想查看文件描述符，需要先获取到进程 id(pid)，然后执行`ls /proc/$pid/fd`即可查看到该进程打开的文件描述符。

为了方便观察，我们稍修改一下上面的程序：

```java
public class Demo {
    public static void main(String[] args) throws Exception {
        System.out.println("hello, world");
        Thread.sleep(60000L);
    }
}
```

`java Demo`运行程序后，使用`jps -l | grep Demo`找到 `pid`，然后运行`ls -l /proc/$pid/fd` 。

```
lrwx------ 1 root root 64 7 月   4 23:56 0 -> /dev/pts/9
lrwx------ 1 root root 64 7 月   4 23:56 1 -> /dev/pts/9
lrwx------ 1 root root 64 7 月   4 23:56 2 -> /dev/pts/9
lr-x------ 1 root root 64 7 月   4 23:56 3 -> /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.252.b09-2.el7_8.x86_64/jre/lib/rt.jar
```

在我的电脑上，结果如上所示。其中*文件描述符 3* 是一个 Java 运行时必须的 jar 包，其他 0~2 分别代表标准输入、标准输出及标准错误，每个进程都会有这 3 个描述符。我们可以使用`strace`看到`System.out.println("hello, world");` 底层系统调用了`write(1, "hello, world", 12)            = 12`，即向标准输出中输出"hello, world"。

## BIO

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class Main {
    public static void main(String[] args) throws IOException {
        // socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 5
        ServerSocket socket = new ServerSocket();
        System.out.println("serversocket is created");
        // bind(5, {sa_family=AF_INET, sin_port=htons(6666), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
        // listen(5, 50)                           = 0
        socket.bind(new InetSocketAddress(6666));
        System.out.println("serversocket is binded with 6666");
        // 阻塞：
        // poll([{fd=5, events=POLLIN|POLLERR}], 1, -1
        // 当有客户端连接时：
        // poll([{fd=5, events=POLLIN|POLLERR}], 1, -1) = 1 ([{fd=5, revents=POLLIN}])
        // accept(5, {sa_family=AF_INET, sin_port=htons(42594), sin_addr=inet_addr("127.0.0.1")}, [16]) = 6
        Socket client = socket.accept();
        System.out.println("serversocket is polling");
        InputStream in = client.getInputStream();
        BufferedReader reader = new BufferedReader(new InputStreamReader(in));
        while (true) {
            // 阻塞：
            // recvfrom(6,
            System.out.println(reader.readLine());
        }
    }
}
```

如上代码，是一个简单的 socket 通信服务端的代码，可以使用`strace`命令自己查看系统调用情况，其中关键几步的系统调用在注释中已经给出。可以看到，一个服务端的 socket 经历`socket`- `bind` - `listen`，然后调用`poll`等待客户端连接。当有客户端连接时，`poll`结束阻塞返回结果，然后系统调用`accept`会创建新的文件描述符（上面注释中的 *6* ）用来和客户端通信。

因为我们代码中需要读取一行数据，`reader.readLine()`底层的系统调用`recvfrom`是**阻塞**的，当没有内容时会一直阻塞该线程。

![110713M3DQQ58OWUea](https://cdn.jsdelivr.net/gh/lfdotleo/pic2022q1@main/110713M3DQQ58OWUea.jpg)

此时，如果我们想要再用一个客户端连接服务端，是不能得到及时响应的，因为线程被阻塞住了，大家可以动手尝试（如果代码和上面相同，则可以使用`nc localhost 6666`在本机模拟连接）。

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class Main2 {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket();
        socket.bind(new InetSocketAddress(6666));
        while (true) {
            Socket client = socket.accept();
            new Thread(() -> {
                try {
                    InputStream in = client.getInputStream();
                    BufferedReader reader = new BufferedReader(new InputStreamReader(in));
                    while (true) {
                        System.out.println(reader.readLine());
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }).start();
        }

    }
}
```

为了能响应多个客户端的连接，我们通过每个客户端连接创建一个线程的方式改造了代码。这样，即使客户端没有发送内容导致服务端对应线程**读取阻塞**，也不会影响其他线程及客户端的接入。上述代码是一种比较常见的 Java Socket 编程方式，当然如果使用线程池创建线程可以减少因为不断创建线程带来的性能损耗。

上面这种方式是一种很常见的使用 BIO 与客户端通信，它有非常明显的**缺点** ：

1. 多线程时，CPU 的线程切换会导致上下文切换，带来不小的性能开销。
1. 每个线程本身需要占用资源。每个客户端接入需要新建线程，每个线程本身就需要占用不小内存资源。

## NIO（非阻塞 IO)

BIO 存在上面所说缺点，它的本质是因为读取时会阻塞。因此如果我们读取时，如果客户端有数据（已发送数据或正在发送），则读取数据。没有数据时，立即返回。如此，我们就可以使用一个线程，轮询所有的客户端 socket 连接。有数据则读取，没有数据则立即返回，继续下一次的读取尝试。这便是 **NIO** 。

注：这里的 NIO 并非 Java 中的 NIO，Java 中的 NIO 为 New IO，它实际上是结合了多路复用技术。

伪代码如下：

```java
while (true) {
    for (int i = 0; i < fds.length; i++) {
        if (fds[i].recviced()) {
            fds[i].read();
        }
    }
}
```

![110713xKrHzM4JYws4](https://cdn.jsdelivr.net/gh/lfdotleo/pic2022q1@main/110713xKrHzM4JYws4.jpg)

如果我们把整个 IO 过程分为等待数据阶段和内核数据到用户空间的复制两部分，和 BIO 比第二部分是没有变化的。区别在于第一部分，在等待数据时不阻塞，没有数据立即返回，有数据时才阻塞读取，因此我们只需要一个线程轮询读取数据即可。

NIO 解决了 BIO 的两个缺点：它只用一个线程轮询即可，不需要创建过多线程导致 CPU 频繁的上下文切换；也没有线程过多带来的内存占用不可忽视的情况。那么它就完美了吗？

不，从上图可以看出，NIO 的`recvfrom`虽然能立即返回，但这一次的请求仍然是通过 **系统调用** 的方式，假设我们有 1 万个客户端同时连接服务端，那么一次轮询就需要 1 万次系统调用，大家都知道系统调用会进行用户态和内核态的切换，也是不小的消耗。

总结一下，NIO 解决了 BIO 的 2 个明显的缺点，但它自身依然存在不足：

1. 当客户端连接多时，每次轮询需要多次系统调用，会在用户态和内核态之间切换，带来不小的消耗。
1. 每 2 次轮询的时间间隔不好掌握，时间短容易白忙活；时间长会造成高延迟。

## 多路复用

NIO 解决了 BIO 的问题，但是当客户端连接多时，每次轮询需要很多次系统调用。如果想避免它，我们可以想想有什么办法让这么多次的系统调用**变成 1 次系统调用** ，这样的话问题不就解决了吗？

**多路复用**其实就是由内核提供给几个命令，通过调用它们，我们可以一次性将所有需要监听的客户端文件描述符传递给内核后阻塞，由内核通过自己的方式去监控这些文件描述符，当它们有变化时及时通知我们就可以。这样，我们就解决了 NIO 多次系统调用的不足。

![110713TjBSLMNj0vhi](https://cdn.jsdelivr.net/gh/lfdotleo/pic2022q1@main/110713TjBSLMNj0vhi.jpg)

以 select 为例，等待数据阶段，程序将所有需要监听的 socket 组装，调用系统调用函数 select 后阻塞。当有任一可读时，函数便返回。然后再发起 recvfrom 读取数据。因此很好解决了 NIO 多次系统调用的缺点。

### select

select 函数声明如下：

```c++
#INCLUDE <SYS/TIME.H>
#INCLUDE <SYS/TYPES.H>
#INCLUDE <UNISTD.H>

int select(int maxfd, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```

参数说明：

- readfds、writefds、exceptfds：fd_set 我们可以理解为类似位图。 fd_set 声明的是一个集合，也就是说，readfs、writefds、exceptfds 都是容器，分别为需要监听读、监听写、监听异常的文件描述符，该“位图”在定义时设置最大有 1024 位，因此对文件描述符多少有限制。
- maxfd：是一个整数值，为所有描述符中最大值 +1，它的作用就是遍历文件描述符时减少遍历次数，只遍历 0 到 maxfd 即可。linux 默认文件描述符最大 1024，如果我们的三个 fds 只传入了少数几个文件描述符，指定 maxfd 可以就能大大减少循环次数。
- timeout：可以选择阻塞，可以选择非阻塞，还可以选择定时返回。如果为 NULL，则为阻塞。如果将该结构体里的值设置为 0 则为非阻塞。设置大于 0 时会在超时后返回。

使用 select 的一般步骤：

1. 将 fds 各位全部清空
1. 将 fds 需要监听的文件描述符对应位置为 1
1. 调用 select 函数监听置为 1 的文件描述符是否有数据到来
1. 当状态为 1 的文件描述符有数据到来时，此时你的状态仍然为 1，但是其他状态为 1 的文件描述如果没有数据到来，那么此时会将这些文件描述符置为 0
1. 当 select 函数返回后，只能通过遍历的方式获取准备就绪的文件描述符，然后对该文件描述符做后续操作。

**注意** ：我们只可以通过返回结果获取到有数据到来的文件描述符，但不可以获取到数据，如果需要获取数据，还需要通过该文件描述符去读取。

select 底层一般的执行步骤：

1. 从用户空间拷贝 fd_set 到内核空间
1. 先把所有 fd_set 扫一遍
1. 如果发现有可用的 fd，则调到 5
1. 如果没有，当前线程去 Sleep xx 秒 (schedle timeout 机制）
1. xx 秒后自己醒来，或者状态变化的 fd 唤醒自己，调到 2 重新执行
1. 结束循环，返回（可能因为 timeout 超时返回）

可以看到，虽然 select 也会在内核态和用户态做遍历操作，但遍历时没有进行系统调用了，这会大大提高它的性能。

select 的缺点：

1. fds 默认大小 1024 限制，虽然可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但是这样也会造成效率的降低
1. select 返回时，会修改 fds，因此每次重新调用时需要初始化
1. 得到返回结果后，需要遍历 fds 才能找到有变化的文件描述符，在文件描述符较多时效率不高
1. 每次调用 select，所有的 fds 都需要在内核态和用户态进行拷贝

这部分内容的实践需要用到 C 代码，感兴趣的可以查看这篇文章 [linux select 函数解析以及事例 - 知乎](https://zhuanlan.zhihu.com/p/57518857)，这里只简单理解其原理即可。

### poll

poll 函数声明如下：

```c++
#INCLUDE <POLL.H>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);

 struct pollfd {

       int   fd;  /* file descriptor */

       short events;  /* requested events */

       short revents;  /* returned events */

   };
```

pull 的所有改变都是围绕结构体 pollfd 进行的。

参数说明：

- fds：和 select 的 readfds 等不同的是结构体内部的区别，之前我们说过 select 的 readfds 等类似于位图，poll 中的结构体有三个字段：1) fd，表示文件描述符 2) events，表示关注的事件 3) revents，表示实际发生的事件。
- nfds：用来指定第一个参数数组元素个数。
- timeout：指定等待的毫秒数，无论 I/O 是否准备好，poll() 都会返回。

和 select 的区别：

poll 和 select 的区别并不大，不过是通过这种链表的形式，可以传递近乎无限的文件描述符。并且调用后返回的结果，会写入到 revents 字段，不会影响请求时传入的 events 字段，将 revents 字段抹除后就可以就可以再次调用。

- 解决了 select 文件描述符多少的限制。
- 通过特定字段返回数据，不会影响请求数据，数据可复用。

但它依然存在问题：

- 需要在内核态和用户态之间进行拷贝。
- 需要在内核态和用户态中遍历。

### epoll

epoll 共有 3 个函数，分别为 `epoll_create` , `epoll_ctl` 和 `epoll_wait`, 下面我们分别聊一下这三个函数。

#### epoll_create

```c
int epoll_create(int size);
```

创建一个 epoll 的句柄。在 `epoll_create` 的最初实现版本时， size 参数的作用是创建 epoll 实例时候告诉内核需要使用多少个文件描述符。内核会使用 size 的大小去申请对应的内存（如果在使用的时候超过了给定的 size， 内核会申请更多的空间）。现在，这个 size 参数不再使用了（内核会动态的申请需要的内存）。但要注意的是，这个 size 必须要大于 0，为了兼容旧版的 linux 内核的代码。

#### epoll_ctl

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
```

函数是对指定描述符 fd 执行 op 操作。其中参数分别代表：

- epfd: 是 `epoll_create` 的返回值。
- op: 表示 op 操作，用三个宏来表示：添加 EPOLL_CTL_ADD，删除 EPOLL_CTL_DEL，修改 EPOLL_CTL_MOD。分别添加、删除和修改对 fd 的监听事件。
- fd：是需要监听的 fd（文件描述符）。
- epoll_event：是告诉内核需要监听什么事件，struct epoll_event 结构如下：

```c
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};

//events 可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端 SOCKET 正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将 EPOLL 设为边缘触发 (Edge Triggered) 模式，这是相对于水平触发 (Level Triggered) 来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个 socket 的话，需要再次把这个 socket 加入到 EPOLL 队列里
```

#### epoll_wait

```c
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

等待 epfd 上的 io 事件，最多返回 maxevents 个事件。

- epfd: `epoll_create` 的返回值，在 `epoll_ctl` 中也有用到。
- events: 用来从内核得到事件的集合。
- maxevents: 告诉内核返回事件集合的最大条数。
- timeout: 在该时间内没有事件发生，则超时返回。

#### 边缘触发和水平触发

在介绍 `epoll_ctl` 时，结构体 `epoll_event` 的 `events` 宏定义中，EPOLLET 代表边缘触发 (Edge Triggered) 模式。与之相对的是水平触发 (Level Triggered)，它们之间的区别在于：

- **LT 模式** ：当 `epoll_wait` 检测到描述符事件发生并将此事件通知应用程序，**应用程序可以不立即处理该事件** 。下次调用 `epoll_wait` 时，会再次响应应用程序并通知此事件。
- **ET 模式** ：当 `epoll_wait` 检测到描述符事件发生并将此事件通知应用程序，**应用程序必须立即处理该事件** 。如果不处理，下次调用 `epoll_wait` 时，不会再次响应应用程序并通知此事件。

水平触发是只要读缓冲区有数据，就会一直触发可读信号，而边缘触发仅仅在空变为非空的时候通知一次。举个例子：

1. 读缓冲区刚开始是空的
1. 读缓冲区写入 2KB 数据
1. 水平触发和边缘触发模式此时都会发出可读信号
1. 收到信号通知后，读取了 1kb 的数据，读缓冲区还剩余 1KB 数据
1. 水平触发会再次进行通知，而边缘触发不会再进行通知

```c
//水平触发
ret = read(fd, buf, sizeof(buf));

//边缘触发（代码不完整，仅为简单区别与水平触发方式的代码）
while(true) {
    ret = read(fd, buf, sizeof(buf);
    if (ret == EAGAIN) break;
}

```

## 总结

该部分总结主要源于阅读了 UNIX 网络编程卷 1：套接字联网 API（第三版）（大约 122 页之后的一部分内容）觉得有了更深层次的理解，所以根据该书的观点和上述内容做一个总结，让大家进一步理解它们之间的区别。

### 第一种理解思路

不论 BIO、NIO、多路复用，都围绕着两个问题进行设计：

1. 创建太多线程带来的性能消耗
1. 多次系统调用带来的性能消耗

下面来看看这些 IO 模型如何解决这两个问题：

- BIO，因为每次的读取都会阻塞，我们不得不创建多个线程来请求，创建多线程会导致：1）线程本身占用较多资源 2）线程切换导致上下文切换带来性能消耗 3）线程是有限的，因此对请求数有限制
- NIO，因为 NIO 在没有可读内容时立即返回，所以我们可以通过一个线程遍历需要读取的请求而不创建多线程，但这也会带来：1）多次系统调用带来性能消耗 2）当没有读取到内容时，不能精确控制下次调用时机导致大量不必要调用 3）可能某个读取耗时较多时，会导致延时
- 多路复用，多路复用解决了这两个问题。通过一个线程注册多个读取请求，该线程阻塞等待（只有一个线程），如果有内容可读取则返回。此时再去读取，肯定可以获取到内容。
  - select, 通过 readfds、writefds 和 exceptfds 每次传递所有需要监听的各种类型的 fds。存在以下问题：1）每次需要在用户态和内核态拷贝所有的 fds 2）在用户态和内核态如果要获得有事件的 fds，需要遍历集合 3）fds 类似于位图，有长度限制，相应的 fd 有个数限制（默认 1024) 4）内核态会置零 fds，每次都需要重新设置
  - poll, 通过 pollfd 的定义，解决了 select 因为类似于位图的结构导致的个数限制，且 pollfd 中有 reevents 来供内核态设置事件，每次只需要清理这个字段就可以，不需要重新设置 pollfd。但仍然存在两个问题：1）每次需要在用户态和内核态拷贝所有的 fds 2）在用户态和内核态如果要获得有事件的 fds，需要遍历集合
  - epoll, 通过 3 个方法，将事件监听交给内核记录，有事件发生时，也仅返回发生的事件集合。因此解决了用户态和内核态拷贝所有 fds ，用户态和内核态需要遍历 fds 这两个问题。

### 第二种理解思路

![110713zr8QSSR1MsY1](https://cdn.jsdelivr.net/gh/lfdotleo/pic2022q1@main/110713zr8QSSR1MsY1.jpg)

我们可以认为所有的系统 I/O 都分为两个阶段：等待就绪和操作。

- BIO: 等待就绪和操作整个过程都是阻塞状态。
- NIO: 在等待数据阶段非阻塞进行多次检查，数据就绪后阻塞读取。
- 多路复用：通过一个线程阻塞等待所有需要监听的文件描述符数据就绪，如果有任一数据就绪，则再阻塞读取。
- 信号驱动：可以认为是向系统注册了关注的事件，当事件发生时，系统通过发送信号的方式通知启动 I/O 操作。这个看了很多知乎的比较，比较靠谱的一种认为，多路复用基于事件驱动来实现，事件驱动是多路复用的核心机制。
- 异步 IO: 也是信号驱动的一种实现，事件发生时，内核将数据拷贝到调用时指定的内存空间中，然后发送信号。

## 参考内容

1. [【并发】IO 多路复用 select/poll/epoll 介绍_哔哩哔哩 （゜ - ゜） つロ 干杯~-bilibili](https://www.bilibili.com/video/BV1qJ411w7du?t=11)
1. UNIX 网络编程卷 1：套接字联网 API（第三版）- 122 页
1. [linux select 函数解析以及事例 - 知乎](https://zhuanlan.zhihu.com/p/57518857)
1. [Java NIO 浅析 - 美团技术团队](https://tech.meituan.com/2016/11/04/nio.html)
1. [如何深刻理解 Reactor 和 Proactor？ - 知乎](https://www.zhihu.com/question/26943938/answer/68773398)
1. [Linux IO 模式及 select、poll、epoll 详解 - 人云思云 - SegmentFault 思否](https://segmentfault.com/a/1190000003063859?utm_source=Weibo&utm_medium=shareLink&utm_campaign=socialShare&from=timeline&isappinstalled=0)
1. [epoll 边缘触发与水平触发 - 掘金](https://juejin.im/post/6844903878685622285#heading-5)
1. [Linux 惊群效应详解（最详细的了吧）_lyztyycode 的博客 - CSDN 博客_惊群效应](https://blog.csdn.net/lyztyycode/article/details/78648798)
