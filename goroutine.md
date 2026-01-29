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

❌ 错误示例：

```go
for i := 0; i < 1e6; i++ {
    go heavyCompute()
}
```

**问题：**

* goroutine ≠ 并行
* 会被 `GOMAXPROCS` 限制
* 上下文切换反而更慢

✅ 正确做法：

* worker pool
* 控制并发数 = CPU 核数

---

### 2️⃣ 强顺序依赖的逻辑

例如：

* 金额计算
* 状态机
* 强一致写操作

```go
balance -= x
go writeDB(balance) // ❌
```

---

### 3️⃣ 无法确定生命周期的任务

⚠️ 极易导致 **goroutine 泄漏**

典型坑：

* channel 无接收者
* context 没 cancel
* select 永远阻塞

```go
go func() {
    <-ch // 永远没人写
}()
```

---

### 4️⃣ 高频、极短任务（过度并发）

```go
for i := 0; i < 1000; i++ {
    go x++
}
```

* goroutine 创建成本 > 执行成本
* 锁竞争严重

---

## 三、是否该用 goroutine 的判断清单（面试神表）

你可以直接背这一段：

> 我一般从 5 个维度判断是否使用 goroutine：
> 1）是否 I/O 密集
> 2）任务是否独立
> 3）是否能接受乱序或并发执行
> 4）是否有明确退出机制
> 5）是否对并发数量有上限控制

---

## 四、正确使用 goroutine 的“工程级姿势” ⭐

### 1️⃣ 必须配 context

```go
go func(ctx context.Context) {
    select {
    case <-ctx.Done():
        return
    case <-work:
    }
}(ctx)
```

---

### 2️⃣ 必须有限制（并发上限）

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