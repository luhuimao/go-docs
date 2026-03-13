# 一、goroutine 基础

## 一句话总结（面试先手）

> **当任务是 I/O 密集、可并发、相互独立，且需要高吞吐或低等待时，用 goroutine；
> 当任务是强 CPU 竞争、强顺序依赖、或需要严格控制资源时，要谨慎甚至不用。**

---

## 一、适合使用 goroutine 的场景 ✅

### 1️⃣ I/O 密集型任务（最典型）

**这是 goroutine 的主战场**

* HTTP / RPC 请求
* 数据库查询
* 读写文件
* 调用第三方 API

```go
go fetchUser()
go fetchOrder()
go fetchBalance()
```

**原因：**

* goroutine 切换成本极低
* I/O 阻塞时不占用 OS 线程
* 调度器自动复用 M

👉 **面试官最想听的点**

> goroutine 非常适合隐藏 I/O 延迟

---

### 2️⃣ 大量独立、无共享状态的任务

特征：

* 任务之间无依赖
* 不需要共享内存
* 失败互不影响

例如：

* 批量发送消息
* 批量校验数据
* 并发爬取

```go
for _, id := range ids {
    go process(id)
}
```

---

### 3️⃣ 提升响应速度（并行等待）

典型场景：**“并发请求，最快返回”**

```go
go queryCache()
go queryDB()
go queryRemote()

select {
case res := <-cache:
    return res
case res := <-db:
    return res
}
```

👉 常见于：

* 缓存 + DB
* 多数据源兜底

---

### 4️⃣ 后台异步任务（fire-and-forget）

* 打日志
* 上报埋点
* 发送通知
* 清理缓存

```go
go reportMetrics()
```

⚠️ **高级点：**

* 必须可丢失
* 不能影响主流程
* 不能无限创建

---

### 5️⃣ 并发流水线（Pipeline）

```go
go read()
go parse()
go store()
```

每一阶段一个 goroutine，用 channel 串起来

👉 非常 Go 风格，面试加分

---

## 二、不适合 / 需要谨慎使用 goroutine 的场景 ❌

### 1️⃣ CPU 密集型计算（无控制）

早期 Go 只在函数调用时检查栈是否需要扩展，同时判断是否需要抢占：

```go
// 编译器在每个函数开头插入栈检查
if sp < stackguard0 {
    morestack()  // 若 preempt 标记为 true → 触发调度
}
```

**问题**：无函数调用的紧密循环（如 `for { ; }`）永远不会被抢占，阻止 GC STW。

### 4.2 异步抢占（Go 1.14+，信号驱动）

Go 1.14 引入基于 **SIGURG 信号**的异步抢占：

```
sysmon 线程
    │ 发现 G 运行超过 10ms
    │
    ↓
向 M 的 OS 线程发送 SIGURG 信号
    │
    ↓
sighandler（在 M 上执行）
    │ 检查当前是否在安全点
    │
    ├─ 是安全点 → 修改 G 的 PC，插入 asyncPreempt 调用
    │
    └─ asyncPreempt()
            │ 保存所有寄存器到 g.asyncPreemptBuf
            │
            └─ gopreempt_m() → G 变为 Gpreempted → 重新入队 → schedule()
```

**安全点（Safe Point）**：

- 非内联函数调用点
- 不在写屏障（Write Barrier）执行期间
- 不在 nosplit 函数中
- 不在汇编代码中（需手动 `runtime.Gosched()`）

### 4.3 抢占标记

```go
// sysmon 设置抢占
gp.preempt = true
gp.stackguard0 = stackPreempt  // 触发下次函数调用时的协作抢占

// 协作抢占检查点（编译器生成）
TEXT runtime·morestack(SB)...
    MOVQ    (TLS), R14       // 加载当前 g
    CMPQ    SP, stackguard0  // 检查是否需要抢占
    JBE     morestack        // 栈溢出 or 抢占
```

---

## 五、系统调用处理

### 5.1 阻塞系统调用

```
G 发起阻塞 syscall（如 read、write）
    │
    ↓
entersyscall()
    ├─ 保存 G 的状态：Grunning → Gsyscall
    ├─ P 解绑 M（M.p = nil），P 进入 Psyscall 状态
    └─ M 独立进行系统调用
    
    {系统调用执行中...}
    
    ↓ sysmon 检测到 M 在 syscall 超过 20μs
    ├─ retake(p)：将 P 从 M 手中夺走
    └─ P 绑定到其他空闲 M，继续运行其他 G
    
    ↓ syscall 完成
exitsyscall()
    ├─ 尝试绑定原来的 P（通常已被占用）
    ├─ 尝试绑定其他空闲 P
    └─ 若无空闲 P：G 进入全局队列，M 进入 stopm() 休眠
```

### 5.2 非阻塞系统调用（Linux io_uring / epoll）

```
G 发起网络 I/O
    │
    ↓
runtime 将 fd 注册到 netpoller（epoll/kqueue/iocp）
G 进入 Gwaiting
M 和 P 继续运行其他 G
    
    {I/O 就绪...}
    
    ↓
netpoll() 返回就绪的 G
G 变为 Grunnable，重新入队
```

---

## 六、sysmon 系统监控线程

`sysmon` 是一个不绑定 P 的特殊 M（**不需要 P 即可运行**），每隔 10μs~10ms 运行一次：

```go
func sysmon() {
    for {
        // 1. 释放长时间空闲的 P（避免空转）

        // 2. epoll/kqueue 轮询，唤醒网络 I/O 就绪的 G
        lastpoll := sched.lastpoll.Load()
        if netpollinited() && ... {
            list, delta := netpoll(0)
            injectglist(&list)
        }

        // 3. 抢占运行超过 10ms 的 G
        retake(now)

        // 4. 触发 GC（如果 GC 被长时间推迟）
        forcegchelper()
    }
}
```

---

## 七、Goroutine 栈管理

### 7.1 分段栈 vs 连续栈

| 版本 | 策略 | 缺点 |
|------|------|------|
| Go 1.3 之前 | 分段栈（Segmented Stack）| 热分裂（Hot Split）性能问题 |
| Go 1.4+ | 连续栈（Contiguous Stack）| 栈增长需拷贝整个栈 |

### 7.2 栈增长过程

```
G 的栈不足
    │
    ↓
编译器插入的栈检查触发 morestack()
    │
    ↓
在 g0 上执行 runtime.newstack()
    ├─ 分配 2 倍大的新栈
    ├─ 将旧栈内容拷贝到新栈
    ├─ 调整所有指向旧栈的指针（含 defer、channel 等）
    └─ 释放旧栈，返回新栈继续执行
```

**初始栈大小**：2KB（`_StackMin = 2048`），每次翻倍增长，上限 1GB（64位系统）。

### 7.3 栈收缩（Stack Shrinking）

在 GC 扫描期间，若 G 的栈实际使用不足 1/4，会将栈缩小到一半：

```go
// GC STW 期间
shrinkstack(gp)
    ├─ 若 used < old_size/4
    └─ 迁移到 old_size/2 的新栈
```

---

## 八、channel 与调度的交互

```go
// channel send（缓冲区满，阻塞）
func chansend(c *hchan, ep unsafe.Pointer, ...) bool {
    lock(&c.lock)

    // 有等待的接收者？直接"握手"传值 → 唤醒接收者 G
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, ...)    // goready(sg.g) → Grunnable
        return true
    }

    // 缓冲区未满？写入缓冲区
    if c.qcount < c.dataqsiz {
        ...
        return true
    }

    // 阻塞：将当前 G 加入 sendq，挂起
    gp := getg()
    mysg.g = gp
    c.sendq.enqueue(mysg)
    gopark(chanparkcommit, ...)  // G → Gwaiting，让出 M
    ...
}
```

`gopark` 使 G 进入 `Gwaiting`，M 立即重新调度（`schedule()`）；当接收者就绪时 `goready` 将 G 重新置为 `Grunnable`。

---

## 九、调度器关键参数与调优

```go
sem := make(chan struct{}, 10)

sem <- struct{}{}
go func() {
    defer func() { <-sem }()
    work()
}()
```

---

### 3️⃣ goroutine ≠ 并发安全

* 共享数据一定要：

  * mutex
  * channel
  * 原子操作

---

## 五、面试官追问 & 高分回答

### Q：goroutine 一定比线程快吗？

**答：**

> 不一定。goroutine 创建和切换成本更低，但如果是 CPU 密集或大量竞争，goroutine 反而可能更慢。

---

### Q：什么时候宁可不用 goroutine？

**答：**

> 当逻辑强顺序、资源敏感、或无法保证 goroutine 生命周期时，我会选择同步执行，换取可控性。

---

## 十、调度器关键参数与调优

```go
// 设置 GOMAXPROCS（运行时可动态调整）
runtime.GOMAXPROCS(n)

// 主动让出当前 goroutine 的执行权
runtime.Gosched()

// 锁定当前 G 到当前 M（cgo / 线程本地存储场景）
runtime.LockOSThread()
runtime.UnlockOSThread()

// 查看当前活跃 goroutine 数
runtime.NumGoroutine()
```

**常见性能问题与排查**：

| 问题 | 症状 | 排查工具 |
|------|------|---------|
| G 饥饿 | 某些 G 长时间无法调度 | `GODEBUG=schedtrace=1000` |
| M 暴增 | 大量阻塞 syscall | `runtime.NumGoroutine()` + pprof |
| 栈热分裂（历史） | 特定递归函数极慢 | 已由连续栈解决 |
| GC STW 长 | 应用延迟抖动 | `GODEBUG=gctrace=1` |
| 调度延迟高 | 协程切换慢 | `go tool trace` |

```bash
# 输出调度器跟踪信息（每 1000ms）
GODEBUG=schedtrace=1000,scheddetail=1 ./your_program

# 生成调度器 trace 文件并分析
# 1. 在程序中埋点
import "runtime/trace"
trace.Start(f)
defer trace.Stop()

# 2. 分析 trace 文件
go tool trace trace.out
```

---

## 十一、goroutine 使用场景完整指南

### 11.1 适合使用 goroutine 的场景 ✅

| 场景 | 典型例子 | 说明 |
|------|---------|------|
| **I/O 密集型** | HTTP 请求、DB 查询、文件读写 | goroutine 最主战场，隐藏 I/O 延迟 |
| **独立并行任务** | 批量消息发送、并发数据校验 | 任务无依赖，失败互不影响 |
| **并行等待多源** | 缓存+DB 双查、多数据源兜底 | 取最快结果，降低响应时间 |
| **后台异步任务** | 日志上报、埋点、指标收集 | fire-and-forget，不影响主流程 |
| **流水线处理** | read → parse → store | channel 串联各阶段 |

**并发控制（信号量限流）**：

```go
// 用 buffered channel 做信号量，限制最大并发数
sem := make(chan struct{}, 10)

for _, item := range items {
    sem <- struct{}{}
    go func(item Item) {
        defer func() { <-sem }()
        process(item)
    }(item)
}
```

### 11.2 不适合 / 需谨慎使用的场景 ❌

| 场景 | 问题 | 替代方案 |
|------|------|---------|
| **CPU 密集型（无限制）** | 大量 goroutine 竞争 CPU，切换开销反而更大 | 限制并发数为 `GOMAXPROCS` |
| **强顺序依赖** | goroutine 执行顺序不可预期 | 串行执行或显式同步 |
| **无法保证生命周期** | goroutine 泄漏（leak） | 使用 `context.WithCancel` 管理 |
| **共享内存频繁读写** | data race，需要大量锁 | 考虑 channel 传递所有权 |

### 11.3 goroutine 并发安全提示

共享数据必须通过以下方式保护：
- `sync.Mutex` / `sync.RWMutex`
- `channel`（通过通信来共享内存）
- `sync/atomic` 原子操作

```go
// 使用 race detector 检测数据竞争
go run -race main.go
go test -race ./...
```

---

## 十二、总结：调度器设计哲学

```
1. N:M 线程模型
   N 个 Goroutine → M 个 OS 线程
   通过 P 解耦，实现高并发低开销

2. 工作窃取（Work Stealing）
   负载均衡，避免 CPU 空转

3. 无锁本地队列（Lock-free Local Queue）
   大部分调度操作无需全局锁，扩展性好

4. 网络 I/O 集成（Netpoller）
   I/O goroutine 不消耗 OS 线程，数万并发连接轻松支撑

5. 异步抢占（Async Preemption）
   保证 GC STW 及时触发，任何 goroutine 不会长期霸占 CPU

6. 动态栈（Dynamic Stack）
   goroutine 初始仅需 2KB，百万 goroutine 轻松创建
```

---

## 参考资料

| 资源 | 链接 |
|------|------|
| Go 运行时源码 | [github.com/golang/go/src/runtime](https://github.com/golang/go/tree/master/src/runtime) |
| Dmitry Vyukov 调度器设计文档 | [golang.org/s/go11sched](https://golang.org/s/go11sched) |
| Go GMP 可视化 | [《Scheduling In Go》by Ardan Labs](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html) |
| 异步抢占提案 | [github.com/golang/proposal/issues/24543](https://github.com/golang/proposal/issues/24543) |
| go tool trace 使用指南 | [pkg.go.dev/cmd/trace](https://pkg.go.dev/cmd/trace) |