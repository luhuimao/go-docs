# 一、什么是 Go 微服务架构？

**微服务架构（Microservices Architecture）**是将单体应用拆分为多个独立部署、独立运行、围绕业务领域构建的小型服务。

在 Go 生态中，微服务强调：

* 高并发处理能力
* 低内存占用
* 原生协程模型（goroutine）
* 易容器化（Docker 体积小）
* 静态编译部署简单

---

# 二、典型 Go 微服务系统架构图

```
        ┌──────────────┐
        │   Client     │
        └──────┬───────┘
               │
        ┌──────▼────────┐
        │ API Gateway   │
        └──────┬────────┘
               │
 ┌─────────────┼──────────────────┐
 │             │                  │
 ▼             ▼                  ▼
UserSvc     OrderSvc          PaymentSvc
 │             │                  │
 ▼             ▼                  ▼
MySQL        MySQL              Redis
```

通常还包含：

* 注册中心
* 配置中心
* 消息队列
* 链路追踪
* 日志系统
* 监控系统

---

# 三、Go 微服务核心技术栈

## 1️⃣ Web / RPC 框架

| 类型       | 推荐框架             |
| -------- | ---------------- |
| HTTP     | Gin / Echo       |
| RPC      | gRPC             |
| 企业级微服务框架 | go-zero / Kratos |

* **Gin**
* **Echo**
* **gRPC**
* **go-zero**
* **Kratos**

---

## 2️⃣ 服务注册与发现

* etcd

* Consul

* Kubernetes DNS

* **etcd**

* **Consul**

* **Kubernetes**

---

## 3️⃣ 配置中心

* Nacos

* Apollo

* etcd

* **Nacos**

* **Apollo**

---

## 4️⃣ 数据库

* MySQL

* PostgreSQL

* Redis

* **MySQL**

* **PostgreSQL**

* **Redis**

---

## 5️⃣ 消息队列

* Kafka

* RabbitMQ

* NSQ

* **Apache Kafka**

* **RabbitMQ**

* **NSQ**

---

## 6️⃣ 监控与链路追踪

* Prometheus

* Grafana

* Jaeger

* **Prometheus**

* **Grafana**

* **Jaeger**

---

# 四、Go 微服务分层设计（推荐）

建议采用 **Clean Architecture / DDD 分层**

```
/internal
    /domain        // 领域模型
    /repository    // 数据持久层
    /service       // 业务逻辑层
    /transport     // HTTP / gRPC 层
/cmd
    main.go
```

### 分层说明

| 层          | 作用        |
| ---------- | --------- |
| transport  | 接收请求、参数校验 |
| service    | 业务逻辑      |
| repository | DB 操作     |
| domain     | 纯业务模型     |

原则：

* 依赖单向
* interface 解耦
* 禁止跨层访问

---

# 五、微服务通信模式

## 1️⃣ 同步调用（RPC）

* gRPC
* HTTP REST

适用于：

* 强一致场景
* 实时查询

---

## 2️⃣ 异步调用（MQ）

* Kafka
* RabbitMQ

适用于：

* 削峰
* 最终一致
* 事件驱动

---

# 六、核心架构问题（面试高频）

## 1️⃣ 如何保证分布式事务？

方案：

* 本地消息表
* TCC
* Saga
* 两阶段提交（不推荐）

推荐：Saga + MQ

---

## 2️⃣ 如何防止服务雪崩？

* 熔断（circuit breaker）
* 限流（rate limit）
* 降级
* 超时控制
* 重试机制

可使用：

* go-zero 内置限流
* Hystrix 思路实现

---

## 3️⃣ 如何做服务治理？

* 服务注册
* 负载均衡
* 健康检查
* 灰度发布
* 动态配置

K8s 原生支持大量能力。

---

# 七、生产级 Go 微服务完整架构（企业实战）

```
用户请求
    ↓
Nginx / API Gateway
    ↓
微服务（Gin + gRPC）
    ↓
etcd 注册
    ↓
MySQL / Redis
    ↓
Kafka
    ↓
Prometheus + Grafana
```

特性：

* JWT 认证
* 中间件链
* 统一日志（zap）
* TraceID 透传
* OpenTelemetry 接入

---

# 八、Go 微服务优缺点

## 优点

* 并发性能强
* 部署简单
* 资源占用低
* 容器友好

## 缺点

* 分布式复杂度高
* 调试成本高
* 运维成本高

---

# 九、典型项目结构示例（go-zero 风格）

```
user-api/
    api/
    internal/
        config/
        handler/
        logic/
        svc/
```

---

# 十、如果你是高级 Go 工程师，建议掌握

* goroutine 调度模型
* channel 使用场景
* context 传递
* sync 包
* pprof 性能分析
* gRPC 底层原理
* HTTP2 原理
* Kubernetes 部署

---

# 总结

Go 微服务 =

> 高并发语言特性 + 轻量 RPC + 容器化部署 + 分布式治理体系