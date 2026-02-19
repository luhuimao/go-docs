`context` 是 Go 并发与微服务架构里的**控制面（control plane）机制**，核心解决三件事：

1. 取消（Cancellation）
2. 超时（Timeout / Deadline）
3. 跨调用链传递请求级数据（Request-scoped data）

---

# 一、官方定义

`context` 来自标准库：

* Go 标准库 `context`

设计目标：

> 在 goroutine 树之间安全地传播取消信号、截止时间和元数据。

---

# 二、为什么必须使用 context？

在高并发系统中，如果没有 context：

* HTTP 客户端断开，后端 goroutine 仍在跑
* 数据库查询超时却无法中断
* 微服务调用链无法整体取消
* 无法传播 trace_id

这在交易系统 / 微服务架构中是严重问题。

---

# 三、context 的四种创建方式

## 1️⃣ context.Background()

根节点（main 函数 / 初始化用）

```go
ctx := context.Background()
```

---

## 2️⃣ context.WithCancel()

可手动取消

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    time.Sleep(time.Second)
    cancel()
}()
```

适用于：

* 控制 goroutine 生命周期

---

## 3️⃣ context.WithTimeout()

自动超时

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
```

用于：

* RPC
* DB 查询
* 外部 API 调用

---

## 4️⃣ context.WithValue()

传递请求级数据

```go
ctx := context.WithValue(context.Background(), "trace_id", "abc123")
```

⚠ 只用于 request-scoped data
❌ 不用于传递业务参数

---

# 四、标准使用模式（企业级写法）

## 1️⃣ HTTP → Service → Repo

```go
func (h *Handler) Create(c *gin.Context) {
    ctx := c.Request.Context()

    err := h.service.Create(ctx)
}
```

```go
func (s *Service) Create(ctx context.Context) error {
    return s.repo.Save(ctx)
}
```

```go
func (r *Repo) Save(ctx context.Context) error {
    return r.db.ExecContext(ctx, "INSERT ...")
}
```

关键点：

> context 必须一层层往下传

---

# 五、取消传播原理

context 是树状结构：

```
Background
   ↓
WithTimeout
   ↓
WithCancel
   ↓
goroutine
```

父 ctx cancel → 子 ctx 全部 cancel

这是 Go 并发控制的核心机制。

---

# 六、在微服务中的典型使用

## 1️⃣ Gin + gRPC

Gin handler：

```go
ctx := c.Request.Context()
resp, err := grpcClient.CreateOrder(ctx, req)
```

gRPC 内部自动监听 ctx.Done()

---

## 2️⃣ MySQL

```go
db.QueryContext(ctx, "SELECT ...")
```

ctx cancel → SQL 终止

---

## 3️⃣ Redis

```go
rdb.Get(ctx, "key")
```

---

# 七、context 使用原则（高级工程师必须遵守）

### ✔ 必须作为第一个参数

```go
func DoSomething(ctx context.Context, ...)
```

---

### ✔ 不要存到 struct 里

错误：

```go
type Service struct {
    ctx context.Context // ❌
}
```

---

### ✔ 不要传 nil

永远使用：

```go
context.Background()
```

---

### ✔ WithValue 使用强类型 key

避免：

```go
ctx := context.WithValue(ctx, "user_id", 123)
```

推荐：

```go
type ctxKey string
const userIDKey ctxKey = "user_id"
```

---

# 八、底层原理简析

context 是接口：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

核心机制：

* `Done()` 返回 channel
* cancel 时 close channel
* goroutine select 监听

典型模式：

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-work:
    return result
}
```

---

# 九、交易系统中的最佳实践

如果你做撮合引擎：

### 1️⃣ HTTP 超时 2s

### 2️⃣ gRPC 1.5s

### 3️⃣ DB 1s

通过 context 层层收缩 timeout。

---

# 十、常见错误

| 错误           | 后果           |
| ------------ | ------------ |
| 不传 ctx       | goroutine 泄漏 |
| 忘记 cancel    | 资源泄漏         |
| WithValue 乱用 | 隐性耦合         |
| 超时太大         | 雪崩风险         |

---

# 十一、一句话总结

> context 是 Go 的“请求生命周期控制器”。

它不是：

* 全局变量
* 业务参数容器
* 日志存储

它是：

* 取消信号传播器
* 超时控制器
* 调用链上下文载体


下面给你一张**微服务调用链中 context 传播的完整结构图**（贴近真实生产环境）。

场景：

* API Gateway（Gin）
* Order Service（gRPC）
* Account Service（gRPC）
* MySQL
* Redis

---

# 一、整体传播拓扑图

```
客户端请求
    │
    ▼
┌───────────────────────────┐
│      API Gateway          │
│      (Gin Handler)        │
│                           │
│  ctx := c.Request.Context() 
└───────────────┬───────────┘
                │
                │ gRPC 调用（携带 ctx）
                ▼
┌───────────────────────────┐
│       Order Service       │
│       (gRPC Server)       │
│                           │
│  func(ctx context.Context)
│          │
│          ├── DB.QueryContext(ctx)
│          │
│          ├── Redis.Get(ctx)
│          │
│          └── 调用 Account Service
│                  │
│                  ▼
│        gRPC Client (ctx)
└───────────────────────────┘
                │
                ▼
┌───────────────────────────┐
│      Account Service      │
│       (gRPC Server)       │
│                           │
│  func(ctx context.Context)
│          │
│          └── DB.ExecContext(ctx)
└───────────────────────────┘
```

---

# 二、取消信号传播路径

假设：

* 客户端 2 秒超时
* Gateway 设置 2s timeout
* Order Service 内部 DB 查询 3s

传播过程：

```
Client timeout
      ↓
Gin Request Context cancel
      ↓
gRPC client 收到 ctx.Done()
      ↓
Order Service ctx cancel
      ↓
DB.QueryContext 终止
      ↓
Account Service 调用终止
```

所有 goroutine 同步终止。

这就是 context 的树状传播。

---

# 三、调用链时序图

```
Client
  │
  │ HTTP (2s timeout)
  ▼
Gateway (Gin)
  │ ctx.WithTimeout(2s)
  │
  │──────────── gRPC (携带 ctx) ───────────▶
  ▼                                         Order Service
                                            │
                                            │ DB.QueryContext(ctx)
                                            │
                                            │──── gRPC (ctx) ───▶
                                            ▼                    Account Service
                                                                │
                                                                │ DB.ExecContext(ctx)
```

---

# 四、超时收缩模型（生产级推荐）

不要所有层使用相同超时。

建议分层收缩：

```
HTTP 层      2.0s
gRPC 层      1.8s
DB 查询      1.5s
Redis 查询   500ms
```

示例：

```go
// Gateway
ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
defer cancel()

// Order Service 内部
ctx, cancel := context.WithTimeout(ctx, 1500*time.Millisecond)
defer cancel()
```

避免尾部超时雪崩。

---

# 五、调用链中 Value 传播

context 还可以传递：

* trace_id
* user_id
* request_id
* auth claims

传播图：

```
Client
   │
   │  X-Trace-ID: abc123
   ▼
Gateway
   │ ctx = WithValue(trace_id)
   ▼
Order Service
   │ ctx.Value(trace_id)
   ▼
Account Service
```

这就是分布式链路追踪的基础。

例如：

* OpenTelemetry
* Jaeger

都是基于 context 传播 trace。

---

# 六、Goroutine 传播模型

典型并发场景：

```go
func process(ctx context.Context) {
    go worker(ctx)
}

func worker(ctx context.Context) {
    select {
    case <-ctx.Done():
        return
    }
}
```

传播图：

```
Parent ctx cancel
        ↓
worker ctx.Done()
        ↓
goroutine 退出
```

这能防止 goroutine 泄漏。

---

# 七、完整微服务 context 树结构

```
Background
   ↓
HTTP Context (2s)
   ↓
gRPC Context
   ↓
DB Context
   ↓
Sub-goroutines
```

父 cancel → 全部子节点 cancel。

---

# 八、架构级理解

在微服务系统中：

* context 是“控制信号通道”
* 它与业务数据完全解耦
* 它控制调用链生命周期
* 它防止资源泄漏
* 它实现分布式追踪

---

# 九、如果是交易系统

在撮合系统中：

* 下单接口超时 → 必须终止后续持久化
* 风控 RPC 超时 → 必须终止撮合前流程
* MQ 写入超时 → 必须 fail fast

所有这些都靠 context 控制。

---

# 十、一句话总结

> context 是微服务调用链的“生命线”。

它让：

* 超时可控
* 取消可传播
* 资源可回收
* 链路可追踪
