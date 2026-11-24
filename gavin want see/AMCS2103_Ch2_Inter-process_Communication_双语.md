# AMCS2103 — Chapter 2: Inter-process Communication (IPC)（双语版）

> **怎么读：** 讲解用华文，terms 保留英文；**💬 答题句 (EN)** = 考试直接写的英文句子。
>
> Slides 列了很多模型（client–server、P2P、sockets、pipes、RPC），但讲师真正问的 pipe 问题——*buffer 空了怎样？读到什么时候？怎样让 consumer 停下来等？*——slides 没回答。这份笔记用实际 compile 跑过的程序来回答。两个 pipe 程序和 UDP client/server 都逐行解释。目录有 "Middleware" 但没有 slide，这里补了简短定义，并标明是额外内容。

---

## 0. 本章总览

**一句话：** 合作的 process 要互相传 data，方法有两大类——**送讯息**或**共用 memory**；这章从底层（sockets、pipes）讲到高层（RPC/RMI）。

| # | 主题 | Slides | 考试重点 |
|---|---|---|---|
| 1 | Independent vs cooperating processes、IPC 两种模型 | 3–9 | |
| 2 | Client–server、broadcast、multicast、peer-to-peer | 10–17 | |
| 3 | Sockets、ports、UDP、client/server 代码 | 18–30 | |
| 4 | **Pipes（ordinary & named）** | 31–35 | ⭐⭐ buffer 空、读到什么时候、怎样 block consumer |
| 5 | Message passing、MPI、RPC/RMI、middleware | 36–41 | |

**先备知识：** 每个 *process* 有自己私有的 memory（**address space**，地址空间），所以一个 process **不能**直接读另一个 process 的变量——这正是需要 IPC 的原因。`fork()` 会建立一个 child process，它是 parent 的复制品。

---

## 开场
你在 Linux 打 `cat log.txt | grep ERROR`。两个程序、两个 process，可是 `cat` 输出的文字直接流进 `grep`。Data 是怎么从一个 process 的 memory 跑到另一个的？如果 `cat` 还没输出东西，`grep` 会怎样？

---

## 1. 为什么 process 需要 IPC（slides 3–9）

**Independent processes**（独立进程）（slides 3–4）：不会影响别人，也不会被别人影响；虽然同时跑，但**不共享资源**。
**Cooperating processes**（协作进程）（slide 5）：会影响别人，也会被别人影响。

**Process 合作的好处（slide 5）**——记「**I C M C**」：

| Advantage | 中文意思 |
|---|---|
| **Information sharing** | 信息共享：几个用户可能要读同一个 file |
| **Computation speedup** | 加速计算：一个工作拆成几个同时进行的小工作 |
| **Modularity**（模块化） | 系统功能拆成不同的 process/thread |
| **Convenience** | 方便：用户可以同时做几件事，例如编辑、compile、打印 |

**IPC（slide 6）：** 合作的 process 需要 IPC 机制来交换 data。两种基本模型：

```
      Message passing (MPI)                Shared memory (OpenMP)
   +-------------+                      +-------------+
   |  process A  |--+                   |  process A  |<-+
   +-------------+  |                   +-------------+  |
   |  process B  |<-|-+                 |shared memory|<-+--+
   +-------------+  | |                 +-------------+     |
   |             |  | |                 |  process B  |<----+
   +-------------+  | |                 +-------------+
   |message queue|<-+ |                 |             |
   |m0|m1|...|mn |----+                 +-------------+
   +-------------+                      |   kernel    |
   |   kernel    |                      +-------------+
   +-------------+
```
**看点（slide 7）：** message passing 的 data 要**经过 kernel 的 message queue**；shared memory 是 process 直接读写一块**共同的区域**。

| | Message passing | Shared memory |
|---|---|---|
| How | Implemented with **system calls**（系统调用）— time-consuming kernel intervention | Processes read/write the shared region directly |
| Speed | Slower | **Faster** — kernel not required after setup |
| Danger | — | Must ensure processes don't write the **same location simultaneously** (race condition) |
| Example API | MPI | OpenMP |

Slide 8（OpenMP shared-memory model）：一个 process 里多个 threads，全部共用 main memory；任何 thread 都能读写共享 data；适合 multi-core CPU。

**💬 答题句 (EN):** *Message passing exchanges data through send/receive system calls handled by the kernel, which is simpler to synchronise but slower; shared memory lets processes read and write a common memory region directly, which is faster but requires synchronisation to prevent simultaneous writes.*

---

## 2. Communication models（slides 10–17）

### Client–server（slides 11–14）
应用程序的逻辑分成两部分：
- **Server** – **提供**计算服务的 process；通常放在专用、强大的电脑（memory 多、CPU 快），延迟低。
- **Client** – **请求**服务的 process；通常代表一个人类用户。

**Model 1（slide 12）：**
```
  Client A ──Connection 1──► [Server S] ◄──Connection 2── Client B
          「请求服务」          「提供服务」
```
- 连接是一个 client 和一个 server **之间私有的**。
- 一台 server 可以**同时服务很多 client**。
- 需要服务时由 client **主动发起**。
- **所有沟通都经过 server**，client 之间不直接沟通。
- 例子：你输入 www.tarc.edu.my → browser 发请求 → server 检查*要哪一页？*、*这个用户有没有权限？* → 回传网页。

**Model 2 – two-tier（slides 13–14）**（database 应用常用；client tier + server tier）：
```
 Client A ──DB access request──► [Database Server] ──auth request──► [Authentication Server]
          ◄────── Result ───────                   ◄── Request accepted ──

 Client B ──DB access request──► [Database Server] ──auth request──► [Authentication Server]
          ◄────── Refused ──────                   ◄── Request denied ──
```
组成：**database server**（存 DB + 存取/更新逻辑）、**database clients**（用户界面）、**authentication service**（验证用户身份）。看点：DB server 本身也是 authentication server 的 *client*——client–server 可以有好几层。

### Broadcast vs Multicast（slides 15–16）

| | Broadcast（广播） | Multicast（组播） |
|---|---|---|
| Delivered to | **All** available receivers | A **selected subset** (multicast group) |
| Security | **Insecure** — any process on the port hears it | Better — only members receive |
| Simplicity | **Simple** — sender needn't know who/how many | **More complex** — devices must join/leave groups; routers need multicast support |
| Efficiency | **Lower** if many devices don't need the message | **More efficient** for group communication |

### Peer-to-peer（slide 17）
```
 Process A ──(time T0)──► Process B ◄──(time T1)── Process C
```
- **没有中央 server**；peer 之间直接沟通、共享资源。
- **不需要专用 server** → 成本低；小网络**容易架设**。
- **难管理**（没有中央控制）；**安全性**难以统一执行。

---

## 3. Sockets（slides 18–30）

### UDP（slides 19–20）
UDP datagram（数据报）：从发送方送到接收方，**没有确认（acknowledgement）也不会重传**；出错的话讯息可能就到不了。
特征：**connectionless（无连接）、unreliable（不可靠）、lightweight（轻量）、point-to-point (unicast)、broadcast**。
适合**偶尔丢失讯息也没关系**（omission failure，遗漏故障）的服务：**DNS**、**VoIP**。

（TCP 在 slide 18 有列但没讲。额外补充：TCP 是 connection-oriented、reliable——有确认、会重传、按顺序送达——用在网页、email。）

### Socket 和 port（slides 21–22）
- **Socket** – 两台机器之间沟通的端点（endpoint）。
- **Port**（端口）– 逻辑通道，用来分辨电脑里的**哪个应用/服务**。HTTP → 80，HTTPS → 443，email（SMTP）→ 25。
- Client socket **自动分配**一个空的 port；server socket **监听一个固定、大家知道的 port**（agreed port）。
- 一条连接由一对地址决定：**(Client IP, Client Port) ↔ (Server IP, Server Port)**，例如 (138.37.94.248, **49520**) ↔ (138.37.88.249, **80**)。
- Socket 是一套在 transport layer 用 UDP/TCP 沟通的基本函数——**最灵活（最底层）**，但开发者要自己处理沟通细节。

### 实作：UDP "Hello World" 自动回复（slides 23–30）
要思考的问题：*server 怎么自动回复？* → server 在 `recvfrom()` 时记下 client 的地址，再用这个地址 `sendto()`。

```
   CLIENT                        SERVER
   socket()                      socket()
                                  bind()      （绑到 127.0.0.1:5000）
   sendto("Hello World") ──────► recvfrom()
   recvfrom()  ◄────────────────  sendto("Hello Client")
   close()                        继续等（while(1)）
```

**Server（slides 25–27）：**
```c
int serverSocket;
char buffer[1024];                       // 存收到的讯息
char reply[] = "Hello Client";
struct sockaddr_in serverAddress;        // server 自己的地址
struct sockaddr_in clientAddress;        // recvfrom() 会填进来：谁送的
socklen_t clientLength = sizeof(clientAddress);

serverSocket = socket(AF_INET, SOCK_DGRAM, 0);   // IPv4、UDP、默认 protocol
if (serverSocket < 0) { printf("Socket creation failed.\n"); return 1; }

serverAddress.sin_family = AF_INET;
serverAddress.sin_port = htons(5000);            // port 5000（network byte order）
serverAddress.sin_addr.s_addr = inet_addr("127.0.0.1");

if (bind(serverSocket, (struct sockaddr *)&serverAddress, sizeof(serverAddress)) < 0) {
    printf("Bind failed.\n"); return 1;          // bind = 把 socket 绑到 IP+port
}
printf("Server is waiting...\n");
while (1) {                                      // 持续服务 client
    int bytesReceived = recvfrom(serverSocket, buffer, sizeof(buffer) - 1, 0,
                         (struct sockaddr *)&clientAddress, &clientLength);
    if (bytesReceived > 0) {
        buffer[bytesReceived] = '\0';            // 加上字符串结尾
        printf("Received from client: %s\n", buffer);
        sendto(serverSocket, reply, strlen(reply), 0,       // ← 自动回复
               (struct sockaddr *)&clientAddress, clientLength);
        printf("Reply sent: %s\n", reply);
    }
}
close(serverSocket);
```
讲师标注的重点：
- `sizeof(buffer) - 1` → 最多读 1023 bytes，**留一个 byte 给 `'\0'`**。
- `buffer[bytesReceived] = '\0'` → "Hello World" 占 index 0–10，`'\0'` 放在 index 11。
- 没有 `while(1)` 的话，server 收一条讯息就结束。

**Client（slides 28–30）：**
```c
char message[] = "Hello World";
clientSocket = socket(AF_INET, SOCK_DGRAM, 0);
serverAddress.sin_family = AF_INET;
serverAddress.sin_port = htons(5000);
serverAddress.sin_addr.s_addr = inet_addr("127.0.0.1");
sendto(clientSocket, message, strlen(message), 0,
       (struct sockaddr *)&serverAddress, sizeof(serverAddress));
printf("Sent to server: %s\n", message);
int bytesReceived = recvfrom(clientSocket, buffer, sizeof(buffer) - 1, 0,
                    (struct sockaddr *)&serverAddress, &serverLength);
if (bytesReceived > 0) {
    buffer[bytesReceived] = '\0';                // "Hello Client" = 0..11，'\0' 在 12
    printf("Server replied: %s\n", buffer);
}
close(clientSocket);
```
✔ **我在 Linux 用 gcc 跑过：**
```
Server is waiting...
Received from client: Hello World
Reply sent: Hello Client
Sent to server: Hello World
Server replied: Hello Client
```
注意：`serverLength` 存的是**地址结构的大小**，不是讯息的大小。Slide 30 写 `(structsockaddr*)`，少了空格，应该是 `(struct sockaddr *)`。

---

## 4. Pipes ⭐⭐（slides 31–35）

### Pipe 是什么（slide 31）
**Pipe**（管道）是 Linux 里在 process 之间传 data 的 IPC 机制。它用一块 **kernel-managed buffer**（由 kernel 管理的缓冲区）：一个 process **写**，另一个 process **读**。可以想成 OS 里面一个临时的**信箱**。

| | **Ordinary pipe**（slide 32） | **Named pipe / FIFO**（slide 33） |
|---|---|---|
| Direction | **Unidirectional**（单向） | 一般一次也是单向使用；双向通常开两个 |
| Model | Standard **producer–consumer** | More powerful |
| Relationship | Requires **parent–child** relationship | **No** parent–child needed — unrelated processes can use it |
| Name | No name; only via file descriptors inherited through `fork()` | Has a **name in the file system** (e.g. `myfifo`) |
| Lifetime | **Ceases to exist** when communication is over and processes terminate | Stays until **explicitly deleted** (`unlink`) |
| Created by | `pipe(fd)` | `mkfifo(name, 0666)` |

> 中文提示：ordinary pipe 为什么一定要 parent–child？因为它**没有名字**，别的 process 找不到它，只能靠 `fork()` 把 fd[0]/fd[1] **继承**给 child。

用 ordinary pipe 做双向沟通要开**两条 pipe**（slide 31 的图：Pipe A parent→child，Pipe B child→parent）。

### Ordinary pipe 程序（slide 34）
```c
 1 #include <stdio.h>
 2 #include <unistd.h>
 3 int main() {
 4     int fd[2];                 // fd[0] = 读的那一端，fd[1] = 写的那一端
 6     pipe(fd);                  // OS 建立 kernel buffer（「信箱」）
 8     if (fork() == 0) {         // ---- Child process ----
10         close(fd[1]);          // child 只读 → 关掉写端
12         char msg[20];
14         read(fd[0], msg, sizeof(msg));   // child 读（没 data 就等）
16         printf("Child received: %s\n", msg);
18         close(fd[0]);
19     }
21     else {                     // ---- Parent process ----
22         close(fd[0]);          // parent 只写 → 关掉读端
23         write(fd[1], "Hello Pipe", 11);  // 10 个字 + '\0' = 11 bytes
24         close(fd[1]);
25     }
26     return 0;
}
```
```
 Parent process               Child process
 close(fd[0])                 close(fd[1])
 write(fd[1], ...)   ──►PIPE──►  read(fd[0], ...)
 Message "Hello Pipe"         Child received: Hello Pipe
```
✔ 跑过 → `Child received: Hello Pipe`。

### Named pipe 程序（slide 35）
```c
const char *fifo = "myfifo";
mkfifo(fifo, 0666);              // 1. 建立 FIFO 文件；0666 = 所有人可读写
pid_t pid = fork();
if (pid < 0) { perror("fork failed"); exit(1); }
if (pid == 0) {                  // CHILD（读方）
    char buffer[100];
    int fd = open(fifo, O_RDONLY);       // 用「只读」打开
    read(fd, buffer, sizeof(buffer));
    printf("Child received: %s\n", buffer);
    close(fd);                           // 用完释放 descriptor
} else {                         // PARENT（写方）
    sleep(1);                            // 「确保 child 准备好」
    int fd = open(fifo, O_WRONLY);       // 用「只写」打开
    write(fd, "Hello from parent using named pipe", 35);
    close(fd);
}
unlink(fifo);                    // 4. 删除 FIFO 文件（建议要做）
```
✔ 跑过 → `Child received: Hello from parent using named pipe`。

### ⭐ 讲师的三个 pipe 问题

**Q1. Buffer 空了会怎样？（What if the buffer is empty?）**
**没东西可读**，所以 `read()` 会 **block**（阻塞）：OS 让读的 process 去睡觉，**等（被延迟）**到写方把 data 放进 pipe 为止。它不会回传 error，也不会一直空转（busy-wait）。

> ✔ 测试：parent 等 2 秒才写。Child 的 `read()` **2 秒后**才返回并拿到 data（`read returned 5 after 2 s: late`）。

**Q2. Process 会读到什么时候？（The process will read until…?）**
`read(fd, buf, n)` 只要**有一些 data** 就会返回——最多 `n` bytes（pipe 里有多少就拿多少，不超过 n）。如果用 loop 一直读，就会读到 **EOF（end-of-file，文件结束）**：当 pipe **已经空了，而且所有写端都关掉了**，`read()` 返回 **0**。

所以每个 process 都要 `close()` 自己不用的那一端——如果 child 忘了 `close(fd[1])`，就还有一个写端开着，child 最后那次 `read()` 会**永远等下去**，看不到 EOF。

> ✔ 测试：parent 关掉 `fd[1]` 之后，child 第二次 `read()` 返回 **0（EOF）**。

**Q3. 怎样 block consumer？（How to block the consumer?）**
**不用特别做什么——默认就是 blocking：**
- `pipe()` 开出来的 ordinary pipe 是 **blocking mode**，consumer 在空 pipe 上 `read()` 会自动**等到 producer 写**。
- Named pipe 的话，`open(fifo, O_RDONLY)` 本身就会**等到有另一个 process 用写的方式打开这个 FIFO**；之后 `read()` 再等 data 来。
- 反过来也一样：pipe buffer **满了**，producer 的 `write()` 会等到 consumer 读走一些——这就是 kernel 帮你做的 producer–consumer 同步。
- （额外）用 `O_NONBLOCK` 打开就会关掉这个行为：空 pipe 上 `read()` 立刻返回 −1（EAGAIN），不等。

**✅ 满分答法 — "Explain what happens when a consumer reads from an empty ordinary pipe and how the consumer is blocked." (4–6 marks)**
*An ordinary pipe is a unidirectional kernel buffer shared by a parent and child process: the producer writes to fd[1] and the consumer reads from fd[0]. If the buffer is empty there is nothing to read, so the consumer's read() system call blocks — the kernel suspends the consumer and delays it until the producer writes data, then read() returns the available bytes. The consumer keeps reading until read() returns 0 (EOF), which happens when the pipe is empty and all write ends are closed; therefore each process must close the end it does not use. Blocking is the default behaviour of pipes, so no busy-waiting is required; similarly the producer blocks if the buffer is full.*

> 中文重点：**空 → read 阻塞等待；读到 → read 返回 0（EOF）= 空 + 写端全关；blocking 是默认的。**

**💬 答题句 (EN)（ordinary vs named）:** *An ordinary pipe is unidirectional, unnamed, requires a parent–child relationship and disappears when the processes terminate; a named pipe (FIFO) has a name in the file system, can be used by unrelated processes, and persists until it is explicitly deleted.*

---

## 5. Message passing、RPC/RMI、middleware（slides 36–41）

### Message passing（slide 37）
**简单、没有结构**的沟通方式——讯息从发送方传到接收方；概念上跟 **UDP datagram** 很像。
**MPI** 是其中一种具体实现，在 parallel processing 很流行，比 UDP 多了很多功能，用来支持平行程序的**同步**（synchronisation）。

### Message-passing model（slide 38）
```
 Thread 1 address space                        Thread 2 address space
  [ X = 50 ]  send(X, 2, my_msg_id) ──●──►  recv(Y, 1, my_msg_id)  [ Y ]
                         逻辑上的会面 / 交换讯息
```
- 每个 thread/process 在**自己私有的 address space** 里运作。
- `send`：指定**收件人、要送的 buffer、可选的 tag**。`send(X, 2, my_msg_id)` = 把本地 X 的内容送给 thread 2，tag 是 my_msg_id。
- `receive`：指定**发件人、存 data 的 buffer、可选的 tag**。`recv(Y, 1, my_msg_id)` = 从 thread 1 收 tag 为 my_msg_id 的讯息，存进 Y。
- 送讯息是它们交换 data 的**唯一方法**。**Tag** 帮系统辨认和配对讯息。

**Message passing systems（slide 39）：** 硬件不需要全系统共享的 load/store，只要能送讯息，所以**普通的商用电脑（commodity computers）**就能连成大型平行机器（workstation cluster；IBM Blue Gene/P 用 InfiniBand）。Message passing 是 **cluster** 的编程模型。

### RPC 和 RMI（slides 40–41）
**Remote Procedure Call（RPC，远程过程调用）：** 给 **procedural** 语言用（C、Fortran、Pascal）。一个 process 调用**另一个** process 里的 procedure。系统提供 **proxies/stubs**（代理/存根），所以调用方其实是在做一个**本地调用**；被调用方也以为调用方在本地。这大大简化了开发——不用自己处理沟通细节。
**Remote Method Invocation（RMI，远程方法调用）：** RPC 的 **object-oriented** 版本（Java、C++）——调用另一个 process 里某个 object 的 **method**。

```
 Caller process                                   Called process
 +-------------------+                          +-------------+
 | main thread       |  local procedure call    |             |
 |   |---► stub ●===( RPC/RMI mechanism )===● stub ─► remote   |
 |   |   (proxy)     |   处理沟通细节            |   procedure |
 |   ▼  remote call  |                          |             |
 +-------------------+                          +-------------+
```
**看点：** 两边的 stub 让远程调用**看起来像本地调用**——这就是 Ch1 讲的 **access/location transparency**。

**💬 答题句 (EN):** *In RPC/RMI the caller invokes a local stub (proxy) which packs the parameters and sends them over the network to the stub in the remote process; that stub calls the real procedure and returns the result the same way, so the remote call appears to the programmer like a local call.*

### Middleware（目录 slide 2 有列，但没有 slide；额外补充）
Middleware（中间件）是夹在 OS/网络和应用程序之间的一层软件，把分布式系统的沟通细节和 heterogeneity（异构性，各机器不一样）藏起来，提供 RPC/RMI、messaging、naming 等共用服务。RPC/RMI framework 和 MPI 都算 middleware。*（Slides 没有，被问到才用。）*

*EN version:* Middleware is a software layer between the operating system/network and the applications that hides communication details and heterogeneity, providing common services such as RPC/RMI, messaging and naming.

---

## 回到开场
`cat log.txt | grep ERROR` 就是一条 **ordinary pipe**：shell 先 `pipe()`，再 fork 出两个程序；`cat` 写进写端，`grep` 从读端读。`cat` 慢的话，`grep` 的 `read()` 就 **block** 等着；`cat` 做完关掉写端，`grep` 看到 **EOF** 就结束。Pipe 只能在同一台机器用；**不同**机器之间就用 sockets，想把 socket 细节藏起来就用 RPC/RMI 或 MPI。下一章：process **共用 memory** 时会怎样。

---

## ⚠️ Slides 有问题或漏掉的地方

| Slide | Slide 写的 / 做的 | 正确理解 |
|---|---|---|
| 2 | 目录列了 "Middleware" | 没有 middleware 的 slide，见上面的补充定义 |
| 18 | 列了 TCP | 完全没讲 TCP——要知道：connection-oriented、reliable、ordered |
| 30 | `(structsockaddr*)` | 少了空格：`(struct sockaddr *)` |
| 33 | Named pipe「more powerful」 | 还要知道：它是 file system 里的一个文件，`unlink()` 之前一直存在 |
| 35 | `sleep(1)`「确保 child 准备好」 | 正确性不需要它：FIFO 的 `open()` 本来就会等到读方和写方都到齐 |

---

## 术语表

| English | 中文 | 一句话说明 |
|---|---|---|
| Inter-process communication | 进程间通信 | 在 process 之间交换 data |
| Cooperating process | 协作进程 | 会影响别人/被别人影响的 process |
| Address space | 地址空间 | 一个 process 私有的 memory |
| System call | 系统调用 | 向 OS kernel 发出的请求 |
| Broadcast / multicast | 广播 / 组播 | 送给所有人 / 送给一个群组 |
| Datagram | 数据报 | 一个独立的 UDP 封包 |
| Connectionless | 无连接 | 送之前不用先建立连接 |
| Omission failure | 遗漏故障 | 讯息丢了 |
| Port | 端口 | 代表某个服务的号码 |
| Bind | 绑定 | 把 socket 绑到 IP + port |
| Pipe / FIFO | 管道 / 命名管道 | process 之间的 kernel buffer |
| File descriptor | 文件描述符 | 代表打开的文件/端口的整数（fd[0]、fd[1]） |
| Block | 阻塞 | 被 OS 暂停，等待 |
| EOF | 文件结束 | read() 返回 0：没 data 了，写端都关了 |
| Stub / proxy | 存根 / 代理 | 代替远程 procedure 的本地替身 |
| Middleware | 中间件 | 隐藏分布细节的软件层 |

## 速记卡
- 合作的好处：information sharing、computation speedup、modularity、convenience。
- Message passing（经过 kernel、较慢、MPI）vs shared memory（直接读写、较快、要同步、OpenMP）。
- Client–server：client 主动发起；所有沟通经过 server。P2P：没 server、便宜、难管理/难保安全。
- Broadcast = 全部、简单、不安全。Multicast = 群组、有效率、较复杂。
- UDP：无连接、不可靠、轻量；DNS、VoIP。Port：HTTP 80、HTTPS 443、SMTP 25。
- UDP server：socket → bind → recvfrom → sendto（送回 recvfrom 拿到的地址）。
- Ordinary pipe：`pipe(fd)`，fd[0] 读 / fd[1] 写，parent–child，单向，临时。
- Named pipe：`mkfifo`，不相关的 process 也能用，用 `unlink` 删除。
- 空 pipe → `read()` 阻塞；读到 EOF（返回 0）= 空 + 写端全关；blocking 是默认的。
- RPC（procedural）/ RMI（OO）——stub 让远程调用看起来像本地调用。

---

## 练习题（附答案）
*题目用英文，答案附华文说明。*

### A. MCQ
1. Which statement about ordinary pipes is FALSE? (a) unidirectional (b) need parent–child relationship (c) persist after processes end (d) created with `pipe()`
2. In `pipe(fd)`, the read end is (a) fd[1] (b) fd[0] (c) fd[2] (d) the file name
3. Which service most suits UDP? (a) online banking transfer (b) file download (c) VoIP call (d) email
4. A consumer calls `read()` on an empty ordinary pipe whose write end is still open. It (a) returns 0 (b) returns −1 (c) blocks (d) crashes
5. RMI differs from RPC because RMI (a) uses UDP only (b) is object-oriented (c) needs no stubs (d) works only locally

**答案：** 1-c；2-b；3-c；4-c（写端还开着，所以不是 EOF，而是等）；5-b。

### B. Short answer
1. Give TWO differences between broadcast and multicast.
   *Broadcast delivers to all receivers, multicast only to group members; broadcast is simple but insecure/inefficient, multicast is efficient but more complex (join/leave groups, router support).*
2. Why must the child close `fd[1]` in the ordinary pipe program?
   *So that once the parent closes its write end there are no write ends left, allowing the child's read() to return 0 (EOF); otherwise the child could block forever. It also enforces one-way use.*
   *华文：不关的话写端永远还有一个开着，child 永远等不到 EOF。*
3. State TWO advantages and TWO disadvantages of the peer-to-peer model.
   *Adv: no dedicated server (lower cost); easy to set up for small networks. Disadv: harder to manage (no central control); security hard to enforce consistently.*

### C. Application
1. Write the order of system calls on the UDP server and client to exchange "Hello World"/"Hello Client".
   *Server: socket → bind → recvfrom → sendto → (loop). Client: socket → sendto → recvfrom → close.*
2. Trace: parent writes "ABC" (4 bytes) at t = 3 s, closes write end at t = 4 s. The child called `read(fd[0], buf, 20)` at t = 0 and loops. What does each `read()` return and when?
   *1st read blocks from t = 0 until t = 3, returns 4 ("ABC\0"). 2nd read blocks until t = 4, when the last write end is closed, then returns 0 (EOF) — loop ends.*
   *华文：第一次等到有 data 才返回；第二次等到写端关掉才返回 0。*
3. Explain how the UDP server knows where to send the automatic reply.
   *recvfrom() fills `clientAddress` with the sender's IP and port; the server passes that same structure to sendto(), so the reply goes back to whichever client sent the message.*

### D. Thinking
1. Two unrelated programs started from different terminals must exchange data on one machine. Ordinary or named pipe? Why?
   *Named pipe: ordinary pipes need a parent–child relationship to share file descriptors; a FIFO has a file-system name that any process can open.*
2. Why is shared memory faster than message passing, yet riskier?
   *After setup, processes access the region directly without kernel system calls for every exchange; but nothing stops two processes writing the same location at once, so they need synchronisation (Ch4) to avoid race conditions.*

---

## Slide 索引
| 章节 | Slides |
|---|---|
| 1 IPC models | 3–9 |
| 2 Client–server、broadcast、multicast、P2P | 10–17 |
| 3 Sockets / UDP code | 18–30 |
| 4 Pipes | 31–35 |
| 5 Message passing、RPC/RMI | 36–41 |

## 跟其他章的连接
- Ch1：MPI send/recv、loosely coupled systems；transparency（RPC 提供 access/location transparency）。
- Ch3：shared-memory 编程（OpenMP）和 MPI 的 synchronous/asynchronous messaging。
- Ch4：bounded buffer 的 producer–consumer——跟 pipe「空了就等、满了就等」是同一个概念，只是用 semaphore 来做。
