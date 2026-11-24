# AMCS2103 Parallel & Distributed Computing — 考试指南（Chapter 1–4，双语版）

这次没有 past-year paper，所以这份指南是根据**讲师给的考试 hint**（你那两张截图）和 slides 整理的。Hint 写了 "few set (3)"，看起来大概有三组题。"No barrier" 表示不太会考 barrier。

## 1. Hint → 答案在哪里

| 讲师 hint | 笔记章节 | 要会做什么 |
|---|---|---|
| Traditional database / Distributed database（"distributed storage places data across multiple networked servers"） | Ch1 §1 | 写出两个定义；画 4 行的比较表 |
| Benefits（scalability、reliability、performance、geographical） | Ch1 §2 | 解释 3 个 benefit 加例子；vertical vs horizontal scaling + 价钱表 |
| Loosely & tightly coupled difference（pg 20、pg 38/43） | Ch1 §4.1–4.4 | 画两种架构；3 行以上的比较表 |
| OpenMP vs CUDA vs MPI（pg 40/45） | Ch1 §5 | 比较表（model、architecture、hardware、unit、memory、communication） |
| Cristian's algorithm（show steps） | Ch1 §8.1 | Trec = C + (T1 − T0)/2，完整算式；3 个问题 |
| OpenMP – 解释 directive、`#pragma omp parallel` 的用途（不用写 code；不考 MPI） | Ch1 §4.5、Ch3 §4 | 解释 fork–join、thread team、implicit barrier；预测输出 |
| Ch3 pg 22 "section" / pg 29、31 `#pragma` | Ch3 §5、§6、§8 | sections vs single vs for vs critical/atomic 表；critical 和 atomic 代码哪些对哪些错 |
| Pipes（Ch2）：buffer 空？读到什么时候？怎样 block consumer？ordinary vs named | Ch2 §4 | 空 → read() 阻塞；返回 data，写端全关后返回 0（EOF）；默认就是 blocking |
| Ch4 pg 7 – 怎么算 parallel / processing time（最多 3 个 processor） | Ch4 §2 | Dependency graph → Ts → Gantt → Tp → S → E |
| Ch4 pg 84 extra question | Ch4 §2 满分答法 | W = …：Ts 41、Tp 23、S 1.78、E 59.33 % |
| Ch4 pg 44 critical & atomic | Ch4 §5、Ch3 §8 | 「保护一整块 vs 保护一个操作」 |
| omp_lock_t（pg 61） | Ch4 §10 | 解释 init/set/unset/destroy；最后余额 1900 |

## 2. 复习优先顺序
1. Ch4 §2 计算题（两个例题 + 练习题一直做，做到 10 分钟内能完成一题）。
2. Ch1 §8.1 Cristian（§8.2 Berkeley 当备用）。
3. Ch1 §4.4 loosely vs tightly 表和 §5 OpenMP/CUDA/MPI 表。
4. Ch3 §8 对比表 + critical vs atomic 代码；Ch3 §4–6 parallel / sections / single 的输出。
5. Ch4 §10 omp_lock_t 满分答法。
6. Ch2 §4 pipe 问题满分答法。
7. Ch1 §1–2 database 和 benefits。
8. 其他全部（Ch4 semaphore/Peterson、Ch3 DSM/MPI、Ch2 sockets），应付短答题。

## 3. 一天复习计划

| 时间 | 做什么 |
|---|---|
| 1 h | Ch4 §2 — 不看笔记，在纸上重画两个 Gantt chart |
| 1 h | Ch1 §8 + Ch1 练习 C1、C2 |
| 1.5 h | Ch1 的表（§1、§2、§4.4、§5）— 每个凭记忆写一次 |
| 1.5 h | Ch3 §4–§9 — 预测输出，背 slide 31 的表 |
| 1 h | Ch4 §5、§10（critical/atomic、omp_lock_t） |
| 45 min | Ch2 §4 pipes |
| 45 min | 做完所有练习题，对答案 |

## 4. 答题技巧
- **「Show steps」**——先写公式，再代数字，最后算（例如 Trec = C + (T1 − T0)/2 = 20:05:57 + 8/2 = 20:06:01）。
- **Gantt chart**——标好 P1–P3，每个 bar 里写 task 名和时间，时间轴标出每个开始/结束点，然后写出 critical path。
- **「Differentiate / compare」**——用表格；一行一个 point，两边比较同一个方面。
- **「Explain the directive」**——讲清楚*它建立什么*、*谁执行什么*、*有没有 barrier*、*有没有 mutual exclusion*。
- 用讲师的用词：*fork–join、implicit barrier、mutual exclusion、race condition、critical path*。
- 双语笔记里的 **💬 答题句 (EN)** 是给你直接背、直接写的；华文部分是帮你理解，考试**写英文**。

## 5. Slides 里的错误（不要照抄）

| 章 / slide | 问题 |
|---|---|
| Ch1 slide 64 | Cristian 的答案写成 10:06:01；正确是 **20:06:01** |
| Ch3 slide 76 | `MPI_Recv` 少了 `MPI_STATUS_IGNORE` 参数 |
| Ch3 slide 46 | "8 ints = 64 bytes"——应该是 8 个 long long |
| Ch4 slide 63 | Spinlock 先检查再设定，不是 atomic |
| Ch4 slides 11、88 | Efficiency 用了四舍五入后的 speedup（49.33 % vs 49.28 %；59.33 % vs 59.42 %） |
| Ch2 目录 | 列了 "Middleware"，但没有 slide |

## 6. 你朋友的 revision 资料要改的地方
如果你也读你朋友那两份 PDF，先用笔改掉这 5 个错：
1. Cristian 第 3 步：Ttrans **不是** T1 − T0，是 (T1 − T0)/2；名字是 **Cristian** 不是 Christian。
2. Critical **不是** FIFO——OpenMP 不保证谁先进。
3. Reduction **不能**用 `/`。
4. Ordinary pipe 要 parent–child 的原因：它**没有名字**，只能靠 `fork()` 继承 fd。
5. Named pipe 双向通常要**两个** FIFO。

## 7. 验证过什么、没验证什么
- **用 gcc（Linux）compile 跑过：** ordinary pipe、named pipe、空 pipe 阻塞/EOF 测试、UDP client/server、OpenMP race vs critical vs atomic vs lock、schedule(static)/(static,1) 的分配、single、sections、reduction、barrier 的 x/y 例子、omp_lock_t 余额（1900）。
- **所有计算都重算过：** Cristian（10:05:22、20:06:01）、Berkeley（3:05）、两题 Ts/Tp/S/E、所有练习题答案。
- **没跑过：** MPI 和 CUDA 代码（这里没安装 MPI/CUDA）——从 slides 抄下来，逐行看过检查。
- **每一页 slide 都当图片看过**（共 293 页），包括代码截图和图表。
