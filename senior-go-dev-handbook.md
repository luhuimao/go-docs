
# 一、Go 语言基础 & 运行时（Runtime）

## 1. Go 的内存分配机制是怎样的？

**答：**

Go 的内存分配主要由 **runtime** 管理，核心组件包括：

* **mcache**：每个 P 私有的小对象缓存（无锁）
* **mcentral**：全局中心缓存（有锁）
* **mheap**：真正向 OS 申请内存（`mmap`）

### 分配流程（小对象 < 32KB）：

1. 优先从当前 P 的 `mcache` 分配
2. 不够则向 `mcentral` 申请 span
3. 再不够才向 `mheap` 申请

### 大对象（≥ 32KB）：

* 直接从 `mheap` 分配

### 优点：

* **减少锁竞争**
* **局部性好**
* **高并发性能强**

**追问点：**

* 为什么 Go 不适合频繁创建大对象？
* sync.Pool 的使用场景？

---

## 2. Go 的 GC 原理是什么？（三色标记）

**答：**

Go 使用的是 **并发三色标记清除（Concurrent Mark & Sweep）**。

### 三色对象：

* **白色**：未访问对象（可能被回收）
* **灰色**：已访问，但子对象未扫描
* **黑色**：自身和子对象都已扫描

### 主要阶段：

1. **STW（极短）**：root 标记
2. **并发标记**：灰 → 黑
3. **STW（极短）**：标记结束
4. **并发清除**

### 写屏障（Write Barrier）：

* 保证黑对象不会指向白对象
* 维持 GC 正确性

### GC 触发条件：

* 内存分配达到 `GOGC`（默认 100%）

**高级点：**

* Go 1.18+ 使用 **混合写屏障**
* STW 时间通常 < 1ms

---

## 3. 逃逸分析是什么？如何优化？

**答：**

逃逸分析决定变量是分配在 **栈** 还是 **堆**。

### 常见逃逸场景：

* 返回局部变量指针
* interface{} 接收具体类型
* 闭包引用外部变量
* slice / map 动态扩容

### 查看逃逸：

```bash
go build -gcflags="-m"
```

### 优化手段：

* 减少 interface{}
* 尽量传值而非指针
* 控制 slice 扩容
* 使用 sync.Pool 复用对象

---

# 二、并发模型 & 调度器（GMP）

## 4. Go 的 GMP 模型是什么？

**答：**

* **G**：goroutine
* **M**：OS 线程
* **P**：调度器（执行上下文）

### 核心思想：

* **P 绑定可运行队列**
* **M 需要 P 才能运行 G**
* P 的数量由 `GOMAXPROCS` 决定

### 调度流程：

1. G 放入 P 的本地队列
2. 本地无 G → 全局队列
3. 工作窃取（steal）

**优点：**

* 高并发
* 减少线程上下文切换

---

## 5. goroutine 泄漏如何产生？如何排查？

**答：**

### 常见原因：

* channel 无人接收 / 发送
* context 未 cancel
* select 无 default 且阻塞
* 死循环 + 阻塞 IO

### 排查方法：

* `runtime.NumGoroutine()`
* pprof：

```go
import _ "net/http/pprof"
```

### 解决方案：

* 所有 goroutine 都要有退出路径
* context 控制生命周期
* select + done channel

---

## 6. channel 的底层实现？

**答：**

channel 本质是 `hchan` 结构体：

* 环形队列（buffer）
* sendq / recvq 等待队列
* mutex 保护

### 关键点：

* 无缓冲 channel：同步通信
* 有缓冲 channel：异步 + 队列
* 关闭 channel：

  * 读：立即返回零值
  * 写：panic

---

# 三、标准库 & 常见坑

## 7. map 是线程安全的吗？为什么？

**答：**

不是。

* map 并发读写会 **panic**
* 原因：扩容 & rehash 非原子

### 解决方案：

* `sync.Mutex`
* `sync.RWMutex`
* `sync.Map`（读多写少）

---

## 8. slice 扩容机制？

**答：**

* 小于 1024：容量 * 2
* 大于 1024：增长 ~1.25x

### 易错点：

```go
func f(s []int) {
    s = append(s, 1)
}
```

不会影响原 slice（可能扩容）

---

## 9. defer 的执行顺序和性能问题？

**答：**

* **LIFO（栈）**
* 参数在 defer 声明时求值

### 性能：

* defer 有一定开销（已优化很多）
* 高频循环中谨慎使用

---

# 四、工程实践 & 架构

## 10. 如何设计一个高并发 HTTP 服务？

**答：**

### 核心点：

* 连接复用（HTTP KeepAlive）
* goroutine 池 / 限流
* context 控制请求生命周期
* 超时 & 熔断
* pprof + metrics

### 常用组件：

* `net/http`
* `errgroup`
* `rate limiter`
* `prometheus`

---

## 11. 如何做限流？

**答：**

* 令牌桶（推荐）
* 漏桶
* 固定窗口
* 滑动窗口

### Go 实现：

```go
golang.org/x/time/rate
```

---

## 12. 如何做优雅关闭（graceful shutdown）？

**答：**

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
defer stop()

go server.ListenAndServe()

<-ctx.Done()
server.Shutdown(context.Background())
```

---

# 五、性能优化 & 调试

## 13. Go 程序性能分析手段？

**答：**

* pprof（CPU / heap / goroutine）
* trace
* benchmark
* flame graph

### 常见优化方向：

* 减少分配
* 减少锁
* 减少 GC 压力
* 对象复用

---

## 14. interface{} 的成本在哪里？

**答：**

* 包含 type + data
* 可能导致逃逸
* 动态派发有开销

---

# 六、系统设计 & 场景题

## 15. 设计一个任务调度系统（Go 实现）

**答：**

* job queue（channel / MQ）
* worker pool
* retry + backoff
* 幂等
* 可观测性

---

## 16. Go 适合做什么？不适合做什么？

**适合：**

* 高并发服务
* 网络服务
* 微服务
* 基础设施

**不适合：**

* 强实时系统
* GUI
* 极端低延迟（纳秒级）