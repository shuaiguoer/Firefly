---
title: Golang 并发控制：从 WaitGroup 到 errgroup 的深度实践与避坑指南
description: 深度解析 errgroup 工作机制，对比 WaitGroup 揭示其在错误捕获、生命周期控制及并发限流方面的优势，附实战 Demo。
published: 2026-02-06
updated: 2026-02-06
draft: false
category: 技术笔记
tags: [Golang, 并发编程, errgroup, WaitGroup, context, sync, 并发控制, 性能优化, 实战指南]
pinned: false
author: Shuai
licenseName: "CC BY 4.0"
sourceLink: ""
image: ./images/golang_errgroup.png
slug: golang-errgroup
---

# Golang 并发控制：从 WaitGroup 到 errgroup 的深度实践与避坑指南

在 Golang 的业务开发中，如果你还在用串行方式请求 A、B、C、D 四个不相关的接口，那你的程序性能可能和树懒没什么区别。今天我们聊聊如何从原始的 `WaitGroup` 进化到现代化的 `errgroup`，让你的并发代码从"包工头"升级为"精英主管"。

---

## 1. 核心执行逻辑：一人报错，全家收工

很多同学搞不清楚 `errgroup` 的 `cancel` 到底是什么时候触发的。看下面这张图就秒懂了：

```text
    +-------------------------------------------------------+
    | Main Goroutine (g, ctx := errgroup.WithContext)       |
    +-------------------------------------------------------+
               |                |                |
        g.Go(Func A)     g.Go(Func B)     g.Go(Func C)
               |                |                |
               |          [ B 报错返回 ]           |
               |                |                |
               |         自动调用 cancel()         |
               |<---------------|--------------->|
        [ A 感知 ctx.Done ]              [ C 感知 ctx.Done ]
        [ 提前退出 ]                      [ 提前退出 ]
               |                |                |
    +-------------------------------------------------------+
    | g.Wait() 立即接收到第一个非空 error 并返回              |
    +-------------------------------------------------------+
```

**执行流程说明：**

1. 主 goroutine 通过 `errgroup.WithContext` 创建 errgroup 实例和 context
2. 通过 `g.Go()` 启动多个子 goroutine
3. 任意一个 goroutine 返回非空 error 时，自动触发 `cancel()`
4. 其他 goroutine 通过 `ctx.Done()` 感知取消信号并优雅退出
5. `g.Wait()` 返回第一个非空 error

**关键点：** `cancel()` 是自动调用的，你不需要手动触发。这就是 errgroup 的魔法所在——它像个贴心的管家，一旦发现有人出事，立马通知所有人撤退。

---

## 2. 缘起：为什么不推荐原生 WaitGroup？

想象一下，你负责一个"大杂烩"接口，需要同时查询：用户信息、订单记录、优惠券。

### 方案对比

| 方案 | 执行方式 | 耗时 | 问题 |
|------|---------|------|------|
| **方案 A** | 串行执行 | 所有接口总和 | 性能极差，用户体验堪忧 |
| **方案 B** | 原生 WaitGroup | 最慢接口耗时 | 错误难捕捉、生命周期失控 |

### WaitGroup 的痛点

使用原生 `WaitGroup` 时，你会遇到以下问题：

- **错误难捕捉**：其中一个崩了，你得开 channel 费劲地接住 error，就像在暴风雨中接住一片雪花
- **生命周期失控**：如果 A 出错了，B 和 C 还在傻跑，浪费服务器资源，就像三个员工在办公室里加班，其实老板早就下班了

**代码示例对比：**

```go
// WaitGroup 的痛苦写法
var wg sync.WaitGroup
errChan := make(chan error, 3)

wg.Add(3)
go func() {
    defer wg.Done()
    if err := callA(); err != nil {
        errChan <- err
    }
}()
// ... 重复 3 次

wg.Wait()
close(errChan)
for err := range errChan {
    // 处理错误
}

// errgroup 的优雅写法
g, ctx := errgroup.WithContext(context.Background())
g.Go(func() error {
    return callA()
})
// ... 重复 3 次

if err := g.Wait(); err != nil {
    // 处理错误
}
```

看到区别了吗？errgroup 让你的代码从"写满注释的意大利面条"变成了"优雅的法式大餐"。

---

## 3. errgroup vs WaitGroup：从"包工头"到"精英主管"

如果说 `sync.WaitGroup` 是个只会数数的包工头，那 `errgroup` 就是个精英主管：

### 核心优势

- **共同进退**：只要一人报错，大家一起收工，避免资源浪费
- **错误收集**：自动帮你保存第一个返回的非空错误，不用自己写一堆 channel 逻辑
- **资源受控**：自带 `SetLimit` 限流，防止把下游打挂，就像给服务器装了个"防超载保护器"

### 重构清单

从 `WaitGroup` 迁移到 `errgroup` 的关键步骤：

```go
// 1. 初始化（替代 sync.WaitGroup{}）
g, ctx := errgroup.WithContext(ctx)

// 2. 启动任务（替代 wg.Add(1) + go func()）
g.Go(func() error {
    // 你的业务逻辑
    return nil
})

// 3. 等待完成（替代 wg.Wait()）
if err := g.Wait(); err != nil {
    // 处理错误
}
```

**迁移检查清单：**

- [ ] 替换 `sync.WaitGroup` 为 `errgroup.WithContext`
- [ ] 将 `wg.Add(1)` 和 `wg.Done()` 替换为 `g.Go()`
- [ ] 将错误 channel 替换为 `g.Wait()` 返回值
- [ ] 确保所有子 goroutine 使用同一个 context
- [ ] 添加 `ctx.Done()` 检查以支持优雅退出

---

## 4. 硬核进阶：g.SetLimit 与 g.TryGo

### 4.1 g.SetLimit(n)：控制并发上限

别把服务器当驴使！合理的 `SetLimit` 应该参考公式：

```
N = CPU核心数 × (1 + 等待时间 / 计算时间)
```

这个公式来自《计算机程序设计艺术》，是并发控制的黄金法则。

**使用示例：**

```go
g.SetLimit(10) // 最多同时运行 10 个 goroutine
```

**实战场景：** 假设你要处理 1000 个用户的批量操作，每个操作需要调用外部 API。如果不设置 `SetLimit`，你的程序会瞬间创建 1000 个 goroutine，可能导致：

- 内存爆炸
- 网络连接耗尽
- 下游服务被打挂（对方可能会拉黑你）

```go
// 错误示范：无限制并发
for _, user := range users {
    g.Go(func() error {
        return callExternalAPI(user)
    })
}

// 正确示范：设置合理限流
g.SetLimit(50) // 根据下游服务承受能力调整
for _, user := range users {
    g.Go(func() error {
        return callExternalAPI(user)
    })
}
```

### 4.2 g.TryGo：职场试探学

`TryGo` 就像是问女神："明天有空吗？" 如果女神（Limit 满了）说没空，你立马执行 fallback（降级方案），绝不舔着脸在那等。

**使用示例：**

```go
g.SetLimit(10)

// 尝试开启协程，如果并发已满则立即返回 false
if !g.TryGo(func() error {
    return callHeavyRPC(ctx)
}) {
    // 没位子了，走备选方案（如走缓存或返回默认值）
    return fallback()
}
```

**TryGo 的优势：**

- **非阻塞操作**：不会等待，立即返回结果
- **可以立即执行降级策略**：避免用户等待超时
- **防止资源耗尽导致的雪崩**：在系统高负载时自动降级

**实战场景：** 在秒杀系统中，当并发请求超过系统承载能力时，使用 `TryGo` 可以立即返回"系统繁忙"，而不是让用户一直等待，最终超时。

```go
func handleSeckillRequest(ctx context.Context, userID string) error {
    g.SetLimit(1000) // 系统最大并发处理能力

    if !g.TryGo(func() error {
        return processSeckill(ctx, userID)
    }) {
        return errors.New("系统繁忙，请稍后重试")
    }

    return nil
}
```

---

## 5. 完整 Demo：竞争执行 (Race Execution)

这段代码演示了利用 `errgroup` 实现"任一成功/失败即取消其他任务"的硬核操作。

### 5.1 代码实现

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"
    "golang.org/x/sync/errgroup"
)

func SearchData(ctx context.Context, name string, delay time.Duration, willFail bool) (string, error) {
    fmt.Printf("🚀 任务 [%s] 开始执行...\n", name)
    select {
    case <-time.After(delay):
        if willFail {
            return "", errors.New(name + " 发生故障")
        }
        return name + " 获取成功", nil
    case <-ctx.Done():
        fmt.Printf("⏹️ 任务 [%s] 收到取消信号，优雅退出。\n", name)
        return "", ctx.Err()
    }
}

func main() {
    g, ctx := errgroup.WithContext(context.Background())
    resultChan := make(chan string, 1) // 缓冲为 1，防止发送者阻塞

    tasks := []struct {
        n string
        d time.Duration
        f bool
    }{
        {"Service-A", 3 * time.Second, false},
        {"Service-B", 1 * time.Second, true}, // 它最快且报错，会触发全局 cancel
        {"Service-C", 5 * time.Second, false},
    }

    for _, t := range tasks {
        tt := t // 闭包陷阱：必须进行变量拷贝
        g.Go(func() error {
            res, err := SearchData(ctx, tt.n, tt.d, tt.f)
            if err == nil {
                select {
                case resultChan <- res:
                default:
                }
            }
            return err
        })
    }

    go func() {
        if err := g.Wait(); err != nil {
            fmt.Printf("\n📢 Wait() 最终捕获错误: %v\n", err)
        }
        close(resultChan)
    }()

    if finalRes, ok := <-resultChan; ok {
        fmt.Printf("\n🏆 拿到结果: %s\n", finalRes)
    } else {
        fmt.Println("\n💀 任务全部失败或被取消。")
    }
}
```

### 5.2 代码解析

**关键点说明：**

1. **闭包陷阱处理**：`tt := t` 必须进行变量拷贝，避免所有 goroutine 使用同一个变量。这是 Go 新手最容易踩的坑之一。
2. **Channel 缓冲**：`resultChan` 缓冲为 1，防止发送者阻塞。如果缓冲为 0，当没有接收者时，发送者会永远阻塞。
3. **优雅退出**：通过 `ctx.Done()` 监听取消信号，确保 goroutine 能够及时退出，避免资源泄漏。
4. **错误传播**：`g.Wait()` 返回第一个非空 error，这是 errgroup 的核心特性。

**执行流程：**

1. Service-B 最快执行（1秒）且报错
2. 触发全局 `cancel()`，取消其他任务
3. Service-A 和 Service-C 收到取消信号后优雅退出
4. `g.Wait()` 返回 Service-B 的错误

**输出示例：**

```text
🚀 任务 [Service-A] 开始执行...
🚀 任务 [Service-B] 开始执行...
🚀 任务 [Service-C] 开始执行...
⏹️ 任务 [Service-A] 收到取消信号，优雅退出。
⏹️ 任务 [Service-C] 收到取消信号，优雅退出。

📢 Wait() 最终捕获错误: Service-B 发生故障

💀 任务全部失败或被取消。
```

---

## 6. 最佳实践总结

### 6.1 使用建议

| 场景 | 推荐方案 | 说明 |
|------|---------|------|
| 简单并发等待 | `sync.WaitGroup` | 不需要错误处理时，比如批量日志处理 |
| 需要错误传播 | `errgroup` | 自动收集第一个错误，适合 API 聚合场景 |
| 需要并发限流 | `errgroup.SetLimit` | 防止资源耗尽，适合批量处理外部请求 |
| 需要降级策略 | `errgroup.TryGo` | 非阻塞，可立即降级，适合高并发场景 |

### 6.2 常见陷阱

#### 陷阱 1：忘记变量拷贝

```go
// 错误示范
for _, t := range tasks {
    g.Go(func() error {
        return process(t) // 所有 goroutine 都使用最后一个 t
    })
}

// 正确示范
for _, t := range tasks {
    tt := t // 必须拷贝
    g.Go(func() error {
        return process(tt)
    })
}
```

#### 陷阱 2：Context 传递不一致

```go
// 错误示范
g, ctx := errgroup.WithContext(context.Background())
g.Go(func() error {
    return process(context.Background()) // 使用新的 context，无法感知取消
})

// 正确示范
g, ctx := errgroup.WithContext(context.Background())
g.Go(func() error {
    return process(ctx) // 使用同一个 context
})
```

#### 陷阱 3：Channel 阻塞

```go
// 错误示范
resultChan := make(chan string) // 无缓冲
g.Go(func() error {
    resultChan <- "result" // 如果没有接收者，会永远阻塞
    return nil
})

// 正确示范
resultChan := make(chan string, 1) // 带缓冲
g.Go(func() error {
    resultChan <- "result"
    return nil
})
```

#### 陷阱 4：资源泄漏

```go
// 错误示范：没有检查 ctx.Done()
g.Go(func() error {
    for {
        // 无限循环，即使 context 被取消也不会退出
        doSomething()
    }
})

// 正确示范：检查 ctx.Done()
g.Go(func() error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            doSomething()
        }
    }
})
```

### 6.3 性能优化建议

1. **合理设置 SetLimit**：根据系统资源和下游服务承受能力设置，避免过度并发
2. **使用 TryGo 实现非阻塞降级**：在高并发场景下，及时降级比一直等待更重要
3. **监控 goroutine 数量**：使用 `runtime.NumGoroutine()` 监控，防止泄漏
4. **合理使用 context 超时控制**：设置合理的超时时间，避免 goroutine 长时间阻塞

**性能对比：**

```go
// 场景：处理 1000 个外部 API 请求

// 方案 A：无限制并发
// 耗时：~1 秒（最快）
// 问题：可能导致下游服务崩溃，内存占用高

// 方案 B：SetLimit(100)
// 耗时：~10 秒
// 优势：稳定，不会打挂下游服务

// 方案 C：SetLimit(100) + TryGo + 降级
// 耗时：~10 秒
// 优势：稳定 + 用户体验好（立即返回降级结果）
```

---

## 7. 实战应用场景

### 场景 1：微服务聚合

```go
func GetUserInfo(ctx context.Context, userID string) (*UserInfo, error) {
    g, ctx := errgroup.WithContext(ctx)
    var info UserInfo

    g.Go(func() error {
        var err error
        info.Profile, err = getProfile(ctx, userID)
        return err
    })

    g.Go(func() error {
        var err error
        info.Orders, err = getOrders(ctx, userID)
        return err
    })

    g.Go(func() error {
        var err error
        info.Coupons, err = getCoupons(ctx, userID)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }

    return &info, nil
}
```

### 场景 2：批量数据处理

```go
func ProcessBatch(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(50) // 控制并发数

    for _, item := range items {
        item := item // 闭包陷阱
        g.Go(func() error {
            return processItem(ctx, item)
        })
    }

    return g.Wait()
}
```

### 场景 3：健康检查

```go
func HealthCheck(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(5)

    services := []string{"db", "redis", "mq", "cache", "api"}

    for _, svc := range services {
        svc := svc
        g.Go(func() error {
            return checkService(ctx, svc)
        })
    }

    return g.Wait()
}
```

---

## 8. 参考资源

- [errgroup 官方文档](https://pkg.go.dev/golang.org/x/sync/errgroup)
- [Go 并发编程最佳实践](https://go.dev/doc/effective_go#concurrency)
- [Context 包详解](https://go.dev/blog/context)
- [Go 并发模式](https://github.com/golang/go/wiki/LockOSThread)

---

## 总结

`errgroup` 是 Go 并发编程的利器，它不仅解决了 `WaitGroup` 的痛点，还提供了更强大的错误处理和资源控制能力。从 WaitGroup 到 errgroup，不仅是工具的升级，更是思维方式的转变——从"只会数数的包工头"到"懂得管理的精英主管"。

掌握 `errgroup`，让你的并发代码更优雅、更健壮、更高效。记住，好的并发代码不仅要快，还要稳。就像人生一样，不仅要追求速度，更要懂得何时停下。

**最后送给大家一句话：** "并发编程就像谈恋爱，既要懂得放手（context cancel），又要懂得坚持（error handling），最重要的是要懂得控制节奏（SetLimit）。"

---

**相关阅读：**
- [Golang Context 详解：从入门到精通](#)
- [Go 性能优化实战指南](#)
- [微服务架构中的并发模式](#)
