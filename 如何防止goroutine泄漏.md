一个**真实生产级别的 goroutine 泄漏示例 + 完整修复方案**。
场景模拟：**Gin API → 下游 RPC / 慢任务 → goroutine 未退出 → 内存持续上涨**。

我们基于 Go 标准库来演示。

---

# 一、典型 goroutine 泄漏场景

## ❌ 错误示例（生产中非常常见）

假设我们有一个接口，会启动一个异步任务：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    go slowWorker()
    w.Write([]byte("ok"))
}

func slowWorker() {
    time.Sleep(10 * time.Second)
    fmt.Println("done")
}
```

问题：

* 客户端 1 秒就断开
* 但 slowWorker 仍然跑满 10 秒
* QPS 高时 → goroutine 数暴涨
* 最终 OOM

如果这是：

* 下游 RPC
* 数据库调用
* MQ 阻塞
* channel 永远不返回

就会形成**永久泄漏**

---

# 二、另一种更隐蔽的泄漏（channel 阻塞）

```go
func worker() {
    ch := make(chan int)

    go func() {
        ch <- 1
    }()

    // 忘记接收
}
```

这个 goroutine 永远阻塞在发送上。

---

# 三、生产级泄漏场景（微服务真实模型）

场景：

* HTTP 请求超时 2s
* 后端 goroutine 调用 DB
* DB 卡住
* HTTP 断开
* goroutine 永远不退出

错误写法：

```go
func queryData() {
    ctx := context.Background() // ❌ 错误

    go func() {
        db.QueryContext(ctx, "SELECT ...")
    }()
}
```

问题：

* ctx 不可取消
* HTTP 断开无效
* goroutine 永远跑

---

# 四、正确方案：使用 context 控制 goroutine 生命周期

## ✅ 标准安全写法

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    resultCh := make(chan string, 1)

    go func(ctx context.Context) {
        select {
        case <-time.After(5 * time.Second):
            resultCh <- "done"
        case <-ctx.Done():
            return
        }
    }(ctx)

    select {
    case res := <-resultCh:
        w.Write([]byte(res))
    case <-ctx.Done():
        http.Error(w, "timeout", http.StatusGatewayTimeout)
    }
}
```

---

# 五、完整实战示例（模拟真实微服务）

### 场景

* API 接口
* 启动 3 个并发子任务
* 任一失败取消全部

---

## ✅ 完整安全版本

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func main() {
    http.HandleFunc("/test", handler)
    http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    err := process(ctx)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }

    w.Write([]byte("success"))
}

func process(ctx context.Context) error {
    workerCount := 3
    errCh := make(chan error, workerCount)

    for i := 0; i < workerCount; i++ {
        go worker(ctx, i, errCh)
    }

    for i := 0; i < workerCount; i++ {
        select {
        case err := <-errCh:
            if err != nil {
                return err
            }
        case <-ctx.Done():
            return ctx.Err()
        }
    }

    return nil
}

func worker(ctx context.Context, id int, errCh chan<- error) {
    select {
    case <-time.After(5 * time.Second):
        fmt.Println("worker done:", id)
        errCh <- nil
    case <-ctx.Done():
        fmt.Println("worker canceled:", id)
        return
    }
}
```

---

# 六、关键防泄漏模式总结

## 1️⃣ 所有 goroutine 都必须监听

```go
select {
case <-ctx.Done():
    return
}
```

---

## 2️⃣ channel 必须：

* 有缓冲
* 或保证一定被消费

---

## 3️⃣ 永远不要使用

```go
context.Background()
```

在请求链里。

---

## 4️⃣ DB / Redis / gRPC 必须使用 Context 版本 API

正确：

```go
db.QueryContext(ctx, ...)
rdb.Get(ctx, ...)
grpcClient.Call(ctx, ...)
```

---

# 七、生产事故复盘（真实案例模式）

症状：

* goroutine 数量从 2k → 200k
* CPU 飙升
* 内存上涨
* pprof 显示阻塞在 channel send

原因：

* 下游 RPC 超时
* 未监听 ctx.Done()
* goroutine 永远等待

解决：

* 所有 goroutine 加 ctx 监听
* 所有外部调用加 timeout
* 增加限流

---

# 八、企业级最佳实践工具

排查 goroutine 泄漏：

* pprof
* `runtime.NumGoroutine()`
* `go tool trace`

---

# 九、进阶方案（强烈推荐）

使用 errgroup：

```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(ctx)

g.Go(func() error {
    return doSomething(ctx)
})

if err := g.Wait(); err != nil {
    return err
}
```

优点：

* 自动取消
* 自动回收
* 代码更干净

---

# 十、一句话核心原则

> 每一个 goroutine，都必须有退出路径。

否则一定会在高并发下爆炸。

