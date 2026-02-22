# Go 开发知识库

面向 Go 高级工程师的并发模型、微服务架构与工程实践文档集合，覆盖核心概念、底层原理、面试要点和生产落地方案。

---

## 📄 文档目录

### 并发编程

| 文档 | 简介 |
|---|---|
| [channel.md](./channel.md) | Channel 原理、阻塞语义、close 规则、与 Mutex/Atomic 的性能对比、20道高频面试题 |
| [goroutine.md](./goroutine.md) | Goroutine 生命周期、GMP 调度模型、goroutine 与线程的区别 |
| [如何防止goroutine泄漏.md](./如何防止goroutine泄漏.md) | 泄漏的常见原因、排查手段（pprof）、context + done channel 防泄漏模式 |
| [Context.md](./Context.md) | context 四种创建方式、取消树传播原理、微服务调用链中的超时收缩模型、分布式 TraceID 传播 |
| [defer.md](./defer.md) | defer LIFO 执行顺序、参数求值时机、性能开销与使用注意事项 |

---

### 微服务架构

| 文档 | 简介 |
|---|---|
| [Microservices_Architecture.md](./Microservices_Architecture.md) | Go 微服务全景介绍：技术栈选型（Gin/gRPC/go-zero）、Clean Architecture 分层、服务治理、分布式事务、雪崩防护 |
| [交易系统微服务架构设计.md](./交易系统微服务架构设计.md) | 生产级撮合型交易系统架构：撮合引擎设计、事件驱动 + 最终一致、Kafka 分片、资金账本强一致、可观测性方案 |

---

### Web 框架

| 文档 | 简介 |
|---|---|
| [主流web框架.md](./主流web框架.md) | Gin、Echo、Fiber、go-zero、Kratos 横向对比，选型建议 |

---

### 面试 & 工程实践

| 文档 | 简介 |
|---|---|
| [senior-go-dev-handbook.md](./senior-go-dev-handbook.md) | 高级 Go 工程师面试手册：内存分配（mcache/mcentral/mheap）、GC 三色标记、逃逸分析、GMP 模型、限流、优雅关闭、pprof 调优，共 16 道题 |

---

## 🗂️ 知识体系导图

```
Go 高级工程师
├── 并发模型
│   ├── GMP 调度器
│   ├── Goroutine 生命周期管理
│   ├── Channel（通信 + 同步）
│   ├── Context（取消 + 超时 + 传值）
│   └── Goroutine 泄漏防治
├── 微服务架构
│   ├── 服务分层（Clean Architecture / DDD）
│   ├── RPC 通信（gRPC）
│   ├── 消息队列（Kafka）
│   ├── 服务治理（限流 / 熔断 / 链路追踪）
│   └── 交易系统设计（撮合 / 清算 / 风控）
├── 运行时 & 性能
│   ├── 内存分配模型
│   ├── GC 原理
│   ├── 逃逸分析
│   └── pprof 性能分析
└── Web 框架
    ├── Gin / Echo（轻量 HTTP）
    └── go-zero / Kratos（企业级微服务）
```

