Go 的 **Goroutine 调度器（GPM Scheduler）** 是 Go 运行时最核心的设计之一。
GPM 分别代表：

```
G — Goroutine
P — Processor
M — Machine (OS Thread)
```

Go 通过 **G-M-P 三层模型**实现高并发和高性能调度。

---

# 一、GPM 调度模型整体结构

核心关系：

```
        +----------------+
        |   Goroutine G  |
        +----------------+
               |
               v
        +----------------+
        | Processor (P)  |  ← 本地队列
        +----------------+
               |
               v
        +----------------+
        | Machine (M)    |  ← OS Thread
        +----------------+
               |
               v
             CPU
```

关键原则：

```
M 必须绑定 P 才能执行 G
P 负责调度 G
```

关系总结：

| 组件 | 含义        | 数量         |
| -- | --------- | ---------- |
| G  | goroutine | 数百万        |
| P  | 逻辑处理器     | GOMAXPROCS |
| M  | OS 线程     | 动态         |

---

# 二、G (Goroutine)

G 就是 Go 的协程对象。

内部结构大致：

```go
type g struct {
    stack       stack
    sched       gobuf
    goid        int64
    status      uint32
}
```

关键字段：

| 字段     | 作用           |
| ------ | ------------ |
| stack  | goroutine 栈  |
| sched  | 调度信息         |
| goid   | goroutine ID |
| status | 状态           |

G 的状态：

```
Grunnable   可运行
Grunning    运行中
Gwaiting    等待
Gsyscall    系统调用
Gdead       已结束
```

---

# 三、M (Machine)

M 是 **真实的 OS 线程**。

结构：

```go
type m struct {
    g0      *g
    curg    *g
    p       *p
}
```

说明：

| 字段   | 作用            |
| ---- | ------------- |
| g0   | 调度栈           |
| curg | 当前执行的 G       |
| p    | 绑定的 Processor |

关键规则：

```
M 必须绑定 P 才能执行 G
```

如果没有 P：

```
M 会阻塞
```

---

# 四、P (Processor)

P 是 **调度器核心**。

数量：

```
P = GOMAXPROCS
```

默认：

```
runtime.NumCPU()
```

结构：

```go
type p struct {
    runq       [256]guintptr
    runnext    guintptr
    m          muintptr
}
```

关键组件：

| 字段      | 作用   |
| ------- | ---- |
| runq    | 本地队列 |
| runnext | 优先执行 |
| m       | 当前 M |

---

# 五、调度队列

Go 有 **两种队列**：

```
1 全局队列
2 本地队列
```

结构：

```
             Global Run Queue
                    |
         -------------------------
         |          |            |
        P1         P2           P3
     LocalQ     LocalQ       LocalQ
```

优先级：

```
Local Queue > Global Queue
```

原因：

```
减少锁竞争
```

---

# 六、Goroutine 创建流程

当执行：

```go
go func() {}
```

运行时流程：

```
go statement
      ↓
new G
      ↓
加入 P 的 local queue
      ↓
M 调度执行
```

流程图：

```
go func()
   ↓
创建 G
   ↓
放入 P.runq
   ↓
M 执行
```

---

# 七、G 调度执行流程

调度循环：

```
schedule()
   ↓
从 P.runq 取 G
   ↓
execute(G)
   ↓
G 执行完成
```

伪代码：

```go
for {
    g := runqget(p)

    if g == nil {
        g = globrunqget()
    }

    execute(g)
}
```

---

# 八、Work Stealing（工作窃取）

如果某个 P 的队列空了：

```
P1 runq 空
```

会去偷别的 P 的任务。

示例：

```
P1 localQ []
P2 localQ [G1 G2 G3 G4]

P1 steal -> G3 G4
```

规则：

```
一次偷一半
```

目的：

```
负载均衡
```

---

# 九、Syscall 调度

如果 goroutine 进入系统调用：

```
read
write
network
```

流程：

```
G -> syscall
      ↓
M 被阻塞
      ↓
P 解绑 M
      ↓
P 找新 M
```

流程图：

```
G syscall
   ↓
M blocked
   ↓
P detach M
   ↓
P attach new M
```

作用：

```
防止线程阻塞导致调度停滞
```

---

# 十、网络 IO 调度（Netpoller）

Go 使用 **epoll / kqueue / IOCP** 实现 IO 复用。

结构：

```
        epoll
          |
      Netpoller
          |
    ready goroutine
          |
      放回 run queue
```

流程：

```
goroutine read socket
      ↓
注册 epoll
      ↓
goroutine park
      ↓
IO ready
      ↓
goroutine runnable
```

优势：

```
少线程处理大量 IO
```

---

# 十一、抢占式调度（Preemption）

Go 1.14 之后支持 **异步抢占**。

旧版本：

```
协作式调度
```

只有：

```
函数调用
channel
syscall
```

才能切换。

问题：

```
死循环
```

例如：

```go
for {}
```

会阻塞调度。

---

新版本：

```
signal + stack check
```

流程：

```
sysmon thread
     ↓
发送 SIGURG
     ↓
抢占 goroutine
```

作用：

```
避免长时间占用 CPU
```

---

# 十二、sysmon 线程

Go 运行时有一个 **系统监控线程**：

```
sysmon
```

作用：

```
1 GC 触发
2 网络轮询
3 抢占调度
4 deadlock 检测
```

---

# 十三、GPM 调度完整流程

完整流程：

```
创建 goroutine
      ↓
放入 P.local queue
      ↓
M 绑定 P
      ↓
M 从 P.runq 取 G
      ↓
执行 G
      ↓
G 结束或阻塞
      ↓
调度下一个 G
```

系统结构：

```
                Global Queue
                      |
      -------------------------------------
      |                |                 |
     P1               P2                P3
   runq             runq              runq
      |                |                 |
      M1               M2               M3
      |                |                 |
      CPU              CPU               CPU
```

---

# 十四、为什么 GPM 比传统线程模型快

传统线程：

```
1 request = 1 thread
```

问题：

```
线程切换成本高
```

Go：

```
1 thread = N goroutine
```

优势：

```
栈小（2KB）
调度快
创建便宜
```

---

# 十五、经典面试题

### 1 GPM 为什么需要 P？

P 的作用：

```
调度上下文
本地队列
减少锁竞争
```

如果没有 P：

```
M 需要全局锁
性能极差
```

---

### 2 P 的数量是多少？

```
GOMAXPROCS
```

默认：

```
CPU 核心数
```

---

### 3 Goroutine 栈为什么很小？

初始：

```
2KB
```

自动增长：

```
stack grow
```

---

### 4 Goroutine 为什么切换快？

原因：

```
用户态调度
不用 syscall
```

---

### 5 Go 如何避免线程爆炸？

通过：

```
G -> M multiplexing
```

一个线程执行多个 goroutine。

---

# 十六、一张最经典的 GPM 调度图

```
                Global Run Queue
                        |
             -------------------------
             |           |           |
            P1          P2          P3
         LocalQ      LocalQ      LocalQ
             |           |           |
            M1          M2          M3
             |           |           |
            CPU         CPU         CPU
```