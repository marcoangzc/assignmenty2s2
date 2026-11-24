# AMCS2103 — Chapter 4: Concurrency Control (Part I)（双语版）

> **怎么读：** 讲解用华文，terms 和代码保留英文；**💬 答题句 (EN)** = 考试直接写的英文句子。
>
> Slides 把平行时间的计算直接画成做好的 Gantt chart，但从没讲*每个 task 要怎么放*；semaphore 用动画一步步演示，印出来就很难看懂。这份笔记给出 **task dependency graph → Ts → Gantt chart → Tp → speedup → efficiency** 的固定步骤（做了两题，包括 slide 84 的 Extra Question），逐行 trace 每个 race condition 和 semaphore 例子，并完整解释 `omp_lock_t`（slide 61）。OpenMP lock 程序已经 compile 跑过。

---

## 0. 本章总览

**一句话：** 几个 process 同时跑时，(1) 可以算出它们快了多少；(2) 必须防止它们把共享 data 弄坏——用 critical section、lock 和 semaphore。

| # | 主题 | Slides | 考试重点（讲师 hint） |
|---|---|---|---|
| 1 | Concurrent processing、COBEGIN/COEND | 3–4 | |
| 2 | **Task dependency graph、Ts、Tp、speedup、efficiency** | 5–11, 84–88 | ⭐⭐「pg 7」「pg 84 extra question」「怎么算 parallel time」（最多 3 个 processor） |
| 3 | Loop parallelism（DO loop） | 12–16 | |
| 4 | Producer–consumer 和 race condition | 17–41 | ⭐ |
| 5 | OpenMP critical vs atomic（counter） | 42–44 | ⭐「pg 44 critical & atomic」 |
| 6 | Print spooler race | 45–46 | |
| 7 | Critical-section problem + 3 个条件 | 47–51 | |
| 8 | Peterson's solution | 52–56 | |
| 9 | Disabling interrupts | 57–59 | |
| 10 | Locks：spinlock、atomic lock、**`omp_lock_t`** | 60–64, 78–79 | ⭐⭐「omp_lock_t (pg 61)」 |
| 11 | Semaphores、bounded buffer、no-busy-wait 实现 | 65–77 | |
| 12 | Deadlock 和 starvation、semaphore 用错 | 80–83 | |

讲师：这部分**不考 barrier**。

---

## 开场
你和朋友共用一个银行户口，里面有 RM1,000。同一秒，你在 ATM 领 RM200，朋友在网上领 RM100。两台电脑各自读到「RM1,000」，各自减，各自写回。结果户口剩 RM800 或 RM900——就是不会是正确的 RM700。同时，银行想把晚上的利息计算放到 3 个 processor 上跑来加快：最多能快多少？

---

## 1. Concurrent processing（slides 3–4）
**Parallel processing（multiprocessing）** = 2 个或以上的 processor **协同运作**（in unison）：
- 2 个以上的 CPU 同时执行指令；每个 CPU 可以同时有一个 process 在 RUNNING 状态。
- **Processor Manager** 协调每个 processor，并同步 CPU 之间的互动。
- **Synchronisation（同步）**很关键——multiprocessing 系统里很多地方会出错。

**COBEGIN / COEND（slide 4）**——标出可以同时执行的语句：
```
Compute A = (3 x B) + [4/(D+E)] x (F-G)
COBEGIN
    T1 = 3 x B        → CPU 1
    T3 = D + E        → CPU 2      （三个互不相关的运算，一起跑）
    T5 = F - G        → CPU 3
COEND
T4 = 4 / T3
T6 = T4 x T5
A  = T1 + T6
```
OpenMP 里同样的概念是 `#pragma omp parallel sections`，一个 task 一个 `section`。

---

## 2. Task dependency graph、Ts、Tp、Speedup、Efficiency ⭐⭐（slides 5–11）

### 方法（每题都照这个步骤做）
1. **把式子拆成 task**，一个运算符一个 task，按运算顺序（括号 → × / → + −）。
2. **画 task dependency graph**（任务依赖图）：如果 Y 要用 X 的结果，就画 X → Y。
3. **Ts（sequential time，顺序执行时间）** = **所有 task 时间加起来**（一个 processor 一件一件做）。
4. **画 Gantt chart（甘特图）求 Tp：** 把每个 task 分给一个 processor。**Task 的开始时间 = 它依赖的 task 里最晚结束的那个时间**（而且那个 processor 要有空）。一层一层排。
5. **Tp（parallel time，平行执行时间）** = 最后一个 task 结束的时间 = **critical path**（关键路径，最长的一条依赖链）的长度（processor 够用的话）。
6. **Speedup S = Ts / Tp**；**Efficiency E = S / p × 100%**（p = processor 数）。

### 例题（slides 5–11）：A = (3 × B) + [4 / (D + E)] × (F − G)

| Operation | Time (ms) |
|---|---|
| Multiplication (×) | 8 |
| Subtraction (−) | 3 |
| Addition (+) | 4 |
| Division (/) | 7 |

**Step 1 – 拆 task**

| Task | Operation | Time |
|---|---|---|
| T1 | 3 × B | 8 |
| T2 | D + E | 4 |
| T3 | F − G | 3 |
| T4 | 4 / T2 | 7 |
| T5 | T4 × T3 | 8 |
| A | T1 + T5 | 4 |

**Step 2 – Task dependency graph（slide 5）**
```
   3 x B          D + E          F - G
   (T1)           (T2)           (T3)
     |              |              |
     |           4 / T2            |
     |            (T4)             |
     |              |              |
     |              +---- T4 x T3 -+
     |                     (T5)
     |                       |
     +------- A = T1 + T5 ---+
```
**看点：** T1、T2、T3 不依赖任何人 → 都可以在时间 0 一起开始。

**Step 3 – Ts（slide 6）**
Ts = 8 + 4 + 3 + 7 + 8 + 4 = **34 ms**

**Step 4 – Gantt chart，3 个 processor（slides 7–10）**

算每个 task 的开始时间：
- T1、T2、T3 在 0 开始 → T1 在 8 结束，T2 在 4 结束，T3 在 3 结束。
- T4 要等 T2 → **4** 开始，4 + 7 = **11** 结束。
- T5 要等 T4 和 T3 → max(11, 3) = **11** 开始，11 + 8 = **19** 结束。
- A 要等 T1 和 T5 → max(8, 19) = **19** 开始，19 + 4 = **23** 结束。

```
      0   3 4      8    11              19      23
      |   | |      |     |               |       |
 P1   [==== T1 = 3xB (8) ====]
 P2   [T2 D+E(4)][== T4 4/T2 (7) ==][=== T5 T4xT3 (8) ===][A T1+T5 (4)]
 P3   [T3(3)]
      0   3 4      8    11              19      23   time (ms)
```
**看点：** 讲师画了**四个版本**（slides 7–10），T4、T5、A 放在不同 processor（P2/P3/P1）——答案都一样，**Tp = 23 ms**，因为决定时间的是依赖关系，不是放在哪个 processor。

**Critical path 检查：** T2 → T4 → T5 → A = 4 + 7 + 8 + 4 = **23 ms** ✔。
（T1 **不在** critical path 上——它 8 就做完了，但 A 要等到 19。）

**Step 5 – Speedup 和 efficiency（slide 11）**
- S = Ts / Tp = 34 / 23 ≈ **1.48**
- E = S / p × 100 = 1.48 / 3 × 100 ≈ **49.33 %**（moderate efficiency，中等）
  *（不先四舍五入：34/23 = 1.4783 → 49.28 %。有写算式的话两个都可以；slide 是先把 S 四舍五入。）*

| Speedup | 意思 |
|---|---|
| S = 1 | 顺序和平行一样快 |
| S > 1 | 平行比较快 |
| S < 1 | 平行反而比较慢 |
| S = 2 | 平行快 2 倍 |
| S = 0.5 | 平行要花 2 倍时间 |

| Efficiency | 解读 |
|---|---|
| 100 % | Ideal（理想） |
| 90 % | Excellent |
| 80 % | High |
| 50 % | Moderate |
| 20 % | Low |
| 5 % | Very low |

**为什么 efficiency 只有约 49%？** Processor 在**闲着**：P1 在 8 ms 之后没事做，P3 在 3 ms 之后没事做，因为那条很长的依赖链 T2→T4→T5→A 只能一步一步来。

### ✅ 满分答法 — Extra Question（slides 84–88）：W = (A × B × 5) − (C / D) + (E − F) × G，3 个 processor

| Operation | × | − | / | + |
|---|---|---|---|---|
| Time (ms) | 7 | 5 | 6 | 4 |

**1. Task dependency graph**

| Task | Operation | Time |
|---|---|---|
| T1 | A × B | 7 |
| T2 | C / D | 6 |
| T3 | E − F | 5 |
| T4 | T1 × 5 | 7 |
| T5 | T3 × G | 7 |
| T6 | T4 − T2 | 5 |
| W | T6 + T5 | 4 |

```
   A x B          C / D          E - F
   (T1)           (T2)           (T3)
     |              |              |
   T1 x 5           |            T3 x G
   (T4)             |            (T5)
     |              |              |
     +-- T4 - T2 ---+              |
          (T6)                     |
            |                      |
            +---- W = T6 + T5 -----+
```

**2. Ts** = 7 + 6 + 5 + 7 + 7 + 5 + 4 = **41 ms**

**3. Gantt chart 和 Tp**
- T1（P1）0–7，T2（P2）0–6，T3（P3）0–5。
- T4 要等 T1 → P1 7–14。T5 要等 T3 → P3 5–12。
- T6 要等 T4 和 T2 → max(14, 6) = 14 开始 → P2 14–19。
- W 要等 T6 和 T5 → max(19, 12) = 19 开始 → P2 19–23。

```
      0    5 6 7        12  14        19      23
 P1   [T1 AxB (7)][T4 T1x5 (7)]
 P2   [T2 C/D(6)]               [T6 T4-T2 (5)][W T6+T5 (4)]
 P3   [T3 E-F(5)][T5 T3xG (7)]
      0    5 6 7        12  14        19      23   time (ms)
```
Critical path：T1 → T4 → T6 → W = 7 + 7 + 5 + 4 = **23 ms** → **Tp = 23 ms**。

**4. Speedup 和 efficiency**
- S = Ts / Tp = 41 / 23 ≈ **1.78**
- E = 1.78 / 3 × 100 ≈ **59.33 %**（不先四舍五入是 59.42 %）

> ✔ 所有数字都重新算过：34/23 = 1.478，41/23 = 1.783。
>
> 中文提示：**Gantt chart 一定要标** processor（P1–P3）、每个 bar 里写 task 名和时间、时间轴标出每个开始/结束点，最后写出 critical path。

**💬 答题句 (EN):** *The parallel execution time is determined by the critical path — the longest chain of dependent tasks — because a task cannot start until all the tasks it depends on have finished, no matter how many processors are free.*

---

## 3. Loop parallelism（slides 12–16）
```
DO i = 1, 3
    A(i) = B(i) + C(i)
ENDDO
```
每个 iteration **互不相关**，所以 Processor 1 做 A(1) = B(1) + C(1)，Processor 2 做 A(2)，Processor 3 做 A(3)——同时进行。
OpenMP 版本（slides 13–16）：`int A[3], B[3] = {1,2,3}, C[3] = {4,5,6}; #pragma omp parallel for for (i=0;i<3;i++) A[i] = B[i] + C[i];` → A = {5, 7, 9}；输出行（Thread 0: A[0] = 1 + 4 = 5 …）每次**顺序不同**；"After loop" 永远最后。

---

## 4. Producer–Consumer 和 race condition ⭐（slides 17–41）

### Critical region（slide 17）
所有同步方法的共同想法：让一个 process **做完它在 critical region 里的工作**，别人才能进来。Multiprocessor 和单一 CPU 分时（time-shared）都适用。叫 *critical region*，是因为它必须**当作一个整体（ONE UNIT）**来执行。

### Producer–consumer（slides 18–20）
一个 process **生产** data，另一个 process 之后**消费**它，中间透过一个大小有限的 buffer。同步必须做到：
- buffer **满了**，**让 producer 等**；
- buffer **空了**，**让 consumer 等**。
用 **3 个 semaphore** 来做：**full** 格数、**empty** 格数，和一个 **mutex**（MUTual EXclusion，互斥锁），确保一次只有一个 process 碰 buffer。

### 用 `count` 的做法（slides 21–25）
```c
// Producer                                  // Consumer
while (true) {                               while (true) {
    /* produce nextProduced */                   while (count == 0)
    while (count == BUFFER_SIZE)                     ;   // 什么都不做（buffer 空）
        ;   // 什么都不做（buffer 满）            nextConsumed = buffer[out];
    buffer[in] = nextProduced;                   out = (out + 1) % BUFFER_SIZE;
    in = (in + 1) % BUFFER_SIZE;                 count--;
    count++;                                     /* consume nextConsumed */
}                                            }
```
`count` 从 0 开始；producer 生产后加 1；consumer 消费后减 1。`in` = 下一个空格，`out` = 下一个满格；`% BUFFER_SIZE` 让 buffer 变成**环形**（circular）。

### 为什么会坏掉（slides 26–33）
`count++` 和 `count--` 在机器码里**不是一步**：
```
count++  :  register1 = count            count--  :  register2 = count
            register1 = register1 + 1                register2 = register2 - 1
            count = register1                        count = register2
```
**交错执行的 trace，count = 5（slide 26）：**

| Time | Who | Instruction | register1 | register2 | count |
|---|---|---|---|---|---|
| t0 | Producer | register1 = count | 5 | | 5 |
| t1 | Producer | register1 = register1 + 1 | 6 | | 5 |
| t2 | Consumer | register2 = count | 6 | 5 | 5 |
| t3 | Consumer | register2 = register2 − 1 | | 4 | 5 |
| t4 | Producer | count = register1 | | | **6** |
| t5 | Consumer | count = register2 | | | **4** |

正确答案：生产一个、消费一个 → **5**。结果是 **4**（如果 t4 和 t5 对调就是 **6**）。

**Race condition（竞争条件）（slide 34）：** *several processes access and manipulate the same data concurrently and the outcome depends on the particular order in which the access takes place.*（几个 process 同时存取和修改同一份 data，结果取决于存取的先后顺序。）会发生是因为两个 process 都被允许同时修改 `count`。如果分开跑，count 会正确地是 5（slide 35）。

Slides 36–41 用动画演示 buffer：producer 加入 A–F（count 5 → 6），consumer 拿走 A（count 6 → 5），`in` 和 `out` 在环形 buffer 上往前移。

**💬 答题句 (EN):** *count++ is compiled into load, increment and store; if the producer and consumer interleave these steps, one update overwrites the other, so count becomes 4 or 6 instead of 5 — a race condition, because the result depends on the order of execution.*

---

## 5. OpenMP 的修法：critical vs atomic（slides 42–44）⭐
```c
int counter = 0;
#pragma omp parallel num_threads(2) shared(counter)
{
    #pragma omp critical
    {
        counter++;
        printf("Thread %d: Counter = %d\n", omp_get_thread_num(), counter);
    }
}
printf("Final counter = %d\n", counter);        // Thread 0: 1, Thread 1: 2, Final 2
```
```c
#pragma omp parallel num_threads(2) shared(counter)
{
    #pragma omp atomic
    counter++;                                      // 只有这句被保护
    printf("Thread %d increments counter\n", omp_get_thread_num());
}
// Final counter = 2
```

| `#pragma omp critical` | `#pragma omp atomic` |
|---|---|
| Protecting the **region**（保护一整块） | Protecting the **operation**（保护一个操作） |
| `counter++; printf(...);` | `counter++;` |
| **Both** statements protected | **Only** `counter++` protected |

银行转账（slides 48–50）：`accountA = 1000, accountB = 500`，2 个 threads，各从 A 转 RM100 到 B。
- 用 `critical` 包住两句加 printf：Thread 0 → A=900, B=600；Thread 1 → A=800, B=700。**最后 A = RM800，B = RM700** ✔。
- 用**两个分开的** `atomic`（`accountA -= 100;` 和 `accountB += 100;`）：最后余额还是 800/700（每次更新都安全），但两个更新**不是一个整体**——别的 thread 可能看到 A 已经扣了、B 还没加的中间状态。几句必须一起改的时候用 `critical`。

---

## 6. Print spooler race（slides 45–46）
Spooler directory 有格子 0, 1, 2, …；共享变量 **`out`** = 下一个要印的文件，**`in`** = 下一个空格。目前 out = 4，in = 7。
```
 Process A：读 in (7) → 存进 A_NextFreeSlot = 7
            ── 被打断 ──
 Process B：读 in (7) → B_NextFreeSlot = 7 → 把 "file b" 放进格子 7 → in = 8
 Process A：（继续，手上还是 7）把 "file a" 放进格子 7 → in = 8
```
格子 7 现在是 **file a**；file b 被**盖掉**了。`in` = 8 看起来很正常，所以 printer daemon 完全察觉不到——**Process B 永远拿不到它的打印**。

---

## 7. The Critical-Section Problem（slides 47–51）
**Critical section（临界区）：** process 里修改共享变量、更新表格、写文件等的那段代码。规则：**两个 process 不能同时在各自的 critical section 里。**

每个 process 进去前都要**请求许可**：
```
do {
    [ entry section ]       ← 请求许可（lock / wait）
      critical section
    [ exit section ]        ← 释放（unlock / signal）
      remainder section
} while (TRUE);
```
例子（slide 48）：共享 `int Balance;`——Process A：`Balance = Balance − 200;` Process B：`Balance = Balance − 100;` 每一句都是 critical section，前后包着 entry/exit section。

**解法必须满足的三个条件（slide 51）** ⭐

| Requirement | 中文意思 |
|---|---|
| 1. **Mutual exclusion**（互斥） | Pi 在 critical section 里时，**其他 process 都不能**在它们的 critical section 里 |
| 2. **Progress**（推进） | 没人在 critical section、又有人想进去时，决定谁下一个进去这件事**不能无限期拖延** |
| 3. **Bounded waiting**（有限等待） | 一个 process 提出请求之后、被批准之前，其他 process 进入 critical section 的次数**有上限**（没有人会永远等） |

假设：每个 process 以**非零速度**运行；**不假设** N 个 process 之间的相对速度。

**💬 答题句 (EN):** *A solution to the critical-section problem must satisfy mutual exclusion (only one process in its critical section at a time), progress (if no process is in its critical section, the selection of the next process to enter cannot be postponed indefinitely) and bounded waiting (there is a limit on how many times other processes can enter before a waiting process is allowed in).*

---

## 8. Peterson's Solution（slides 52–56）
一个**两个 process** 的软件解法。假设 LOAD 和 STORE 是 **atomic**（不会被打断）。
共享变量：`int turn;`——轮到谁；`boolean flag[2];`——`flag[i] = true` 表示 Pi **准备好**（想进去）。

```c
// Process Pi（j = 另一个 process）
do {
    flag[i] = TRUE;                  // 我想进去
    turn = j;                        // 但如果你也想，你先
    while (flag[j] && turn == j)     // 对方想进 而且 轮到对方 → 我就等
        ;                            // busy wait（忙等）
        /* critical section */
    flag[i] = FALSE;                 // 我做完了
        /* remainder section */
} while (TRUE);
```
**Trace（slides 55–56）：两个同时想进去**

| Step | P0 | P1 | flag[0] | flag[1] | turn |
|---|---|---|---|---|---|
| 1 | flag[0] = T | | T | F | – |
| 2 | | flag[1] = T | T | T | – |
| 3 | turn = 1 | | T | T | 1 |
| 4 | | turn = 0 | T | T | **0** |
| 5 | 检查：flag[1] && turn==1？→ T && F → **进入 CS** | 检查：flag[0] && turn==0？→ T && T → **busy wait** | | | |
| 6 | 出来：flag[0] = F | 检查 → F → **进入 CS** | F | T | 0 |

**最后写 `turn` 的那个 process 会让步**——所以只有一个能进去（mutual exclusion），另一个接着进（progress、bounded waiting）。

---

## 9. Disabling interrupts（slides 57–59）
**想法：** 在**单一 processor** 上，修改共享变量时把 interrupt 关掉 → 当前的指令会按顺序执行，**不会被抢占（preemption）**。**Non-preemptive kernel** 用这招。

**「为什么不用 disable interrupts？」（slide 59）— 答案：**
1. 如果用户程序可以关 interrupt，它就能关掉 **timer interrupt**，停止 context switching，**霸占 CPU**。
2. 在 **multiprocessor** 上，要关所有 CPU 的 interrupt，得**传讯息给每个 processor**——**很花时间**，每次进 critical section 都被拖慢，效率下降。
3. 关 interrupt **只影响执行这条指令的那个 CPU**；其他 processor 照常跑，**还是能存取共享 memory**——所以根本做不到 mutual exclusion。

**✅ 满分答法 (EN):** *Disabling interrupts is unsuitable because (1) if user programs could disable interrupts, a process could disable the timer interrupt and monopolise the CPU; (2) on a multiprocessor, disabling interrupts on every processor requires sending a message to all processors, which is time-consuming and delays entry into each critical section; (3) disabling interrupts only affects the processor that executes the instruction, so other processors can still access the shared memory and mutual exclusion is not guaranteed.*

---

## 10. Locks ⭐⭐（slides 60–64, 78–79）

### Lock 的模式（slide 60）
```
do {
    acquire lock（取得锁）
        critical section
    release lock（释放锁）
        remainder section
} while (TRUE);
```
很多系统为此提供**硬件支持**。**OpenMP 类似 mutex 的机制就是 `omp_lock_t`。**

### `omp_lock_t` 程序（slide 61）⭐
```c
#include <stdio.h>
#include <omp.h>

int balance = 0;          // 共享变量
omp_lock_t myLock;        // 宣告一个 OpenMP lock

int main() {
    omp_init_lock(&myLock);                 // 1. 初始化 lock（未锁状态）

    #pragma omp parallel num_threads(10)
    {
        int tid = omp_get_thread_num();
        int amount;
        if (tid == 0) amount = 100;         // thread 0 存 RM100
        else          amount = 200;         // threads 1–9 各存 RM200

        omp_set_lock(&myLock);              // 2. 取得 lock（别人拿着就等）
        balance += amount;                  //    critical section
        omp_unset_lock(&myLock);            // 3. 释放 lock
    }

    omp_destroy_lock(&myLock);              // 4. 销毁 lock
    printf("Final balance = %d\n", balance);
    return 0;
}
```
**输出：** `Final balance = 1900`——1 × 100 + 9 × 200 = 100 + 1800 = 1900。✔（用 gcc compile 跑过）

| Function | 作用 |
|---|---|
| `omp_lock_t myLock;` | 宣告 lock 变量 |
| `omp_init_lock(&myLock)` | 初始化（未锁）——用之前一定要做 |
| `omp_set_lock(&myLock)` | **取得**：没人用就拿走继续；有人拿着就**等**到它被释放 |
| `omp_unset_lock(&myLock)` | **释放**：让一个在等的 thread 进去 |
| `omp_destroy_lock(&myLock)` | 最后释放 lock 占用的资源 |

**为什么需要它：** 没有 lock 的话，两个 thread 可能同时读 `balance`、加、写回 → 存款不见（race）。有 lock，一次只有一个 thread 执行 `balance += amount`。

> 中文记法：**init → set → CS → unset → destroy**（开锁头 → 上锁 → 做事 → 开锁 → 丢掉锁头）。

**✅ 满分答法 — "Explain how `omp_lock_t` achieves mutual exclusion in this program." (6 marks)**
*`omp_lock_t myLock` declares an OpenMP lock (a mutex) and `omp_init_lock` initialises it to the unlocked state before the parallel region. Inside the region each of the 10 threads calls `omp_set_lock` before updating the shared variable `balance`: the first thread to arrive acquires the lock and enters the critical section, while any other thread calling `omp_set_lock` must wait until the lock is released. After `balance += amount`, the thread calls `omp_unset_lock` to release it so the next waiting thread can enter. Therefore only one thread updates balance at a time, preventing a race condition, and the final balance is always 100 + 9 × 200 = 1900. Finally `omp_destroy_lock` releases the lock's resources.*

### Spinlock（自旋锁）（slides 62–63）
一个共享的 `lock` 变量，初始为 0。要进去：检查 lock；是 0 就设成 1 然后进去；是 1 就**一直等（空转，spinning）**到变 0。出来：设 lock = 0。
```c
while (1) {                     // ENTRY SECTION
    if (lock == 0) {            // lock 空着吗？
        lock = 1;               // 拿锁
        break;                  // 进 critical section
    }                           // 否则继续空转（busy wait）
}
/* CRITICAL SECTION */
lock = 0;                       // EXIT SECTION – 释放
```
**缺点：** threads 要等（空转时还在烧 CPU）；如果拿着锁的 thread **死掉、卡住、被 block 或无限循环**，其他人可能**永远等下去**。
⚠️ Slide 63 这段代码本身**不安全**：在 `if (lock == 0)` 和 `lock = 1` 之间，另一个 thread 也可能看到 0 → 两个都进去。「检查」和「设定」必须是**一个不可分割的动作**——这就是下一张 slide「atomic lock」的意思。

### Atomic lock（slide 64）
1. Thread A 到 critical region，看到 lock = 0，**一次过（atomically）把它设成 1**，然后进去。
2. 就算 B 紧跟在 A 后面，B 看到 lock 已经是 1 → 等。
3. A 出来，把 lock 设回 0。
4. B 看到 0，一次过设成 1，然后进去。

### critical/atomic vs `omp_lock_t`（slide 79）

| `#pragma omp critical` / `atomic` | OpenMP mutex (`omp_lock_t`) |
|---|---|
| Automatically manages mutual exclusion | Requires explicit lock and unlock |
| No lock variable required | Requires a lock variable |
| Simple to implement | Provides more control over lock acquisition |
| `#pragma omp critical { counter++; }` or `#pragma omp atomic counter++;` | `omp_set_lock(&mutex); counter++; omp_unset_lock(&mutex);` |
| Lock released automatically at end of region | Programmer must explicitly release the lock |

Slide 78：两个 threads，`omp_set_lock(&mutex); counter++; printf(...); omp_unset_lock(&mutex);` → Thread 0: Counter = 1，Thread 1: Counter = 2，Final counter = 2。
Slide 76 备注：`omp_lock_t` **可能会 block**；`pthread_mutex` 通常会 block，由 OS 做 context switch。

---

## 11. Semaphores（信号量）（slides 65–77）

### 定义（slide 65）
**Semaphore S** — 一个**整数**变量，只能透过两个**不可分割（atomic）**的操作来存取：
- `wait()` — 原名 **P()**
- `signal()` — 原名 **V()**
```c
wait(S) {                    signal(S) {
    while (S <= 0)               S++;
        ;   // 什么都不做     }
    S--;
}
```

### 用 semaphore 做 mutual exclusion（slides 66–68）
```c
Semaphore S;    // 初始为 1
wait(S);
    critical section
signal(S);
```
银行例子，`S = 1`：

| Step | 动作 | S |
|---|---|---|
| 1 | A：wait(S) → S > 0，所以 S-- → 进去做 `Balance -= 200` | 0 |
| 2 | B：wait(S) → S ≤ 0 → 一直循环（等） | 0 |
| 3 | A：signal(S) → S++ | 1 |
| 4 | B：wait(S) 成功 → S-- → 进去做 `Balance -= 100` | 0 |
| 5 | B：signal(S) | 1 |

### Busy waiting 的问题（slide 69）
`wait()` 和 `signal()` 本身也不能对同一个 semaphore 同时执行 → 它们的实现又是一个 critical-section problem。实现时可能会 **busy-wait**（忙等），但代码很短，**如果 critical section 很少被占用**，忙等的时间就不多。可是应用程序可能在 critical section 里花很多时间，所以一般来说 busy waiting **不是好方法**。

### 用 semaphore 做 bounded buffer（slides 70–73）
- N 个 buffer 格子，每格放一个 item。
- `mutex` = **1**（buffer 的互斥）
- `full` = **0**（满格的数量）
- `empty` = **N**（空格的数量）

```c
// Producer                             // Consumer
do {                                    do {
    // 生产一个 item 放进 nextp             wait(full);     // 等有 item
    wait(empty);    // 等有空位             wait(mutex);
    wait(mutex);                            // 从 buffer 拿 item 到 nextc
    // 把 nextp 放进 buffer                 signal(mutex);
    signal(mutex);                          signal(empty);  // 多一个空格
    signal(full);   // 多一个 item          // 消费 nextc 里的 item
} while (true);                         } while (true);
```
**Slide 73 trace（buffer 大小 n）：** producer 加入 A、B、C → empty：n → n−3，full：0 → 3。Consumer 拿走 A 再拿 B → full 3 → 1，empty n−3 → n−1，显示 "A B"。每次碰 buffer，`mutex` 都是 1 → 0 → 1。
**看点：** `wait(empty)`/`wait(full)` 要放在 `wait(mutex)` **前面**。如果 producer 先拿了 mutex，再因为 buffer 满而卡住，consumer 就永远拿不到 mutex 去腾出空位 → deadlock。

### 没有 busy waiting 的 semaphore（slides 74–77）
每个 semaphore 有一个 **value**（整数）和一个 **waiting queue**（等待中的 process 列表）。
- **block()** – 把调用它的 process 放进 waiting queue（去睡觉，不空转）。
- **wakeup(P)** – 从 queue 拿出一个 process P，放进 **ready queue**。
```c
wait(S) {                              signal(S) {
    S.value--;                             S.value++;
    if (S.value < 0) {                     if (S.value <= 0) {
        add this process to S.list;            remove a process P from S.list;
        block();                               wakeup(P);
    }                                      }
}                                      }
```
**Value 是负数 = 有几个 process 在等。**

**Trace（slide 77），mutex = 1，process A、B、C：**

| Step | 动作 | value | Waiting queue |
|---|---|---|---|
| 1 | A：wait → 0，不 < 0 → A 进 CS | 0 | – |
| 2 | B：wait → −1 < 0 → B **BLOCKED** | −1 | B |
| 3 | C：wait → −2 < 0 → C **BLOCKED** | −2 | B, C |
| 4 | A：signal → −1 ≤ 0 → wakeup(B) → B 进 CS | −1 | C |
| 5 | B：signal → 0 ≤ 0 → wakeup(C) → C 进 CS | 0 | – |
| 6 | C：signal → 1（> 0，没人要叫醒） | 1 | – |

---

## 12. Deadlock、starvation、用错 semaphore（slides 80–83）
**Deadlock（死锁）：** 两个或以上的 process **无限期地等**一个只有其中某个等待中的 process 才能触发的事件。
```
S = 1, Q = 1
     P0              P1
   wait(S);        wait(Q);
   wait(Q);        wait(S);
     ...             ...
   signal(S);      signal(Q);
   signal(Q);      signal(S);
```

| Step | 动作 | S | Q |
|---|---|---|---|
| 1 | P0：wait(S) | 0 | 1 |
| 2 | P1：wait(Q) | 0 | 0 |
| 3 | P0：wait(Q) → −1 → P0 被 block（要等 P1 的 signal(Q)） | 0 | −1 |
| 4 | P1：wait(S) → −1 → P1 被 block（要等 P0 的 signal(S)） | −1 | −1 |

两个 signal 都永远执行不到 → **deadlock**（slide 82）。

**Starvation（饥饿）** *（slide 80 标题有，但没定义——额外补充）：* 无限期阻塞——一个 process 一直在 semaphore queue 里等，因为别人一直被选中（例如 queue 是 LIFO，后进先出）。

**用错 semaphore 的问题（slide 83）：**

| 错误 | 结果 |
|---|---|
| `signal(mutex) … wait(mutex)`（顺序反了） | 几个 process 可能同时在 CS 里（没有互斥） |
| `wait(mutex) … wait(mutex)` | **Deadlock**（自己把自己卡住） |
| 漏写 `wait(mutex)` 和/或 `signal(mutex)` | 几个人同时在 CS / 其他人永远被挡住 |

---

## 回到开场
共用银行户口：RM1,000 − 200 − 100 结果变成 RM800 或 RM900，是因为两笔提款在 `Balance` 上 race。把每次更新包进 critical section——`#pragma omp critical`、`omp_set_lock/omp_unset_lock`，或 S = 1 的 `wait(S)/signal(S)`——结果就永远是 RM700。晚上的利息计算：画出依赖图，把 task 排到 3 个 processor 上，**critical path** 告诉你最快能多快；speedup 和 efficiency 告诉你多加的 processor 值不值得。

---

## ⚠️ Slides 有问题的地方

| Slide | Slide 写的 | 正确理解 |
|---|---|---|
| 4 | COBEGIN 用 T3 = D+E、T5 = F−G；slide 5 改名为 T2、T3 | 同样的 task，名字不同——答题用 slide 5 的命名（T1–T5） |
| 11, 88 | E = 49.33 %、59.33 % | 是用四舍五入后的 S 算的；不四舍五入是 49.28 %、59.42 %。两个都要写算式 |
| 63 | Spinlock `if (lock==0) { lock = 1; }` | 不是 atomic——两个 thread 可能同时通过检查。需要 atomic test-and-set（slide 64） |
| 77 | "Multex List" | 打错字：mutex list |
| 80 | 标题 "Deadlock and Starvation" | Starvation 没有定义——见上面补充 |
| 7–10 | 四个 Gantt chart | 全部都对；Tp 都是 23 ms |

---

## 术语表

| English | 中文 | 一句话说明 |
|---|---|---|
| Task dependency graph | 任务依赖图 | 哪个 task 需要哪个结果 |
| Critical path | 关键路径 | 最长的依赖链 → Tp |
| Gantt chart | 甘特图 | 每个 processor 的 task 时间线 |
| Speedup / efficiency | 加速比 / 效率 | Ts/Tp；S/p |
| Race condition | 竞争条件 | 结果取决于执行顺序 |
| Critical section | 临界区 | 存取共享 data 的代码 |
| Entry / exit / remainder section | 进入区 / 退出区 / 剩余区 | CS 协议的各部分 |
| Mutual exclusion | 互斥 | 一次只有一个 process 在 CS 里 |
| Progress | 推进 | 决定下一个进去的人不能无限拖延 |
| Bounded waiting | 有限等待 | 一个 process 等的时间有上限 |
| Busy waiting | 忙等待 | 一边等一边空转（浪费 CPU） |
| Preemption | 抢占 | OS 把 CPU 拿走 |
| Spinlock | 自旋锁 | 用空转等待来拿的锁 |
| Semaphore | 信号量 | 有 atomic wait/signal 的整数 |
| Deadlock / starvation | 死锁 / 饥饿 | 互相永远等 / 永远轮不到 |

## 速记卡
- Ts = 所有 task 加总。Tp = critical path（开始 = 依赖的 task 最晚结束时间）。S = Ts/Tp。E = S/p × 100%。
- 例题 1：Ts 34、Tp 23（T2→T4→T5→A）、S 1.48、E 49.33%。Extra Q：Ts 41、Tp 23（T1→T4→T6→W）、S 1.78、E 59.33%。
- count++ = load、add、store → 交错执行会变 4 或 6，而不是 5。
- CS 三个条件：**mutual exclusion、progress、bounded waiting**。
- Peterson：`flag[i]=T; turn=j; while(flag[j] && turn==j);`——只适用于 2 个 process。
- Disable interrupts：单 CPU 可以；用户程序不行（霸占 CPU），multiprocessor 不行（慢，其他 CPU 照样能碰 memory）。
- omp_lock_t：init → set → CS → unset → destroy。Balance = 1900。
- Semaphore：wait（P）/ signal（V）；mutex=1、full=0、empty=N；wait(empty/full) 要在 wait(mutex) 前面。
- 不忙等版本：value--，< 0 就 block；value++，≤ 0 就 wakeup。负数 = 几个在等。
- Deadlock：P0 wait(S) wait(Q)；P1 wait(Q) wait(S)。

---

## 练习题（附答案）
*题目用英文，答案附华文说明。*

### A. MCQ
1. Ts = 30 ms, Tp = 12 ms on 4 processors. Efficiency? (a) 40 % (b) 62.5 % (c) 2.5 % (d) 250 %
2. Which requirement is violated if a process can wait forever while others keep entering? (a) mutual exclusion (b) progress (c) bounded waiting (d) atomicity
3. In the no-busy-wait semaphore, value = −3 means (a) 3 processes in CS (b) 3 processes waiting (c) 3 free slots (d) error
4. Which call releases an OpenMP lock? (a) omp_set_lock (b) omp_unset_lock (c) omp_destroy_lock (d) omp_init_lock
5. For bounded buffer, `empty` is initialised to (a) 0 (b) 1 (c) N (d) −1

**答案：** 1-b（S = 30/12 = 2.5；2.5/4 = 62.5 %）；2-c；3-b；4-b；5-c。

### B. Short answer
1. Explain why disabling interrupts is not suitable for multiprocessors.
   *Disabling interrupts on all processors requires messages to every CPU, which is time-consuming and delays entry to critical sections; and it only affects the CPU that executed it, so other CPUs can still access shared memory — mutual exclusion is not guaranteed.*
2. State the three requirements of a critical-section solution.
   *Mutual exclusion, progress, bounded waiting（定义见 §7）.*
3. Differentiate `#pragma omp critical` and `omp_lock_t`.
   *critical manages mutual exclusion automatically with no lock variable and releases at the end of the region; omp_lock_t needs a lock variable, explicit init/set/unset/destroy, giving more control, but the programmer must remember to release it.*
   *华文：critical 是自动的；lock 要自己上锁、开锁，比较灵活但容易忘了开锁。*

### C. Application
1. Z = (A + B) × C − D / (E − F). Times: × 6, + 3, − 2, / 5 ms; 3 processors. Find Ts, Tp, S, E.
   *Tasks: T1 = A+B (3), T2 = E−F (2), T3 = T1×C (6), T4 = D/T2 (5), Z = T3−T4 (2).*
   *Ts = 3+2+6+5+2 = **18 ms**.*
   *Gantt: P1 T1 0–3, T3 3–9; P2 T2 0–2, T4 2–7; Z starts max(9,7) = 9 → 9–11. **Tp = 11 ms** (critical path T1→T3→Z = 3+6+2).*
   *S = 18/11 ≈ **1.64**; E = 1.64/3 × 100 ≈ **54.5 %**.*
   *华文：P3 完全没用到——只用 2 个 processor，Tp 一样是 11，效率会变 81.8 %。*
2. Trace a semaphore S = 2 used by 4 processes P1–P4 that each call wait(S) in order (no-busy-wait version). Who enters, who blocks, final value?
   *P1: 1 → enters; P2: 0 → enters; P3: −1 → blocks; P4: −2 → blocks. Value −2, queue P3, P4. When P1 signals → −1, wake P3; P2 signals → 0, wake P4.*
3. With count = 3, the consumer's three steps run t0–t2 and the producer's run t3–t5 (no interleaving). Final count?
   *Consumer: 3 → 2; producer reads 2 → 3. Final 3 — correct, because there was no interleaving.*

### D. Thinking
1. Rewrite the deadlocking P0/P1 code so it cannot deadlock.
   *Make both processes acquire in the same order: P0 and P1 both do wait(S) then wait(Q) (and signal in reverse). Then the second process blocks on S before taking Q, so no circular wait.*
   *华文：大家都用同样的顺序拿锁，就不会互相卡住。*
2. In the producer, what happens if `wait(mutex)` is placed **before** `wait(empty)` and the buffer is full?
   *The producer holds the mutex and then blocks on empty; the consumer blocks on wait(mutex) and can never remove an item to signal empty → deadlock.*

---

## Slide 索引
| 章节 | Slides |
|---|---|
| 1 Concurrent processing | 3–4 |
| 2 Dependency graph、Ts、Tp、S、E | 5–11, 84–88 |
| 3 DO loop | 12–16 |
| 4 Producer–consumer race | 17–41 |
| 5 critical vs atomic | 42–44, 48–50 |
| 6 Spooler | 45–46 |
| 7 CS problem | 47, 51, 53 |
| 8 Peterson | 52–56 |
| 9 Disable interrupts | 57–59 |
| 10 Locks | 60–64, 78–79 |
| 11 Semaphores | 65–77 |
| 12 Deadlock | 80–83 |

## 跟其他章的连接
- Ch1 slides 37–42（OpenMP 更新丢失）就是 §4 讲的 race condition。
- Ch2 pipes：kernel 在空的时候 block 读方、满的时候 block 写方——这就是 OS 帮你做的 bounded buffer。
- Ch3：`critical`、`atomic`、`single` 指令和对比表（slide 31）。
