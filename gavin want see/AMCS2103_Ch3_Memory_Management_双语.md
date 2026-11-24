# AMCS2103 — Chapter 3: Memory Management（Shared Memory、OpenMP、DSM、MPI）（双语版）

> **怎么读：** 讲解用华文，terms 和代码保留英文；**💬 答题句 (EN)** = 考试直接写的英文句子。
>
> 这章大部分是 OpenMP 代码截图加输出，几乎没有解释。这份笔记把每个 directive 都抄下来，讲清楚*谁执行什么*、*谁等谁*，并把讲师的两个对比表（slide 14 和 slide 31）整理成一个框架。也指出了 MPI send/recv slide（slide 76）少了一个参数的 bug。OpenMP 的结果都用 gcc 验证过；这里没有安装 MPI，所以 MPI 代码**没有实际跑过**。

---

## 0. 本章总览

**一句话：** 很多 processor 共用 memory 时，要协调*谁做哪部分工作*、*谁什么时候碰共享 data*——OpenMP 两样都有指令；memory **不共享**时（cluster），就改用 DSM 或 MPI 讯息。

| # | 主题 | Slides | 考试重点（讲师 hint） |
|---|---|---|---|
| 1 | 为什么要更多电脑；centralised memory 复习 | 3–6 | |
| 2 | Simple vs shared memory model、bus contention、performance | 7–10 | |
| 3 | Shared-memory computing、threads | 11–12 | |
| 4 | OpenMP：`parallel` vs `parallel for` | 13–17 | ⭐ |
| 5 | `sections` / `section` | 18–20 | ⭐（"pg 22 section"） |
| 6 | `single` | 21–22 | ⭐ |
| 7 | `parallel for` + `schedule` | 23–27 | |
| 8 | **`single` vs `critical` vs `atomic`** + 对比表 | 28–32 | ⭐⭐（pg 29、pg 31） |
| 9 | `parallel for` vs `omp for`、nowait | 33–34 | |
| 10 | Loop 例子、reduction、false sharing/padding | 35–48 | |
| 11 | Barriers | 49–58 | （讲师说 "no barrier"，优先度低） |
| 12 | Distributed Shared Memory（DSM） | 59–62 | |
| 13 | MPI：functions、send/recv、sync vs async | 63–91 | |

---

## 开场
你的 laptop 有很多 core，要把两间公司 25,000 个员工的薪水加起来。一个 core 做只要 5 ms。你想：「我有那么多 core，分开做！」加了一行 `#pragma omp parallel`，结果总数**错了**，而且每次跑都不一样。这章就是要讲为什么——以及每种情况要用哪个 directive 来修。

---

## 1. 为什么要更多电脑？（slides 3–6）
工作越来越大 → 要更多电脑：**更大**（更多 memory 和 storage）、**更快**（每个 processor 更快）、**更多**（同时做很多计算）。

**Centralised memory management 复习（slide 6）：** virtual memory（虚拟内存）让 memory 超过实体 RAM；paging vs segmentation；internal vs external fragmentation（碎片）；segment 放置和 page 替换算法；page fault 和 thrashing（抖动）。*（OS 课的背景知识，不是这章重点。）*

---

## 2. Memory models（slides 7–10）

**Simple memory model（slide 7）：**
- 所有 processor 的存取时间**都一样**。
- 要**严格控制 multiprogramming 的程度**（同时跑几个程序）。
- 因为 overhead，通常**没有 virtual memory 或 caching**。

**Shared memory model（slide 8）：**
- 用来做 **inter-process communication**；多个 process 共用 memory 位置。
- 可以包括实体 RAM、local cache 和 secondary storage。
- 透过**共同的 bus** 存取 memory → 可能出现 **bus contention**。（图：Intel Core i7-970——6 个 core 共用 L3 cache 和 memory controller。API：OpenMP。）

**Bus contention（总线争用）（slide 9）：** 太多人**同时要用 bus**，bus 变成**瓶颈**。大型 multiprocessor（**> 32 个 CPU**）不能只用一条 bus 连 CPU 和 memory，因为争用会严重到无法处理。

**Shared memory 的性能问题（slide 10）：**
- **Scalability** – 规模变大但性能不下降。
- **Real-time needs**（实时需求）。
- **Overlap of communication and computation**（沟通和计算重叠进行）；**prefetching**（预取）data。
- **存取非本地 memory 很贵**——本地 ≈ 10 ns vs 非本地 ≈ 100 ns（最多 **10:1**）。

---

## 3. Shared-memory computing 和 threads（slides 11–12）
- 大型 **symmetric multiprocessor（SMP）** 系统给大问题更多计算资源。
- **Threading** 是最流行的 shared-memory 技术。
  - **优点：** 开发者容易把工作、任务和 data 分开。
  - **缺点：** **data races**（数据竞争）。
- **Thread 是 lightweight process**（slide 12）：所有 threads 共用 process 的 **text（code）、data、heap、files、locks、sockets、process ID**；每个 thread 有**自己的 stack、stack pointer、program counter、registers**。因为共用 process 的状态，**thread 之间切换（context switch）比 process 之间便宜很多**。

```
 Process（user address space）
 +--------------------+  所有 threads 共用：
 | Thread 0 stack     |   - code (text)、global data、heap
 | Thread 1 stack     |   - process ID、files、locks、sockets
 | text | data | heap |  每个 thread 自己的：
 +--------------------+   - stack、stack pointer、PC、registers
```

---

## 4. OpenMP：`parallel` vs `parallel for`（slides 13–17）⭐

### 核心差别（slide 14）— 要背

| | `#pragma omp parallel` | `#pragma omp parallel for` |
|---|---|---|
| Purpose | **Create threads** | **Parallelise a loop** |
| Threads execute | The **entire block** (every thread) | **Different loop iterations** |
| Work sharing | **Manual** (you split work using thread ID) | **Automatic** sharing of iterations |
| Barrier | Both have an **implicit barrier** at the end of the parallel region |

### 例子 A — `parallel` 里面放一个 loop（slide 15）
```c
#pragma omp parallel num_threads(4)
for (int i = 0; i < 5; i++) {
    int tid = omp_get_thread_num();
    printf("Thread %d: Iteration %d\n", tid, i);
}
```
- 4 个 threads；**每个 thread 都独立跑完整个 loop**，各跑 5 次。
- 总共 **4 × 5 = 20** 行（每个 `i` 出现 4 次）。

### 例子 B — `parallel for`（slide 16）
```c
#pragma omp parallel for num_threads(4) schedule(static)
for (int i = 0; i < 5; i++) {
    int tid = omp_get_thread_num();
    printf("Thread %d: Iteration %d\n", tid, i);
}
```
- 4 个 threads；OpenMP 把 5 次 iteration **分给**大家；**每次 iteration 只跑一次**。
- 总共 **5** 行。Slide 输出：T0：0,1；T1：2；T2：3；T3：4。
- `schedule(static)` = loop 开始**之前**就分好。

**Implicit barrier（slide 17）：**
```c
#pragma omp parallel num_threads(4)
{ printf("Hello from Thread %d\n", omp_get_thread_num()); }   // 这里有 implicit barrier
printf("All threads completed!\n");
```
"All threads completed!" 一定**最后**印——4 个 threads 都做完，main thread 才继续。`parallel for` 也一样。

**💬 答题句 (EN):** *`#pragma omp parallel` creates a team of threads and every thread executes the whole block, so work must be divided manually; `#pragma omp parallel for` creates the team and automatically divides the loop iterations so that each iteration is executed by exactly one thread. Both end with an implicit barrier.*

---

## 5. `sections`（slides 18–20）⭐

**问题：** `parallel for` 只能分**一个 loop**。如果有几件**不一样**的工作——例如读文件、算统计、画图——彼此独立，怎么办？

**答案：`parallel sections`——让 threads 分摊「不同的代码块」**
```c
#pragma omp parallel sections        // 建立 threads + 分 sections
{
    #pragma omp section
    { printf("Section 1 executed by thread %d\n", omp_get_thread_num()); }
    #pragma omp section
    { printf("Section 2 executed by thread %d\n", omp_get_thread_num()); }
    #pragma omp section
    { printf("Section 3 executed by thread %d\n", omp_get_thread_num()); }
}                                     // implicit barrier
return 0;
```
- 每个 `section` 是**不同的代码块**；每个 section **只执行一次**，由**一个** thread 执行。
- 不同 section 可以由不同 thread **同时**执行。

**Slide 19 – taskA / taskB：**
```c
#pragma omp parallel sections
{
    #pragma omp section
    { taskA(); }
    #pragma omp section
    { taskB(); }
}                         // implicit barrier
printf("After sections\n");
```
**所有** section 都做完（例如 Thread 0 → taskA，Thread 1 → taskB），才印 "After sections"。

**Threads 数 vs sections 数（slide 20）：**
- 4 个 threads、3 个 sections → 3 个 thread 各做一个，**1 个 thread 闲着**。
- 2 个 threads、3 个 sections → **有 section 要等**到有 thread 空出来（一个 thread 连续做两个）。
- Slide 20 的例子（`num_threads(4)`、3 个 sections）印 "Section 1, Section 3, Section 2, After barrier: Thread 0"——**section 顺序不固定**；region 结束后**只有 master thread（Thread 0）**继续，所以 barrier 后面那句一定是 Thread 0 印的。

> ✔ gcc 验证：哪个 thread 做哪个 section 每次不同（有一次 Thread 0 做了全部三个——这是合法的）。"After sections" 永远最后。

**💬 答题句 (EN):** *`#pragma omp sections` divides different, independent blocks of code among the threads: each section is executed exactly once by one thread, different sections can run in parallel, and there is an implicit barrier at the end so the code after the sections runs only when all sections have finished.*

---

## 6. `single`（slides 21–22）⭐
**`#pragma omp single`：只让 1 个 thread 执行这块代码一次。**
```c
#pragma omp parallel num_threads(4)
{
    #pragma omp single
    { printf("Hello\n"); }          // 一个 thread 印，其他在 single 结尾等
    printf("After single\n");       // 4 个 threads 都印
}
```
输出：`Hello` 一次，然后 `After single` × 4。
为什么 "Hello" 一定最先：`single` 结尾有 **implicit barrier**——其他 threads 在那里等 single 做完。

Slide 22：做 `single` 的**不一定是 Thread 0**——0、1、2、3 都有可能（slide 输出："Hello from Thread 0"，然后 After single：3, 1, 0, 2）。

什么时候用 `single`：只要一个 thread **读文件**或**印总结**（slide 28）。

> ✔ gcc 验证：`Hello` × 1，`After single` × 4。

---

## 7. `parallel for` 和 `schedule`（slides 23–27）

```c
#pragma omp parallel for
for (int i = 0; i < 4; i++) printf("i = %d\n", i);
printf("After loop\n");
```
可能输出：i = 1, 3, 0, 2, After loop。Iterations 分给 threads 同时跑 → **顺序会变**；结尾有 implicit barrier → "After loop" 最后。哪个 thread 拿哪个 iteration 取决于 compiler/runtime 的默认设定。

**Schedule 种类（slide 25）：**

| Clause | 意思 |
|---|---|
| `schedule(static)` | 事先分好；每个 thread 拿差不多大小的**连续一段** |
| `schedule(static, 1)` | 固定的 **round-robin**（轮流）分配，每次 1 个 |
| `schedule(dynamic)` | Thread **做完了再去拿新的** iteration |
| `schedule(dynamic, 1)` | Dynamic，每次拿刚好 1 个 |

**10 个 iterations、4 个 threads（slides 26–27）：**

| Thread | `static` | `static, 1` |
|---|---|---|
| T0 | 0, 1, 2（3 个） | 0, 4, 8（3 个） |
| T1 | 3, 4, 5（3 个） | 1, 5, 9（3 个） |
| T2 | 6, 7（2 个） | 2, 6（2 个） |
| T3 | 8, 9（2 个） | 3, 7（2 个） |
| Total | 10 | 10 |

> ✔ gcc 跑出来的分配跟上表完全一样。

什么时候用 dynamic：每个 iteration 的工作量**不一样**——先做完的 thread 去拿更多工作，不会闲着（较好的 **load balancing**，负载均衡，slide 40）。

---

## 8. `single` vs `critical` vs `atomic` ⭐⭐（slides 28–32）

**问题（来自 Ch1 slide 41）：** 很多 threads 对共享变量做 `x = x + 1` → 更新被吃掉（race condition）。需要 **mutual exclusion**（互斥）：同一时间只能一个 thread 在那段代码里。

### 什么时候用哪个（slide 28）
- **single** – 只要一个 thread 做这件事（读文件、印总结）。
- **critical** – 很多 threads 要写**共享 data**；用一把 **lock**，其他人都**等（被 block）**；用在加总、计数、更新共享变量；提供 mutual exclusion，防止 race condition。
- **atomic** – CPU 只等非常短的时间（不睡觉、不被 block）；专门给**一个变量**用；用**硬件原子指令**（hardware atomic instructions）；**overhead 低很多、更快**。

### `#pragma omp critical`（slide 29）
一次只有**一个** thread 能执行这块；其他人**等**。可以保护**多行**代码。
```c
#pragma omp critical        // ✔ 保护一个有多句的 block
{
    sum += a;
    count++;
    printf("Thread updating\n");
}

#pragma omp critical        // ✘ 没有大括号：只保护 `sum += a;`，
sum += a;                   //    count++ 和 printf 没被保护
count++;
printf("Thread updating\n");

#pragma omp critical {      // ✔（讲师打勾）保护 count++
    count++;
}

#pragma omp critical        // ✔ 只有一句，不用大括号
count++;
```
**关键规则：** 没有 `{ }` 的话，directive 只管**下一句**。

### `#pragma omp atomic`（slide 30）
只保护**一个简单的更新**，对象是一个 memory 位置 → 快很多。**不能保护多行**，也**不能接 `{ }` block**。
```c
#pragma omp atomic          // ✔
sum += x;

#pragma omp atomic          // ✘ atomic 不能接 block
{ sum += x; }

#pragma omp atomic          // ✘ 多句 + printf
{ sum += a; count++; printf("Thread updating\n"); }
```
**Atomic 保护的是一个特定的 memory 操作；critical 保护的是一整块代码。**

### 对比表（slide 31）— 要背

| | `sections` | `single` | `omp for` / `parallel for` | `critical` / `atomic` |
|---|---|---|---|---|
| Who runs | Different threads, one per section | Only one thread | Iterations split among threads | All threads, one-by-one |
| How many times | Each section runs once | Block runs once only | Only after everyone finishes does execution continue | One-at-a-time execution |
| Threads/blocks | Multiple threads, different blocks | Exactly one | Threads execute their assigned iterations | Other threads wait only to **enter** the critical region |
| Barrier | **Implicit barrier** | **Implicit barrier** | **Implicit barrier** | **No implicit barrier** |
| Mutual exclusion | No | No | No | **Yes** |

- **Barrier**（屏障）= 等所有人到齐 → 再一起继续。
- **Mutual exclusion**（互斥）= 保护共享代码/data，避免 race condition。

> 中文记法：**前三个（sections、single、for）是「分工」→ 有 barrier、没互斥；critical/atomic 是「排队」→ 有互斥、没 barrier。**

### Slide 32 — 证明 critical 没有 barrier
```c
#pragma omp parallel num_threads(4)
{
    #pragma omp critical
    { printf("Critical: Thread %d\n", omp_get_thread_num()); }   // 这里「没有」 implicit barrier
    printf("After critical: Thread %d\n", omp_get_thread_num());
}   // 这里才有 implicit barrier（parallel 结尾）
```
输出：Critical 0、**After critical 0**、Critical 1、Critical 2、After critical 1……——Thread 0 在 Threads 1–3 做完 critical **之前**就印了 "After critical"。Thread 出了 critical 就直接往下走。

**✅ 满分答法 — "Explain the difference between `#pragma omp critical` and `#pragma omp atomic`, with code." (6 marks)**

| | critical | atomic |
|---|---|---|
| Protects | A whole **block** of code (multiple statements) | A **single** simple update of one memory location (e.g. `sum += x;`) |
| Mechanism | A lock; waiting threads are blocked | Hardware atomic instruction; very short wait |
| Overhead | Higher (slower) | Much lower (faster) |
| Code | `#pragma omp critical { sum += a; count++; }` | `#pragma omp atomic` then `sum += x;` |

Both give **mutual exclusion** and **no implicit barrier**. Use atomic for one simple counter/sum update; use critical when several statements must run together.

> ✔ gcc 测试，8 threads × 100,000 次加法：没保护 587,152 ✘；critical 800,000 ✔；atomic 800,000 ✔。

---

## 9. `parallel for` vs `omp for`（slides 33–34）

- `#pragma omp parallel for` – **建立** parallel region **并且**分配 loop iterations。
- `#pragma omp for` – **只**分配 iterations；必须写在**已经存在的 parallel region 里面**。

```c
#pragma omp parallel for            ≡       #pragma omp parallel
for (int i = 0; i < n; i++)                 {
    work(i);                                    #pragma omp for
                                                for (int i = 0; i < n; i++)
                                                    work(i);
                                            }
```
`parallel for` 做的事（slide 34）：(1) 建立 parallel region，产生一组 threads（数量由 `OMP_NUM_THREADS`、`num_threads()` 等决定）→ (2) 分配 iterations → (3) threads 同时执行 → (4) loop 结尾 implicit barrier → (5) threads 汇合。

| Directive | Implicit barrier | 在哪里 |
|---|---|---|
| `parallel for` | 有 | Loop 结尾 & region 结尾 |
| `omp for` | 有 | Loop 结尾 |
| `omp for nowait` | **没有** | Barrier 被拿掉 |

---

## 10. Loop 例子、reduction、false sharing（slides 35–48）

### Slide 35 — "Simple OpenMP loop compiled"
```c
extern float foo(void);
int main(int argc, char** argv) {
    int i; float r = 0.0;
    #pragma omp parallel for schedule(dynamic) reduction(+:r)
    for (i = 0; i < 10; i++) { r += foo(); }
    return 0;
}
```
*在做什么？* 把 `foo()` 的结果加 10 次进 `r`。*Runtime 背后怎么做？* **1. scheduling**（动态分 iteration）→ **2. 平行做事** → **3. reduction**（把各自的私有总和合并）→ **4. barrier**。

### 2×2 矩阵、2 个 threads（slide 36）
`omp_set_num_threads(2); #pragma omp parallel for` 对 row `i` 平行，inner loop `j`，`M[i][j] *= 3`。

| Thread | Iterations |
|---|---|
| 0 | (0,0), (0,1) |
| 1 | (1,0), (1,1) |

两个 thread 同时处理**不同的 row**。{{1,2},{3,4}} → **{{3,6},{9,12}}**。

### Reduction（归约）（slides 38–41）
`reduction(+:r)` 的步骤：
1. 给每个 thread 一份 `r` 的**私有 copy**。
2. 每个 thread 只更新**自己的** r——**不会冲突**。
3. Loop 结束后，用 `+` 把所有私有值**合并**。
4. 结果存回原本共享的 `r`。

Slides 40–41（5 个 threads、`schedule(dynamic)`、10 次 iteration、`foo()` 返回 1.0）：
```
T0: 1+1+1+1 = 4      T1: 1+1+1 = 3     T2: 1    T3: 1    T4: 1
r = 4 + 3 + 1 + 1 + 1 = 10      （Final r = 所有私有 r 的总和）
```
用 `dynamic`，先做完的 thread **不会闲着**——负载比较平均。用普通 `parallel for`（static），工作一开始就分好；T3 早做完就**只能闲着**。

> ✔ gcc：`reduction(+:r)` + dynamic、5 threads → r = 10.000000。

**💬 答题句 (EN):** *`reduction(+:r)` gives each thread a private copy of r, so threads update their own copies without a race condition; when the loop finishes, the private copies are combined with the + operator and the result is stored back into the shared variable r.*

**为什么这里 reduction 反而更慢？（slides 42–43）** 两间公司 25,000 个员工的薪水，单线程算要 **0.005156 s**；用 `omp for reduction` + `single` 要 **0.011162 s**——更慢。原因（slide 没写）：每个 iteration 的工作太小，**建立 threads、分配工作、合并私有 copy 的 overhead** 比省下的时间还多。工作量够大，平行才划算。

### False sharing 和 padding（slides 44–48）
每个 thread 写自己的 `sum[id]`。答案是对的——但很慢。为什么？
```
 没有 padding：sum[0] sum[1] sum[2] sum[3] ... 挨在一起
                └──────── 同一条 64-byte cache line ────────┘
 Thread 0 写 sum[0]，Thread 1 写 sum[1] → 同一条 cache line
 → 这条 line 在各个 core 的 cache 之间来回弹（"false sharing"）
```
**False sharing**（伪共享）：threads 写的是**不同**的变量，但这些变量刚好在**同一条 cache line**（64 bytes）里，硬件只好不断让彼此的 cache 失效。

**Padding（填充）：** `sum[16][8]`（long long = 8 bytes → 每 row 8 × 8 = 64 bytes）。Thread 0 用 `sum[0][0]` 加 7 个不用的位置，Thread 1 用 `sum[1][0]`……——每个 thread 的计数器相隔 **64 bytes → 在不同的 cache line**。
```c
static long long sum[2][PAD] = { 0 };        // PAD = 8
for (long long i = 0; i < iterations; i++)
    sum[id][0]++;                             // 每个 thread 写不同的 cache line
printf("Total increments = %lld\n", sum[0][0] + sum[1][0]);
```
Slide 46：`#define NUM_THREADS 8`、`#define PAD 8 // 8 ints = 64 bytes (1 cache line)`——⚠️ 数组元素是 `long`，不是 `int`；8 个 8-byte 的值 = 64 bytes（8 个 `int` 只有 32 bytes）。`single` 印一次 thread 数；总数 = 8 × 100,000 = 800,000。

---

## 11. Barriers（slides 49–58）— 优先度低（hint 说 "no barrier"）
**三种：** explicit barrier、没有 `nowait` 的 implicit barrier、有 `nowait` 的 implicit barrier（被拿掉）。

**x[ ]/y[ ] 例子（slides 50–53）：** 4 个 threads。Step 1：`x[id] = id * 10` → x = [0, 10, 20, 30]。Step 2：`y[id] = x[id] + x[(id+1) % 4]`。
- **没有 barrier：** Thread 0 可能在 Thread 1 写 x[1] **之前**就读了 → 垃圾值（VS 显示 −858993460）→ y[0] 是垃圾。
- **两步之间加 `#pragma omp barrier`：** 先把 x 全部写好。
  y[0] = 0+10 = **10**，y[1] = 10+20 = **30**，y[2] = 20+30 = **50**，y[3] = 30+0 = **30** → y = [10, 30, 50, 30]。✔ gcc 结果一样。

**nowait（slides 54–56）：** 两个 loop，先 `a[i]` 后 `b[i]`。没有 nowait：每个 thread 做完自己那部分 `a`，**等**大家，再开始 `b`。有 `#pragma omp for nowait`：做完 `a` 的 thread 直接去做 `b`（只有在 `b` 不需要别人的 `a` 时才安全）。

| Explicit barrier | Implicit barrier |
|---|---|
| `#pragma omp barrier`，程序员自己加 | 自动出现在 `for`、`sections`、`single`、`parallel` 结尾 |
| 自定义同步点 | 在 OpenMP 结构里自动同步 |

**优点：** 防止 data race（共享 data 全部更新完才继续）；保证正确性和顺序。
**缺点：** 大家都要等最慢的那个（straggler，落后者）→ 闲置时间；用太多有 overhead；有 thread 永远到不了 barrier 就 **deadlock**。

---

## 12. Distributed Shared Memory（DSM）（slides 59–62）

**问题：** shared memory 好写，但只能在一台机器里用。Cluster **没有**实体的共享 memory。

### DSM（分布式共享内存）
- **1989 年**提出的概念。
- 给 **multi-computer** 系统一个**逻辑上的共享 memory**（实体上分散，逻辑上共享）。
- 帮应用程序处理沟通和 **data consistency**（数据一致性）。
- 通常把**每台机器 local memory 的一部分 map 进 DSM**。
- 维护一个 **directory service**（目录服务），记录系统里所有 data 在哪里。

**两个管理决定（slide 60）：**
1. **How to distribute shared data?**（共享 data 怎么分布？）
2. **How many readers and writers** are allowed for a shared data segment?（一段共享 data 允许几个读者和写者？）

**分布共享 data 的方法（slide 61）：**

| Replication（复制） | Migration（迁移） |
|---|---|
| Keep **multiple copies** of shared data at various locations | **Move** the (single) copy to where it is needed — **only one copy allowed** |
| 适合很多读者；要保持 copy 一致 | 没有一致性问题，但 data 可能一直搬来搬去 |

**DSM 性能问题（slide 62）：**
- **Thrashing**（抖动）– data 一直在不同位置之间搬；怎么避免？
- **Block size** – 共享 block 多大最好（page size、packet size、segment size？）
- **Location of data** – 每个共享 data 最初放在哪里最好。
- **Implementation location** – 用 hardware、software 还是两者都用？

**💬 答题句 (EN):** *DSM gives the programs on a multi-computer system the illusion of one shared memory: local memories are mapped into a logical shared space, a directory tracks where each data item is, and data is either replicated (several copies kept consistent) or migrated (one copy moved to the requester).*

---

## 13. MPI — distributed-memory 编程（slides 63–91）

### 为什么用（slides 63–65）
Distributed memory：data 要**明确打包**送到另一台机器。用 MPI 时，程序员要自己把问题拆给 cluster，并确保讯息按正确顺序送和收。每个 process 有**自己的 memory、变量、数组——没有共享 memory**（跟 OpenMP 不同）。用途：天气预报、气候模拟、科学计算、分布式机器学习。例子（slide 65）：一个 grid 分给两个 process——**边界的格子**要互相交换（process 0 ↔ process 1），因为算一个格子需要它旁边的格子。

### 为什么用 MPI 而不是普通网络 library？（slides 66–71）

| Reason | 意思 |
|---|---|
| Optimised for performance | 比一般网络编程快 |
| Uses the fastest transport found | Shared memory（同一台机器）、快速互连（InfiniBand、Myrinet）、**TCP/IP 作后备** |
| Guarantees | **Reliable**（讯息不会丢）和 **in-order arrival**（A 送 1,2,3 → B 收 1,2,3） |
| Designed for multi-node technical computing | 为多节点科学计算设计 |
| Completely standard | 到处都有 |
| Collective operations | 有专门的集体操作（broadcast、reduce……） |

### 六个基本 function（slide 72）

| Group | Function | 意思 |
|---|---|---|
| Setup/teardown | `MPI_Init` | 启动 MPI；必须是第一个 MPI 调用 |
| | `MPI_Finalize` | 关闭 MPI |
| Who am I? | `MPI_Comm_size` | Process 总数（P） |
| | `MPI_Comm_rank` | 这个 process 的 ID（0 … P−1） |
| Message passing | `MPI_Send` | 送讯息给另一个 process |
| | `MPI_Recv` | 从另一个 process 收讯息 |

### Hello（slides 73–75）— *没跑过（没装 MPI）*
```c
#include "mpi.h"
#include <stdio.h>
int main(int argc, char *argv[]) {
    int rank, size;
    MPI_Init(&argc, &argv);              // 或 MPI_Init(NULL, NULL)——不传命令行参数
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    printf("Hello! I am %d of %d\n", rank, size);
    MPI_Finalize();
    return 0;
}
```
`mpiexec -n 2 hello.exe` → "Hello! I am 0 of 2"、"Hello! I am 1 of 2"。`MPI_COMM_WORLD` = 所有 process 的集合；所有 MPI function 都返回 error code；值是透过 **pointer** 传进去更新的。

### Send/Recv（slides 76–77）
```c
if (rank == 0) {
    int number = 123;
    MPI_Send(&number, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);   // 送给 rank 1，tag 0
    printf("Process 0 sent %d\n", number);
} else if (rank == 1) {
    int number;
    MPI_Recv(&number, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    printf("Process 1 received %d\n", number);
}
```
> ⚠️ **Slide 76 的 bug：** slide 写 `MPI_Recv(&number, 1, MPI_INT, 0, 0, MPI_COMM_WORLD);`——**只有 6 个参数**。`MPI_Recv` 需要第 7 个参数 status（`MPI_STATUS_IGNORE` 或 `&status`）；slide 的代码 compile 不过。

`mpirun -np 4 ./a.out` → "Process 0 sent 123"、"Process 1 received 123"；**rank 2 和 3 闲着**（代码里没有给它们的分支）。
"What just happened?"（slide 78）：mpiexec 开了 4 个 process，每个独立跑整个程序；也可以开 8、27、93……个；通常 process 数 = processor 数。

### OpenMP vs MPI（slide 79）

| OpenMP | MPI |
|---|---|
| Add **pragmas** to an existing program | Must **rewrite** the program to describe each process's data and communication |
| Compiler + runtime arrange parallel execution | **Explicit data movement** — programmer says what goes where and when |
| Relies on **shared memory** | Works on systems **without shared memory** |
| Many threads in **one** multicore computer | Many processes across **many** computers |

### 各种 framework 比较（slides 80–82）

| Method | Category | How created | Memory | Good for |
|---|---|---|---|---|
| MPI | Multiple processes | At program start (`mpirun`), all run `main` | **Isolated** | Clusters, distributed systems |
| `fork()` | Multiple processes | During execution; copy of address space | Isolated (after copy) | UNIX process creation |
| `pthread_create` | Threads | During execution | Shared | |
| OpenMP | Threads | At parallel regions (`#pragma`) | **Shared** | Easy parallel loops |

### Synchronous vs asynchronous messaging（slides 83–91）
**Synchronous（同步、blocking）：**
- `send()` 要等发送方收到**确认（acknowledgement）**——data 已经在接收方的 address space——才返回。
- `recv()` 要等 data 复制进接收方的 address space、并送出确认后才返回。
- ⚠️ 如果**所有** process 都先 send 再 receive，可能 **deadlock**（每个人都在等一个没人能给的确认）。
```
 Sender                        Receiver
 MPI_Send(number) ── msg ───►  MPI_Recv(number)  （复制进 number）
                  ◄── ack ───
 Sender 继续                    Receiver 继续
```
（每个 rank 的 `number` 在**不同**地址——例如 0x7ffc1a2c vs 0x7ffe9b4e——memory 是分开的。）

**Asynchronous（异步、non-blocking）：** 底层沟通交给额外的 threads 处理。
- `send()` **立刻返回**；送完之前**不能改** send buffer；这段时间可以做别的事；用 `checksend()` 看送完没有。
- `recv()` 先登记「我要收」然后立刻返回；用 `checkrecv()` 看 data 到了没有。

| MPI | Type | 意思 |
|---|---|---|
| `MPI_Send` | Synchronous | 送完之前 sender 不能继续 |
| `MPI_Recv` | Synchronous | 收到之前 receiver 不能继续 |
| `MPI_Isend` | Asynchronous | 开始送，之后在 `MPI_Wait` 确认完成 |
| `MPI_Irecv` | Asynchronous | 开始收，之后在 `MPI_Wait` 确认完成 |
| `MPI_Wait` | — | 等到这个请求完成（相当于 checksend/checkrecv） |

```c
int a = 123; MPI_Request req;
MPI_Isend(&a, 1, MPI_INT, 1, 0, MPI_COMM_WORLD, &req);
printf("Send started but not finished!\n");   // 送的同时做别的事
MPI_Wait(&req, MPI_STATUS_IGNORE);             // 还没完成就在这里等
printf("Send completed!\n");

int x; MPI_Request req2;
MPI_Irecv(&x, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, &req2);
printf("Waiting for message...\n");
MPI_Wait(&req2, MPI_STATUS_IGNORE);
printf("Received: %d\n", x);
```
（*技术补充：* 真正的 MPI 里，普通 `MPI_Send` 可能在 data 放进本地 buffer 后就返回；slides 说的「等确认」其实是 **synchronous mode**，即 `MPI_Ssend`。考试照 slides 答。）

**💬 答题句 (EN):** *In synchronous communication, MPI_Send does not return until the message has been received and acknowledged, and MPI_Recv does not return until the data is in the receiver's memory, so both processes wait; in asynchronous communication, MPI_Isend and MPI_Irecv return immediately so the process can do other work, and MPI_Wait is used later to check that the operation has completed.*

---

## 回到开场
薪水加总会错，是因为所有 threads 都对同一个共享变量做 `total += salary`——race condition。这章给的修法：`reduction(+:total)`（每个 thread 有自己的部分和，最后合并），或者在更新那句加 `atomic`/`critical`。修好了还是**更慢**——工作太小，overhead 太大；而且挨在一起的计数器会 **false sharing**。如果 data 大到一台机器放不下，就用 **MPI**：每个 process 算自己那份，再把部分和 **send** 给 rank 0。第 4 章会打开 `critical` 和 lock 背后*怎么运作*。

---

## ⚠️ Slides 有问题的地方

| Slide | Slide 写的 | 正确理解 |
|---|---|---|
| 76 | `MPI_Recv(&number,1,MPI_INT,0,0,MPI_COMM_WORLD);` | 少了第 7 个参数 `MPI_STATUS_IGNORE`——compile 不过 |
| 46 | `PAD 8 // 8 ints = 64 bytes` | 数组是 `long`（8 bytes）；8 个 int 只有 32 bytes，8 个 long 才是 64 bytes |
| 29 | 第四个打勾的例子 `#pragma omp critical {` 写在同一行 | 一般 `{` 放下一行；MSVC 接受，但考试写下一行比较保险 |
| 42 | 「为什么 reduction 更慢」——没解释 | 建立 threads + reduction 的 overhead 比那一点点工作还大 |
| 83 | Synchronous send 等确认 | 那是 MPI 的 *synchronous* mode（MPI_Ssend）；标准 MPI_Send 可能更早返回。考试照 slide 答 |
| 9 | "unmanegeable" | 打错字：unmanageable |

---

## 术语表

| English | 中文 | 一句话说明 |
|---|---|---|
| Bus contention | 总线争用 | 太多 processor 抢同一条 bus |
| Prefetching | 预取 | 提前把要用的 data 拿来 |
| Data race | 数据竞争 | 没同步就存取共享 data |
| Fork–join | 分叉-汇合 | Threads 分出去再汇合 |
| Work sharing | 工作分配 | 把 iteration/section 分给 threads |
| Load balancing | 负载均衡 | 让每个 thread 都一样忙 |
| Implicit / explicit barrier | 隐式 / 显式屏障 | 自动的 / 程序员写的等待点 |
| Mutual exclusion | 互斥 | 一次只有一个 thread 在里面 |
| Atomic operation | 原子操作 | 不可分割的操作 |
| Reduction | 归约 | 合并各自的部分结果 |
| False sharing | 伪共享 | 不同变量在同一条 cache line |
| Padding | 填充 | 用空位把 cache line 隔开 |
| Straggler | 落后者 | 大家都在等的最慢 thread |
| Replication / migration | 复制 / 迁移 | 多个 copy / 搬动唯一的 copy |
| Thrashing | 抖动 | Data 不停被搬来搬去 |
| Rank | 进程编号 | MPI process 的 ID |
| Blocking / non-blocking | 阻塞 / 非阻塞 | 会等 / 立刻返回 |

## 速记卡
- `parallel` = 每个 thread 跑整块（4 threads × 5 iterations = 20）。`parallel for` = 分 iteration（5）。
- `sections` = 不同的代码块，各执行一次，由不同 thread 做；thread 多过 section 就有人闲着。
- `single` = 一个（任意）thread 做一次；结尾有 implicit barrier。
- `critical` = 一整块、用 lock、较慢、互斥、**没有** barrier。`atomic` = 一个简单更新、用硬件、较快、不能加大括号。
- for/sections/single → 有 implicit barrier、没有互斥；critical/atomic → 有互斥、没有 barrier。
- `omp for` 一定要在 `parallel` 里面；`nowait` 拿掉 barrier。
- static = 事先分成一段段（0-2, 3-5, 6-7, 8-9）；static,1 = 轮流；dynamic = 做完再拿。
- reduction(+:r) = 每人一份私有 copy → 最后用 + 合并。
- False sharing → padding 隔开 64 bytes。
- DSM：在很多台电脑上做出逻辑上的共享 memory；replication vs migration；thrashing、block size、location、implementation。
- MPI 六个：Init、Finalize、Comm_size、Comm_rank、Send、Recv。Sync（Send/Recv）vs async（Isend/Irecv + Wait）。

---

## 练习题（附答案）
*题目用英文，答案附华文说明。*

### A. MCQ
1. `#pragma omp parallel num_threads(3)` around `for(i=0;i<4;i++) printf(...)` prints how many lines? (a) 3 (b) 4 (c) 7 (d) 12
2. Which directive has NO implicit barrier? (a) single (b) sections (c) critical (d) for
3. Which is invalid? (a) `#pragma omp atomic` + `x++;` (b) `#pragma omp critical` + `{ a++; b++; }` (c) `#pragma omp atomic` + `{ a++; b++; }` (d) `#pragma omp single` + `{ printf("Hi"); }`
4. `schedule(static,1)`, 3 threads, 7 iterations: Thread 1 gets (a) 1,4 (b) 2,3 (c) 1,4,7 (d) 3,4,5
5. In DSM, "only one copy of shared data is allowed" describes (a) replication (b) migration (c) thrashing (d) directory

**答案：** 1-d（3 × 4，每个 thread 都跑整个 loop）；2-c；3-c（atomic 不能接 block）；4-a（iterations 0–6：T0 0,3,6；T1 1,4；T2 2,5）；5-b。

### B. Short answer
1. Explain the purpose of `#pragma omp sections` and what happens with 2 threads and 3 sections.
   *It lets threads share different code blocks: each section runs once by one thread, sections run in parallel. With 2 threads and 3 sections, two sections start immediately and the third waits until a thread becomes free; all must finish (implicit barrier) before the code after continues.*
2. Why does `single` print "Hello" before every "After single"?
   *Only one thread executes the single block; there is an implicit barrier at the end of single, so the other threads wait there until it completes, then all print "After single".*
   *华文：因为 single 结尾有 barrier，大家等它做完才一起往下。*
3. Give TWO reasons to use MPI rather than plain socket programming.
   *Optimised for performance and automatically uses the fastest transport (shared memory, InfiniBand, TCP fallback); guarantees reliable, in-order delivery; standard and has collective operations.*

### C. Application
1. Output count and order rules:
```c
#pragma omp parallel num_threads(4)
{
    #pragma omp single
    printf("A\n");
    #pragma omp critical
    printf("B %d\n", omp_get_thread_num());
    printf("C\n");
}
printf("D\n");
```
*Answer:* "A" once and first (single + barrier); "B x" 4 times, one at a time, any thread order; "C" 4 times, may interleave with B lines (critical has no barrier); "D" once and last (end of parallel region barrier).
*华文：A 一次且最先；B 四次、一个一个来；C 四次，可能夹在 B 中间；D 一次且最后。*
2. `schedule(static)`, 4 threads, 10 iterations — which iterations does Thread 2 run? *6, 7.* With `static,1`? *2, 6.*
3. `reduction(+:r)`, `foo()` returns 2.0, 12 iterations, 3 threads with dynamic schedule doing 5, 4, 3 iterations. Show the private values and final r.
   *T0 = 10, T1 = 8, T2 = 6 → r = 10 + 8 + 6 = 24.*

### D. Thinking
1. A program does `#pragma omp parallel for` with `total += a[i];`. Explain the bug and give TWO fixes.
   *Race condition on shared `total` (read–add–write interleaves, updates lost). Fix 1: `reduction(+:total)`. Fix 2: `#pragma omp atomic` before `total += a[i];` (or critical; slower).*
2. Two MPI processes both call blocking `MPI_Send` to each other, then `MPI_Recv`. What can happen and how to fix?
   *With synchronous sends, each waits for the other to receive → deadlock. Fix: rank 0 sends then receives while rank 1 receives then sends, or use MPI_Isend/MPI_Irecv + MPI_Wait.*
   *华文：两个人都在等对方先收，结果谁都动不了；解法是让一个先送一个先收，或者用非阻塞版本。*

---

## Slide 索引
| 章节 | Slides |
|---|---|
| 1–3 Models、threads | 3–12 |
| 4 parallel vs parallel for | 13–17 |
| 5 sections | 18–20 |
| 6 single | 21–22 |
| 7 schedule | 23–27 |
| 8 single/critical/atomic | 28–32 |
| 9 omp for、nowait | 33–34 |
| 10 Loop、reduction、padding | 35–48 |
| 11 Barrier | 49–58 |
| 12 DSM | 59–62 |
| 13 MPI | 63–91 |

## 跟其他章的连接
- Ch1：`#pragma omp parallel`、race 示范（slides 37–42）、OpenMP vs MPI vs CUDA。
- Ch2：message passing vs shared memory。
- Ch4：`critical` 背后是什么（critical-section problem）、`omp_lock_t`、semaphore。
