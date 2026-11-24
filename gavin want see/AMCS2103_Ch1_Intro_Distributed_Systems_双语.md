# AMCS2103 — Chapter 1: Introduction to Distributed Systems（双语版）

> **怎么读这份笔记：** 讲解用华文，technical terms 保留英文（考卷上就是这些字）。每个重点后面有 **💬 答题句 (EN)**：考试时直接写这几句英文就可以。表格保留英文，方便直接抄进答案。
>
> 这份笔记补了什么：slides 图很多但很少讲「为什么」。这里把每个概念的道理讲清楚，把讲师的表格和例题都抄齐，Cristian 和 Berkeley 都有完整算式，还改正了 slide 64 的一个错误答案。

---

## 0. 本章总览

**一句话：** 一台电脑不够用，就把工作分给很多台电脑。好处是可以扩展、更可靠、更快；坏处是会有新问题：机器会坏、copy 要保持一致、要找到资源、大家的时间要对齐。

| # | 主题 | Slides | 考试重点（讲师 hint） |
|---|---|---|---|
| 1 | Traditional vs distributed database | 3–6 | ⭐ |
| 2 | Benefits：scalability（vertical/horizontal）、reliability、performance、geography | 7–13 | ⭐ |
| 3 | Challenges (a)–(e) | 14–19 | |
| 4 | Loosely vs tightly coupled、MPI、OpenMP | 20–43 | ⭐⭐（pg 20、38/43 的表） |
| 5 | OpenMP vs CUDA vs MPI | 44–50 | ⭐⭐（pg 40/45 的表） |
| 6 | Architecture models + transparency | 51–53 | |
| 7 | Distributed real-time systems | 54–59 | |
| 8 | Clock sync：**Cristian's algorithm**（要 show steps）、Berkeley | 60–69 | ⭐⭐ |
| 9 | Distributed OS：NOS vs DOS | 70–73 | |

🎯 **可能考的题：** Differentiate traditional and distributed databases；Explain THREE benefits；Differentiate loosely and tightly coupled；Compare OpenMP, CUDA and MPI；Explain `#pragma omp parallel`；用 Cristian's algorithm 计算并 show steps。

**先备知识：** *process*（进程）= 正在运行的程序，有自己的 memory；*thread*（线程）= process 里面更轻的执行单位，和同一个 process 的其他 threads **共用 memory**。

---

## 开场

TAR UMT 选课日，早上 9:00，8,000 个学生同一秒按「Register」。整个系统只有**一台** server、**一个** database。CPU 冲到 100%，网页 timeout，9:03 server 直接 crash，所有人都选不到课。

这一章的每个概念，都是在回答「怎样避免这种事？」

---

## 1. Traditional vs Distributed Database（slides 3–6）

**问题：** 开场的例子里，所有 data 和处理都在同一个地方，这个地方既是瓶颈（bottleneck），也是一坏全坏的点。

**Traditional (centralised) database（slide 4）**
- 所有 data 存在**一个中央位置**（一台 main server）。
- 由**一个 DBMS** 管理。
- 所有用户的设备（PC、laptop、手机）都连到这**一台**中央 server 读写。

**Distributed database（slide 5）**
- 讲师原句：**Distributed storage places data/resources across multiple networked servers.**（把 data 和资源放在多台联网的 server 上）
- Client 透过网络（Internet）连到几种 server：data center/cloud server、web server、application server、database server。

**Distributed system 的特征（slide 3）：**
1. 多个 **independent nodes**（独立节点）
2. 透过 **network** 沟通
3. **Shared** resources and services
4. **No single point of control**（没有单一控制中心）
5. **Appears as a single system**（对用户来说看起来像一个系统）

**可以分布出去的东西（slide 6）：** operating system、file system、database/data、authentication、business logic、workload（负载分担，load sharing）。

| Aspect | Traditional database | Distributed database |
|---|---|---|
| Where data lives | One central server | Many networked servers |
| Control | One DBMS, one location | Spread across nodes |
| Failure | Server down → whole system down | Other servers can continue |
| Growth | Upgrade the one server | Add more servers |
| Distance to users | Everyone connects to one place | Data can sit close to users |

**💬 答题句 (EN):** *A traditional database stores and manages all data on a single central server using one DBMS, so it is simple but becomes a single point of failure and a bottleneck. A distributed database spreads data across multiple networked servers, which improves scalability, availability and performance, but makes consistency and management more complex.*

---

## 2. Benefits of Distributed Systems（slides 7–13）⭐

### 2.1 Scalability（可扩展性）— slides 7–9, 13
**意思：** 加资源就能应付更多用户或工作。讲师例子：选课时几千个学生同时 login → 加 server。

| | Vertical scaling（scale **up**，纵向） | Horizontal scaling（scale **out**，横向） |
|---|---|---|
| 意思 | 把**一台**机器升级得更强 | **加更多台**机器 |
| 例子 | 8-core → 16 → 32 → 64-core server | 1 → 2 → 5 → 20 → 100 台 server |

**讲师的价钱例子（slide 9），要会抄：**

| Cores needed | Vertical（一台更大的 server） | Horizontal（加 16-core server，每台 RM5,000） |
|---|---|---|
| 16 | RM5,000 | 1 server = RM5,000 |
| 32 | RM12,000 | 2 servers = RM10,000 |
| 64 | RM30,000 | 4 servers = RM20,000 |
| 128 | RM80,000 | 8 servers = RM40,000 |

**看点：** vertical 的价钱涨得**比 core 数还快**（越强的 server 越贵得离谱）；horizontal 是**线性**增加（每多 16 cores 就多 RM5,000）。

**Slide 8 的图（price per capacity unit vs capacity needed）：**
```
 每单位容量的价钱
   ^        vertical（虚线）：一开始便宜，
   |                          之后急升
   |  \                   /
   |   \  horizontal    /
   |    \______________X______   horizontal：起步较贵，
   |       vertical   / break-    之后效率高很多
   |       ________../  even
   |                    point
   +-------------------------------> 需要的额外容量
```
**看点：** 在 **break-even point**（盈亏平衡点）之前 vertical 比较便宜；过了这个点，horizontal 比较便宜。

**Web server 例子（slide 13）：**

| Capacity requirement | Vertical scaling | Horizontal scaling |
|---|---|---|
| Small | Upgrade RAM/CPU | Adding servers may be unnecessary |
| Medium | Bigger server | Several servers |
| Very large | Very powerful server becomes expensive | Add more standard servers |
| Extremely large | Hardware limits may be reached | Can continue adding machines |

### 2.2 Reliability（可靠性）— slide 10
一个部件坏了，系统**还能继续运作**。例子：银行把客户 data 复制在几台 server 上，一台坏了另一台顶上。

### 2.3 Performance（性能）— slide 11
工作**分给多台电脑一起做**。例子：video streaming 平台有几百万观众，很多台 server 同时服务不同用户。

### 2.4 Geographical distribution（地理分布）— slide 12
把 server 放在**靠近用户**的地方（user locality），缩短沟通距离、反应更快：
- 马来西亚客户 → 用马来西亚或附近的 server
- 欧洲客户 → 用欧洲 server
- 常被看的影片 → 放在靠近用户的地方
- 敏感 data → 按法律要求存在指定国家/地区

**✅ 满分答法 — "Explain THREE benefits of a distributed system, with examples." (6 marks)**
1. **Scalability** – the system can handle more users or work by adding more resources. Using horizontal scaling, more standard servers can be added; e.g. during course registration, extra servers are added to handle thousands of students logging in at the same time.
2. **Reliability** – the system continues to operate even if one component fails, because data and services are replicated. E.g. a bank keeps copies of customer data on several servers, so if one server fails another can serve the data.
3. **Performance** – work is divided among multiple computers that run simultaneously, so response time improves. E.g. a video-streaming platform uses many servers so that millions of users can watch at the same time.
*（第 4 个可选：geographical distribution — servers placed near users reduce latency, e.g. Malaysian users served from Malaysian servers.）*

> 中文提示：每个 benefit 写三样东西：**名字 + 意思 + 例子**，一个大概 2 分。

---

## 3. Challenges（slides 14–19）

可是……把系统分散出去，又会产生新问题：

| Challenge | 中文意思 | 讲师的例子 |
|---|---|---|
| (a) Avoiding a **single point of failure**（单点故障） | 某个部件太重要，它一坏整个系统就停 | Users → web servers → **一台** database server。DB 一 crash，web server 还在也没人能下单 |
| (b) **Replication**（复制） | 同一份 data/service 在不同机器上放多个 copy | Student DB 在 Server A，B 和 C 有 copy。A 坏了 B 顶上。**新问题：所有 copy 要保持一致（consistency）** |
| (c) **Availability and performance** | Available = 需要时用得到；performance = 负载大也反应快 | 订票系统：100 人没问题，100,000 人就 overload → 用 **load balancer**（负载均衡器）把请求分给 Server 1–4；Server 2 坏了其他继续 |
| (d) **Resource naming, addressing and location** | 资源（file、DB、printer、service、server、user account）分散在很多台机器，系统要能命名、定位、找到 | 系统知道 file 在 Server B → 找到 B 的 IP 192.168.1.20 → 有权限的学生透过网络读取 |
| (e) **Binding**（绑定） | 把一个名字/请求对应到真正处理它的资源 | 付款请求 bind 到 Payment Server 3；它坏了就 **rebind** 到 Payment Server 5，client 不用知道是哪台机器 |

Slide 17 的 load balancer 图：
```
            Users
              |
        Load Balancer
       /    |     |    \
 Server1 Server2 Server3 Server4
```
**看点：** load balancer 同时提升 performance（请求被分摊）和 availability（Server 2 坏了，其他继续服务）。

---

## 4. Loosely-coupled vs Tightly-coupled（slides 20–43）⭐⭐

**问题：** 想让几个 processor 一起做一份工作，有两种完全不同的接法。

### 4.1 Loosely coupled（松耦合）— slide 20
```
 [Clock] [Cache Memory (private)]  [Clock] [Cache Memory]  [Clock] [Cache Memory]
         [        CPU          ]           [    CPU     ]          [    CPU     ]
                Node 1                         Node 2                  Node 3
 ============================ Network ==================================
```
**看点：** 每个 node 有**自己的 clock、自己的 memory、自己的 CPU**，唯一的连接是 **network**。所以它们只能靠**送讯息、收讯息**沟通。
- 常用 API：**MPI**（Message Passing Interface）。

### 4.2 MPI 代码（slides 21–24）

**Hello World（slide 21）**
```c
#include <stdio.h>
#include <mpi.h>
int main(int argc, char* argv[]) {
    MPI_Init(&argc, &argv);                    // 启动 MPI
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);      // 我是第几号 process
    MPI_Comm_size(MPI_COMM_WORLD, &size);      // 总共几个 process
    printf("Hello World from MPI process %d out of %d\n", rank, size);
    MPI_Finalize();                            // 关闭 MPI
    return 0;
}
```
`mpiexec -n 4 msg_mpi.exe` → 同一个程序跑 4 份；输出顺序是**随机的**（例如 2, 0, 3, 1），因为每个 process 独立运行。`-n 5` 就印 5 行。

**Send / Receive（slide 22）**
```c
if (rank == 0) {
    int value = 50;
    MPI_Send(&value, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);
    printf("Process 0 sent %d\n", value);
} else if (rank == 1) {
    int value;
    MPI_Recv(&value, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    printf("Process 1 received %d\n", value);
}
```
"Process 1 received 50" 可能比 "Process 0 sent 50" 先出现：两个 printf 在不同 process，屏幕上的顺序不保证（slide 22 两种顺序都有）。

**MPI_Send 参数（slide 23）** — `MPI_Send(&data, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);`

| # | 值 | 意思 |
|---|---|---|
| 1 | `&data` | 要送的 data 所在变量的地址 |
| 2 | `1` | 送几个元素 |
| 3 | `MPI_INT` | data type 是 integer |
| 4 | `1` | 目的地 process（rank 1） |
| 5 | `0` | Message **tag**（标签），用来辨认和配对讯息 |
| 6 | `MPI_COMM_WORLD` | 包含所有 MPI process 的 communicator |

Array 版本：`int data[5] = {10,20,30,40,50}; MPI_Send(&data, 5, MPI_INT, 1, 0, MPI_COMM_WORLD);`（count 变 5）。

`MPI_STATUS_IGNORE`（slide 24）= 接收方不需要存关于这条讯息的资料。

### 4.3 Tightly coupled（紧耦合）— slide 25
```
                      [ Global Clock ]
   [  CPU   ]            [  CPU   ]            [  CPU   ]
   [ Cache (private) ]   [ Cache (private) ]   [ Cache (private) ]
   Processor 1           Processor 2           Processor 3
   |_____________________|_____________________|
                 [   Shared Main Memory   ]
```
**看点：** 每个 processor 有自己的 cache，但**共用一个 main memory 和一个 clock**，靠读写共享变量沟通。
- 常用 API：**OpenMP**（shared memory）。
- ⚠️ OpenMP 和 MPI 是 **programming API**；loosely/tightly coupled 讲的是 **hardware architecture**。答题时不要混在一起。

### 4.4 对比表（slide 43）— 一定要背

| Aspect | Loosely-coupled system | Tightly-coupled system |
|---|---|---|
| Memory | Each computer has its **own local memory** | Processors **share common memory** |
| Communication | Through a **network** using **message passing** | Through **shared memory** or high-speed internal interconnect |
| Operating system | Each machine may run its **own OS** | Usually controlled by **one OS** |
| Physical location | Machines may be **geographically separated** | Usually in the **same system** |
| Coupling | Weak connection between components | Strong connection |
| Scalability | **Easier** to add more computers | Harder to expand beyond hardware limits |
| Failure effect | One machine failing may **not** stop the whole system | Failure of shared components can affect the **whole** system |
| Communication speed | Generally **slower** (network) | Generally **faster** |
| Cost | Can use many **ordinary** computers | Often needs **specialised multiprocessor** hardware |
| Example | Cloud computing, distributed systems, clusters | Multicore computer, multiprocessor server |

- Loosely coupled ↔ 通常对应 **horizontal scaling**。
- Tightly coupled ↔ 通常对应 **vertical scaling**。

**Slide 50 的例子：** 需要 4 个 processor。
- Tightly coupled：**一台** server，4 个 CPU/core 共用同一份 RAM。
- Loosely coupled：**四台**独立电脑，各有 CPU 和 RAM，透过网络沟通。
- Distributed system **通常是 loosely coupled**，因为它由独立电脑透过网络组成。

**✅ 满分答法 — "Differentiate loosely-coupled and tightly-coupled systems." (6 marks, 3 points)**

| | Loosely coupled | Tightly coupled |
|---|---|---|
| Memory & communication | Each node has its own local memory; nodes communicate by message passing over a network (e.g. MPI) | Processors share a common main memory and communicate through shared variables (e.g. OpenMP) |
| Failure | Failure of one machine usually does not stop the whole system | Failure of a shared component (memory/bus) can stop the whole system |
| Scalability & cost | Easy to scale horizontally with ordinary computers; e.g. cloud, clusters | Limited by hardware, needs specialised multiprocessor hardware; e.g. multicore server |

> 中文提示：用表格答，每一行比较**同一个方面**，三行就是三个 point。

### 4.5 OpenMP：`#pragma omp parallel` 做什么（slides 26–42）⭐⭐

**设定（slide 26）：** Visual Studio → Project Properties → C/C++ → Language → **Open MP Support = Yes (/openmp)**。（gcc 用 `gcc -fopenmp`。）

**Slide 27：「这个程序输出什么？」**
```cpp
#include "stdafx.h"
#include <iostream>
#include <omp.h>
using namespace std;
int main() {
    #pragma omp parallel          // ← 分出一组 threads（fork）
    {
        cout << "Hello World\n";   // ← 每个 thread 都执行这一块
    }                              // ← implicit barrier，threads 汇合（join）
    return 0;
}
```
**答案：** 每个 thread 印一次 "Hello World"。默认 thread 数 = logical processor 数，所以在讲师 28 个 logical processor 的电脑上印 28 次（12 个的 laptop 印 12 次）。用 `cout` 时，几个 thread 同时写，行与行可能会混在一起。

**把这句指令拆开来看：**
- `#pragma` – compiler directive（编译器指令），是给 compiler 的指示，不是普通 C 代码。没开 OpenMP 的话会被忽略，程序照单线程跑。
- `omp` – 表示这是 OpenMP 的指令。
- `parallel` – 建立一个 **parallel region**：master thread **fork**（分叉）出一组 threads；**每个 thread 都执行整个 block**；到了 `}` 有 **implicit barrier**（隐式屏障，大家等齐），然后 threads **join** 回 master thread。

```
 master ──► #pragma omp parallel ──► T0 ──┐
                                   T1 ──┤
                                   T2 ──┼─► barrier（等全部）─► master 继续
                                   ... ─┘
           fork（分叉）                          join（汇合）
```
**看点：** 这就是 **fork–join model**（分叉-汇合模型）。

**💬 答题句 (EN):** *`#pragma omp parallel` tells the compiler to create a parallel region: the master thread forks a team of threads (by default one per logical processor), every thread executes the enclosed block once, and at the end of the block all threads wait at an implicit barrier and join back into the master thread.*

**Slide 28：拿 thread ID**
```cpp
#pragma omp parallel
{
    int tid = omp_get_thread_num();      // 0, 1, 2, ...（每个 thread 自己的）
    std::cout << "Hello World from thread " << tid << "\n";
}
```
输出 28 行，thread ID 顺序**随机**（23, 9, 1, 18, …），哪个 thread 先跑由 OS 决定。

**常用 OpenMP functions（slides 29–36）：**

| Function / clause | 作用 |
|---|---|
| `omp_get_num_procs()` | 可用的 **logical processor** 数（例如 28） |
| `omp_get_num_threads()` | **当前** team 的 thread 数（在 parallel region 外面是 1） |
| `omp_get_thread_num()` | 当前 thread 的 ID（0 … n−1） |
| `omp_set_num_threads(40)` | 之后的 parallel region 要用 40 个 threads |
| `#pragma omp parallel num_threads(50)` | 只有这个 region 用 50 个 threads |
| `#pragma omp single` | 只有**一个** thread 执行这一块（见 Ch3） |

**「可以跑超过 28 个 threads 吗？」（slides 29–33）** 可以。`omp_set_num_threads(40)` 会建立 40 个 **software threads**，但**同一瞬间**只有 28 个能跑（只有 28 个 logical processor）。Thread 0–27 可能马上拿到 core，28–39 要等有 core 空出来——一个 core 会**轮流**（time-slicing，时间片轮转）跑几个 threads。Task Manager 显示 12 个 logical processor 的电脑有 5,094 个 threads，也是同样道理：那是存在的 threads，不是同时在跑。

Slides 34–36：`num_threads(50)` + `single` 印 `omp_get_num_threads()` → "OpenMP threads created: 50"（只印一次）。把 `single` 注释掉（slide 36），**每个** thread 都印一次（12 个 processor 就印 12 次）。

### 4.6 Race condition 示范（slides 37–42）— 连到 Ch3/Ch4

```c
int x = 0;
#pragma omp parallel
{
    int tid = omp_get_thread_num();
    for (int i = 0; i < 1000; i++) {
        #pragma omp critical        // ← slide 40 有；slides 41–42 拿掉
        x = x + 1;
    }
    printf("Thread %d completed\n", tid);
}
printf("Final value of x = %d\n", x);
```

| Slide | 版本 | Threads | 应该是 | 实际印出 |
|---|---|---|---|---|
| 37 | 有 `critical`，每个 thread 加 1 次 | 12 | 12 | 12 ✔ |
| 38 | `num_threads(4)` + `critical` | 4 | 4 | 4 ✔ |
| 39 | `num_threads(4)`，**没有** critical，各加 1 次 | 4 | 4 | 4（运气好，次数少很少撞到） |
| 40 | 每个 thread 加 1000 次，**有** critical | 28 | 28,000 | **28000** ✔ |
| 41 | 每个 thread 加 1000 次，**没有** critical | 28 | 28,000 | **27574** ✘ |
| 42 | 同 41，再跑一次 | 28 | 28,000 | **26105** ✘（每次都不一样） |

**为什么没有 `critical` 就错？** `x = x + 1` 其实是三步：*读 x → 加 1 → 写回 x*。如果两个 thread 同时读到 x = 100，两个都写回 101，其中一次加法就**消失**了。这叫 **race condition**（竞争条件）：结果取决于 threads 执行的先后时机。`#pragma omp critical` 让一次只有一个 thread 执行这句（mutual exclusion，互斥），所以不会丢失更新，代价是变慢。

> ✔ 我用 gcc 验证过：8 threads 各加 100,000 次，没保护 → 587,152（错）；`critical` → 800,000；`atomic` → 800,000。

⚠️ **常见误解：** slide 39 印出正确答案，**不代表**代码是安全的。Race condition 不一定每次都出现。

---

## 5. OpenMP vs CUDA vs MPI（slides 44–49）⭐⭐

### CUDA（slide 44）
CUDA = NVIDIA 的 GPU 平行运算平台。**Host–device model**：CPU（host）和 main RAM 透过 PCIe 把 **kernel**（核函数）丢到 GPU（device）上跑。GPU 有很多 **SM**（streaming multiprocessor），每个 SM 有很多 CUDA cores、每个 thread 的 registers、以及**每个 SM/block 的 shared memory**；所有 SM 都能存取 **global memory**（device DRAM）。
- CUDA 也算 *tightly coupled*（在一张 GPU 里面），可以几千个 threads 同时跑，但它的 **memory 架构和 OpenMP 不同**：CUDA 的 "shared memory" 是 GPU 里一块小而快的 memory，只有**同一个 block** 的 threads 共用，**不是**系统 RAM。

### 对比表（slide 45）— 一定要背

| | OpenMP | CUDA | MPI |
|---|---|---|---|
| Full name | Open Multi-Processing | Compute Unified Device Architecture | Message Passing Interface |
| Computing model | Shared-memory parallel computing | GPU parallel computing | Message-passing parallel computing |
| Typical architecture | Tightly coupled | Tightly coupled (within a GPU) | Loosely coupled (across nodes) |
| Hardware | Multicore CPUs | NVIDIA GPUs | Multiple computers or processors |
| Execution units | Threads | GPU threads | Processes |
| Memory | Shared main memory | GPU global memory + per-block shared memory | Separate process address spaces |
| Communication | Shared variables | GPU memory & synchronisation | Sending and receiving messages |
| Synchronisation | Barriers, locks, critical sections | Thread/block synchronisation | MPI communication & collective operations |
| Scalability | Multiple CPU cores | Thousands of GPU threads | Multiple computing nodes |
| Languages | C, C++, Fortran | C, C++, Python (via libraries) | C, C++, Fortran, Python (via libraries) |
| Example | Parallel array processing on CPU | Parallel matrix multiplication on GPU | Distributed processing across a cluster |

**用途 + 代表性 function（slide 46）：**

| API | Purpose | Example |
|---|---|---|
| OpenMP | Create and coordinate parallel threads on CPUs | `#pragma omp parallel` |
| MPI | Processes communicate through message passing | `MPI_Send()` |
| CUDA | Execute parallel programs on NVIDIA GPUs | `cudaMalloc()` – allocates GPU memory that GPU threads can access |

**三种代码并排（slides 47–49）：**
```c
// OpenMP – 4 个 threads 共用 x
int x = 100;
#pragma omp parallel num_threads(4)
{ printf("Thread %d reads x = %d\n", omp_get_thread_num(), x); }
// → Thread 0/2/3/1 reads x = 100   （大家看到的是同一个 x）

// MPI – 4 个独立 process
MPI_Init(&argc, &argv);
MPI_Comm_rank(MPI_COMM_WORLD, &rank);
MPI_Comm_size(MPI_COMM_WORLD, &size);
printf("Hello from process %d of %d\n", rank, size);
MPI_Finalize();

// CUDA – 4 个 GPU threads 执行同一个 kernel
__global__ void hello() { printf("Hello from GPU thread %d\n", threadIdx.x); }
int main() {
    hello<<<1, 4>>>();          // 1 个 block，4 个 threads
    cudaDeviceSynchronize();    // 等 GPU threads 做完
    return 0;
}
```

**✅ 满分答法 — "Compare OpenMP, CUDA and MPI."**：画表格，至少用这几行：*computing model、architecture、hardware、execution unit、memory、communication*，每格写一句，最后每个 column 给一个例子。

---

## 6. Architecture models & transparency（slides 51–53）

**Workstation–Server model（slide 51）：** 每个用户有自己的 workstation 做处理；强大的 server 提供共享服务（file、database、web、printing、storage），**每台 server 可以提供不同的服务**。流程：workstation 发请求 → network → server 提供服务 → 回传结果。

**Processor-Pool model（slide 52）：** 所有 processor 放在一个**共享的 pool**，**需要时才分配**给用户或 job，用户不拥有固定的 processor。一个大 job 可以暂时用很多 processor（Processor 1 做第 1 部分 … Processor 4 做第 4 部分），最后合并结果。**Grid computing** 就是基于这个模型，只是规模更大、可以跨多个机构。

**Transparency（透明性）（slide 53）** = 把分布式环境的复杂性**藏起来**，不让用户和应用程序察觉。8 种：

| Type | 藏起来的东西 | Example |
|---|---|---|
| Access | 存取方式的差别 | Local and remote file accessed the same way |
| Location | 资源在哪里 | Visit a website without knowing the server location |
| Migration | 资源会搬家 | File moved to another server, access unchanged |
| Relocation | 资源在**使用中**搬家 | VM moves to another server while service continues |
| Replication | 有很多个 copy | Several DB replicas appear as one DB |
| Concurrency | 很多人同时用 | Many users update a shared database |
| Failure | 部件故障（被隐藏/恢复） | Another server takes over when one fails |
| Persistence | data 在 memory 还是 disk | Object loaded without knowing if from RAM or disk |

> 中文提示：Migration 和 Relocation 最容易搞混——**Relocation 是「用着用着就搬了」**。

---

## 7. Distributed Real-Time Systems（slides 54–59）

**Distributed real-time system（slide 54）：** 多台联网的电脑/node 合作，在**严格的时间期限内**监测、处理、回应事件。流程：sensors/input devices → local nodes/controllers →（带时间戳的 data）network → processing nodes/servers → control commands → actuators/output devices。

**Real-time system（slide 55）：** **controlling system**（电脑）+ **controlled system**（环境），例如汽车电子系统、航空交通管制。

**典型特征（slide 56）：** time critical；由并发的 processes（tasks）组成；tasks 共享资源（例如 processor）并互相沟通；reliability 和 fault tolerance（容错）非常重要；执行特定工作。

**Time-critical 例子（slide 57）：** airbag 几毫秒内弹出；ABS 立刻调整刹车压力；飞机控制指令有严格期限；心跳监测器快速侦测危险；工业机器人准时动作；红绿灯；影音串流；多人线上游戏。

**分布式版本（slide 58）：** 一组 node（A–F）透过 **real-time communication network** 连接。

**两个特别问题（slide 59）：** (1) **clock synchronisation**（时钟同步），(2) **real-time communication**。
为什么时钟重要：每个 node 有自己的 clock，而 clock 会 **drift**（漂移，慢慢走得不一样）。如果 node A 说事件发生在 10:00:05，node B 说 10:00:03，除非时钟一致，否则分不出真正的先后顺序。

---

## 8. Clock Synchronisation ⭐⭐

### 8.1 Cristian's algorithm（slides 60–65）

**想法：** 有一台 **time server** 跟 **UTC**（世界协调时间）同步。Client 问它时间，再补回讯息路上花的时间。

```
 Client（发送方）                           Time server
   T0 ──────────── request ───────────────►
                                             C = server 回复时的时间
   T1 ◄──── reply（带着 C）────────────────
   （T0、T1 都是用 CLIENT 自己的时钟量的）
```

**推导（slide 61）：**
1. 第一个想法：`Trec = C`——但这忽略了回复在路上花的时间。
2. 改进：`Trec = C + Ttrans`，Ttrans = 回复从 server 传到 client 的时间。
3. 怎么估 Ttrans：**round-trip time**（往返时间）= `T1 − T0`（去 + 回）。假设**去和回的延迟一样**，单程 ≈ 一半：

> **Trec = C + (T1 − T0) / 2**

**为什么除以 2？** 讯息走了两个方向（client → server，server → client）。如果两边延迟一样，往返时间的一半就是回复的单程延迟。

**例题（slide 62）：** T0 = 10:00:10，T1 = 10:00:14（client 时钟），C = 10:05:20（server）

| 步骤 | 算式 |
|---|---|
| 公式 | Trec = C + (T1 − T0)/2 |
| 往返时间 | T1 − T0 = 10:00:14 − 10:00:10 = 4 s |
| 单程估计 | 4 / 2 = 2 s |
| 结果 | Trec = 10:05:20 + 2 s = **10:05:22** |

Client 收到回复的那一刻，把时钟设成 10:05:22。

**Slides 63–64 练习（讲师说要 show steps）：**
Client 在 **20:00:41** 发出请求，在 **20:00:49** 收到回复（client 时钟）。Server 发回复时记录的时间是 **20:05:57**。求 client 收到回复时估计的 server 时间。

**✅ 满分答法**
- T0 = 20:00:41, T1 = 20:00:49, C = 20:05:57
- Trec = C + (T1 − T0) / 2
- T1 − T0 = 20:00:49 − 20:00:41 = 8 s
- (T1 − T0)/2 = 8 / 2 = 4 s
- Trec = 20:05:57 + 4 s = **20:06:01**
- The client sets its clock to 20:06:01.

> ⚠️ **Slide 64 的错误：** slide 把 C 写成「**10**:05:57」、答案写成「**10**:06:01」，还留着上一题的「approximately 10:05:22」（copy-paste 没改）。题目说的是 **20**:05:57，所以正确答案是 **20:06:01**。考试时小时要写对。

**Cristian's algorithm 的问题（slide 65）：**
1. Network delay **会变**（每条讯息的延迟都不一样）。
2. 两个方向的延迟**可以不一样**，即使是同一条线路——所以「除以 2」只是估计。
3. 每台电脑的**处理时间**（例如 server 处理请求）也会变。
（额外，slide 没写：它依赖单一 time server，那台 server 本身就是 single point of failure。）

**💬 答题句 (EN):** *Cristian's algorithm assumes the request and reply take equal time, so the one-way delay is estimated as half the round-trip time; if network delays vary or are asymmetric, or the server's processing time varies, the estimate becomes inaccurate.*

### 8.2 Berkeley algorithm（slides 66–69）

**想法：** 没有 UTC server。一台 **master** 收集大家的时间，把整组设成**平均值**。

步骤：
1. Master 向所有 slave node 要时间。
2. Slave 回报自己 system clock 的时间。
3. Master 算出平均差，把同步后的时间（或每个 node 要调多少）**broadcast** 给整个网络。

**例题（slide 68）：**

| Machine | Clock | 跟 master 的差 |
|---|---|---|
| Master | 3:00 | 0 min |
| Slave 1 | 3:10 | +10 min |
| Slave 2 | 2:50 | −10 min |
| Slave 3 | 3:20 | +20 min |

平均差 = [0 + 10 + (−10) + 20] / 4 = 20 / 4 = **5 min** → 同步时间 = 3:00 + 5 = **3:05**

| Machine | 调整 | 调整后 |
|---|---|---|
| Master | 3:00 + 5 min | 3:05 |
| Slave 1 | 3:10 − 5 min | 3:05 |
| Slave 2 | 2:50 + 15 min | 3:05 |
| Slave 3 | 3:20 − 15 min | 3:05 |

> 中文提示：master 自己的差是 0，**也要算进去**，所以除以 4 不是 3。

**Pseudocode（slide 69）：**
```text
# Receive clock values from slaves
repeat_for_all_slaves:
    time_at_slave_node = receive_time_at_slave()
    # Calculate offset relative to master
    time_difference = time_at_slave_node - time_at_master_node
# Include master's offset = 0
average_time_difference = sum(all_time_differences) / total_number_of_nodes
# Calculate synchronized time
synchronized_time = current_master_time + average_time_difference
# Broadcast synchronized time to whole network
broadcast_time_to_all_slaves(synchronized_time)
```

| | Cristian's | Berkeley |
|---|---|---|
| Reference | External UTC time server | No UTC; group average |
| Who asks | Client asks server | Master polls slaves |
| Result | Client matches server | All nodes agree with each other |

---

## 9. Distributed Operating Systems（slides 70–73）

**Distributed OS（slide 70）：** 在多台独立电脑（node）上运行，管理它们的资源，对用户呈现**一个统一的系统**。每个 node 在自己的 CPU/memory/storage 上跑 DOS；network 把它们连起来；DOS 的资源管理协调所有 node 的资源。

**OS 做什么（slide 71）：** process/thread 管理（scheduling、communication、synchronisation）、memory 管理、storage 管理、file system、protection & security、networking。

**NOS vs DOS（slide 73）：**

| Aspect | Network OS (NOS) | Distributed OS (DOS) |
|---|---|---|
| System image | **Multiple** system images — each computer has its own OS and appears separate | **Single** system image — all computers appear as one system |
| Autonomy（自主性） | **High** — each node controls its own resources and decisions | **Low** — nodes cooperate under centralised/coordinated control |
| Fault tolerance | **Limited** — one node failing may affect shared services; manual recovery often needed | **High** — detects and recovers from node failures automatically |
| Example | 30 PCs all running Windows 11 — still separate systems | Google Search — many servers cooperate as one service |

---

## 回到开场
选课日的问题怎么解决：把一台 server 换成**几台 server + load balancer**（horizontal scaling）；**replicate** student database，一台 crash 不会让选课停掉（reliability）；保持 replicas **一致**；所有 server 的时钟要对齐（Cristian/Berkeley），「先到先得」才公平。每台 server 里面再用 **OpenMP threads** 把每个 core 用满——然后马上就遇到 slides 41–42 的 race condition，Ch3 和 Ch4 会解决它。

---

## ⚠️ Slides 有问题的地方

| Slide | Slide 写的 | 正确理解 |
|---|---|---|
| 64 | C = 10:05:57，答案 10:06:01，「approximately 10:05:22」 | 题目是 20:05:57 → 答案 **20:06:01**（从 slide 62 copy-paste 没改） |
| 39 | 没有 `critical` 但 x 正确（4） | 只是时机刚好，仍然是 race condition |
| 12 | "Sensitive Aata" | 打错字，应为 Sensitive Data |
| 2（目录） | Section 2 是 "What is a Real-Time System?" | 内容讲的是 *distributed* real-time system（slide 54） |

考试提示：如果题目抄 slide 的数字，用**正确**的算法回答，而且要 show steps。

---

## 术语表

| English | 中文 | 一句话说明 |
|---|---|---|
| Distributed system | 分布式系统 | 很多台独立电脑，看起来像一个系统 |
| Scalability | 可扩展性 | 加资源就能应付更多工作 |
| Vertical / horizontal scaling | 纵向 / 横向扩展 | 升级一台 / 加更多台 |
| Single point of failure | 单点故障 | 一坏就全部停的部件 |
| Replication | 复制 | 在不同机器放多个 copy |
| Consistency | 一致性 | 所有 copy 的值相同 |
| Load balancer | 负载均衡器 | 把请求分给多台 server |
| Binding | 绑定 | 把名字/请求对应到真正的资源 |
| Loosely / tightly coupled | 松耦合 / 紧耦合 | 各自 memory + 网络 / 共用 memory |
| Message passing | 消息传递 | 用 send/receive 交换 data |
| Fork–join | 分叉-汇合 | master 分出 threads，做完再汇合 |
| Implicit barrier | 隐式屏障 | 自动的「大家等齐」点 |
| Race condition | 竞争条件 | 结果取决于 thread 执行时机 |
| Transparency | 透明性 | 把分布的复杂性藏起来 |
| Clock drift | 时钟漂移 | 时钟慢慢走得不一样 |
| Round-trip time | 往返时间 | T1 − T0 |
| Autonomy | 自主性 | node 能多独立地做决定 |
| Fault tolerance | 容错 | 有故障也能继续运作 |

---

## 速记卡（Cheat sheet）
- Distributed DB = data 分布在多台联网 server；traditional = 一台中央 server + 一个 DBMS。
- Benefits：**Scalability、Reliability、Performance、Geographical distribution**。
- Challenges：**SPOF、Replication（consistency）、Availability & performance、Naming/addressing/location、Binding**。
- Loosely = 各自 memory、network、MPI、horizontal。Tightly = 共用 memory、OpenMP、vertical。
- OpenMP：threads / shared memory / CPU。CUDA：GPU threads / GPU memory。MPI：processes / messages / cluster。
- `#pragma omp parallel` = fork 一组 threads，每个 thread 跑整个 block，implicit barrier，join。
- Cristian：**Trec = C + (T1 − T0)/2**。问题：延迟会变、两边不对称、处理时间会变。
- Berkeley：差的平均（master 差 = 0 也要算），broadcast 调整量。
- NOS = 多个 image、自主性高、容错低。DOS = 单一 image、自主性低、容错高。

---

## 练习题（附答案）
*题目用英文（跟考卷一样），答案下面附华文说明。*

### A. MCQ
1. Which API is normally used for a loosely-coupled cluster? (a) OpenMP (b) CUDA (c) MPI (d) pthreads
2. In Cristian's algorithm, T1 − T0 represents: (a) server processing time (b) one-way delay (c) round-trip time (d) clock drift
3. A VM moves to another server while users keep using it. Which transparency? (a) Migration (b) Relocation (c) Location (d) Access
4. Which is TRUE of a tightly-coupled system? (a) Each node runs its own OS (b) Processors share common memory (c) Communication is via network messages (d) Easy to add machines indefinitely
5. Berkeley: master 4:00, slaves 4:06, 3:58, 4:04. Synchronised time? (a) 4:02 (b) 4:03 (c) 4:04 (d) 4:00

**答案：** 1-c；2-c；3-b（**使用中**搬走）；4-b；5-a — 差是 0, +6, −2, +4，总和 8；8 ÷ 4 个 node = 2 min → 4:00 + 2 = **4:02**。

### B. Short answer
1. Explain the difference between vertical and horizontal scaling and why horizontal scaling is usually preferred for very large systems.
   **Answer:** Vertical scaling upgrades one machine (more cores/RAM); horizontal scaling adds more machines. Very powerful single servers become disproportionately expensive (RM30,000 for 64 cores vs RM20,000 for four 16-core servers) and eventually hit hardware limits, while horizontal scaling can keep adding standard machines and also removes the single point of failure.
   *华文：vertical 越升级越贵，而且有硬件上限；horizontal 可以一直加普通机器，还顺便解决单点故障。*
2. State TWO challenges introduced by replication.
   **Answer:** Keeping all copies consistent when data is updated; extra storage/network cost to maintain copies (and deciding which copy to read/write).
3. Explain why `#pragma omp critical` makes the program in slide 40 slower but correct.
   **Answer:** It allows only one thread at a time to execute `x = x + 1`, so the read–add–write steps of different threads cannot interleave and no update is lost (correct 28,000). But threads must wait in turn to enter, so they execute that part serially, reducing speed.
   *华文：一次只放一个 thread 进去，所以不会丢失更新；但大家要排队，那一段等于变回单线程，所以慢。*

### C. Application
1. A client sends a request at 14:20:15 and receives a reply at 14:20:21 (client clock). The server's time in the reply is 14:22:40. Find Trec.
   **Answer:** T1 − T0 = 6 s; 6/2 = 3 s; Trec = 14:22:40 + 3 s = **14:22:43**.
2. Berkeley: master 9:00, slaves 9:12, 8:54, 9:06, 9:08. Find synchronised time and each adjustment.
   **Answer:** differences 0, +12, −6, +6, +8 → sum 20; 20/5 = 4 min → **9:04**. Master +4, Slave1 −8, Slave2 +10, Slave3 −2, Slave4 −4.
   *华文：5 个 node（含 master）所以除以 5。*
3. What is printed, and how many times, by `#pragma omp parallel num_threads(6) { printf("Hi %d\n", omp_get_thread_num()); }`?
   **Answer:** "Hi 0" … "Hi 5", six lines (one per thread), in an unpredictable order.

### D. Thinking
1. Why is Cristian's estimate wrong when the request takes 1 s and the reply takes 5 s?
   **Answer:** Round trip = 6 s, the algorithm assumes 3 s for the reply but it really took 5 s, so the client sets its clock 2 s **behind** the true server time. The algorithm assumes symmetric delay.
   *华文：算法假设去回一样长，实际不一样，所以估错了 2 秒。*
2. A system uses 30 Windows PCs sharing files on a LAN. Is this a NOS or DOS? Justify with two aspects.
   **Answer:** NOS — each PC has its own OS image and appears as a separate computer (multiple system images), and each PC controls its own resources (high autonomy); a failed PC is not recovered automatically.

---

## Slide 索引
| 笔记章节 | Slides |
|---|---|
| 1 Databases | 3–6 |
| 2 Benefits | 7–13 |
| 3 Challenges | 14–19 |
| 4 Loosely/tightly、MPI、OpenMP | 20–43 |
| 5 OpenMP vs CUDA vs MPI | 44–50 |
| 6 Architecture、transparency | 51–53 |
| 7 Real-time | 54–59 |
| 8 Clock sync | 60–69 |
| 9 DOS | 70–73 |

## 跟其他章的连接
- Ch2：message passing vs shared memory、MPI 作为沟通模型、sockets。
- Ch3：更多 OpenMP 指令（`single`、`for`、`sections`、`critical`、`atomic`）和深入的 MPI。
- Ch4：race condition，以及 lock、semaphore 怎么解决它。
