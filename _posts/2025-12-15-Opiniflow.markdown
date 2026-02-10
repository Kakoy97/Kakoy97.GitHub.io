---
layout:     post
title: "Opiniflow复盘：如何构建一个黑客松获奖的跨市场套利工具"
date: 2025-12-15
author:     "Kakoy"
header-img: "img/home-bg-geek.jpg"
mathjax:    true
catalog:    true
categories: [Web3, 实战复盘]
tags: [Arbitrage, LLM, Redis, System Design]
description: "从跨市场套利逻辑到 LLM 三级匹配策略，深度解析 Opiniflow 的技术架构与性能优化之路。"
---

# Opiniflow复盘：如何构建一个黑客松获奖的跨市场套利工具
> 谈到套利，很多人会联想到复杂的量化模型。但实际上，套利的本质是从不同市场中寻找**信息差**。
> 本文将复盘 **Opiniflow** —— 一款集成了 Opinion 和 Polymarket 的跨市场套利面板。作为 Opinion 黑客松首期的获奖作品，我想分享它背后的技术实现，特别是如何利用 LLM 解决异构数据匹配，以及处理高并发流量的实战经验。


## 01. 什么是套利？不止是“科学家”的游戏

谈到套利，很多人脑海中浮现的是复杂的量化方程或高频交易。但回归本质，套利就是**捕捉信息差与市场波动带来的价差**。

在 Web3 的预测市场中，这一逻辑变得更为纯粹。无论是 Opinion 还是 Polymarket，用户都在对事件结果下注（Yes/No）。其核心数学逻辑非常简单：

$$Profit = 1 - (Price_{YES} + Price_{NO}) - Fees$$

只要在结算前，能够在两个市场（或者同一市场的不同对手盘）以总成本低于 $1 的价格买入一份 Yes 和一份 No，那么无论结果如何，你都锁定了 `1 - 成本` 的利润。

我最早接触套利，是使用推特大佬写的跨交易所自动对冲脚本。那时候大家的目标更多是刷积分。而 Opiniflow 的目标更进一步：**构建一个集成了 Opinion 和 Polymarket 的自动化跨市场套利面板，并结合 Telegram 实时通知，让“捡钱”变得可视化。**

![img](/img/opiniflow/opiniflow_main.png)
*(图：Opiniflow 实时套利监控面板)*

![img](/img/opiniflow/opiniflow_telegram.png)
*(图：Opiniflow telegram通知机器人)*


## 02. 核心挑战一：跨市场的“巴别塔” (模糊匹配)

开发 Opiniflow 遇到的第一个深坑，不是交易执行，而是**数据的对齐**。

A 市场的标题可能是 *"Will Trump win the 2024 election?"*
B 市场的标题可能是 *"US Election 2024: Trump Victory"*

几千个选项，语序不同、表达不同，如何让程序知道它们是同一个事件？我设计了一套**三层漏斗匹配机制**：

```mermaid
graph TD
    %% 样式定义独立成行
    classDef market fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef process fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef result fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000;

    subgraph Source [数据源头]
        M1["Market A Options"]:::market
        M2["Market B Options"]:::market
    end

    M1 & M2 --> L1

    subgraph Funnel [三层筛选漏斗]
        direction TB
        L1("第一层: 本地 LLM 模型"):::process
        L2("第二层: Gemini AI"):::process
        L3("第三层: 规则引擎"):::process
        
        L1 -- "向量化初筛 / Top 10" --> L2
        L2 -- "语义精准校验 / 剔除幻觉" --> L3
        L3 -- "结算时间对齐" --> Result
    end

    Result(("发现套利机会")):::result
```

### 第一层：本地 LLM 初筛（速度优先）
直接使用 API 请求大模型太贵且慢。我首先使用本地轻量级 LLM 模型对选项进行向量化粗筛，快速找出相似度最高的 Top 10 选项。
* **痛点**：LLM 对数字不敏感，容易将 *"Price > 5000"* 和 *"Price > 6000"* 混淆。
* **解决方案**：将数字转换成中文 5000->五千

### 第二层：Gemini 精确校验（准确优先）
在缩小范围后，引入 Google Gemini 模型进行语义分析。
* **优势**：节省 Token 消耗，同时利用 Gemini 较强的逻辑能力剔除“幻觉”匹配。

### 第三层：规则引擎兜底（风控核心）
这是最容易亏钱的地方——**结算规则差异**。
* 例如：A 市场按 TGE 后 12 小时价格结算，B 市场按 6 小时后结算。
* **解决方案**：再次利用 AI 验证结算条件，并强制忽略“起始时间”（只关注结算时间），确保两个赌约在时间维度上是等价的。




## 03. 核心挑战二：从"单体脚本"到"多进程流水线"

在 1.0 版本中，我的后端逻辑非常简单粗暴：一个名为 `scheduler_v2.py` 的单体脚本跑通所有逻辑。它同时负责 WebSocket 数据接收、ROI 计算、Gemini API 调用甚至数据库写入。结果就是：它卡爆了。

### 1. 痛点：被 API 阻塞的"主循环"

这非常像是一个写得糟糕的 Unity 游戏：我在 `Update()` 主循环里直接写了网络请求。在旧架构中，使用的是 `ThreadPoolExecutor`（线程池）。一旦系统发现套利机会，就会调用 Gemini 进行验证。虽然开了多线程，但 Gemini 的 API 响应通常需要 5-10 秒。当机会密集出现时，线程池迅速被占满，导致主线程的 Fast Loop（订单簿更新）被迫停滞。表现就是：当最需要抢速度的时候，行情数据反而不动了。

### 2. 重构思路：ECS 架构与职责分离

为了解决这个问题，我参考了游戏引擎的 ECS (Entity-Component-System) 设计思路，将系统拆分为独立的进程（Process），而非线程。利用 Redis 作为"内存总线"进行通信，彻底解耦了 IO 密集型任务和 CPU 密集型任务。

我将架构重新划分为三层流水线：

```mermaid
graph TD
    classDef io fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef cpu fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef store fill:#f5f5f5,stroke:#333,stroke-width:2px;

    subgraph Layer1 [感知层: Ingesters]
        P1["Poly Ingester"]:::io
        P2["Op Ingester"]:::io
    end

    subgraph Layer2 [逻辑层: Scanner]
        R1(("Redis Bus")):::store
        S1["ROI Scanner"]:::cpu
    end

    subgraph Layer3 [执行层: Workers]
        W1["Verification Worker"]:::io
        W2["Websocket Server"]:::io
    end

    P1 & P2 -- "实时写入 OrderBook" --> R1
    S1 -- "1. 读取最新状态\n2. 纯本地计算 ROI" --> R1
    R1 -- "Stream 推送高优机会" --> W1
    W1 -- "Gemini 验证通过" --> W2
```

#### A. 感知层 (Ingesters)：永不阻塞的"输入系统"

我重写了 `PolyIngester` 和 `OpinionIngester` 两个独立进程。

**职责**：只做一件事——维护 WebSocket 连接，将最新的 OrderBook 写入 Redis。

**优化**：实现了 MicroBuffer 机制和深度限制（只取前 8 档）。无论下游逻辑多么复杂，输入端的延迟始终保持在毫秒级。

#### B. 逻辑层 (ROI Scanner)：高速运转的"物理引擎"

这是新的核心组件。它不再直接发起任何 API 请求。

**机制**：每 10 秒从 Redis 拉取最新的快照，在本地进行纯数学计算（ROI 差值）。

**优势**：因为它不依赖外部网络，所以它极其稳定。就像游戏的物理引擎，无论渲染多慢，碰撞检测永远是准确的。

#### C. 执行层 (Verification Worker)：异步的"网络系统"

当 Scanner 发现 ROI > 0 的机会时，会将信号推送到 Redis Stream。Verification Worker 订阅这个队列，负责那些"慢操作"：

* 调用 Gemini 进行语义二次验证。
* 推送 Telegram 报警。
* 推送到前端 WebSocket。

即使 Gemini 挂了或者超时，它也只会阻塞 Worker 进程，绝不会影响 Ingester 继续更新价格，也不会影响 Scanner 继续发现新机会。

### 3. 流量危机与"静态化"救赎

除了计算性能，网络带宽是另一个大坑。上线初期，服务器报警显示每日流量消耗高达 70GB！

**排查发现**，是因为前端采用了轮询机制，每个用户每隔几秒就全量拉取一次所有 SQL 数据。

**最终优化策略**：

1. **后端静态化**：Manager 进程不再直接吐 SQL 数据，而是定时将计算好的 OrderBook 和 ROI 列表生成静态 JSON，存入 Redis。
2. **API 网关**：前端接口直接读取 Redis 中的静态数据（`redis.hgetall`），响应速度从 500ms 提升到了 5ms。
3. **放弃自动交易**：在本次重构中，我明确排除了自动交易（TradeExecutor）模块。作为独立开发者，我意识到数据的准确性和实时性远比自动下单更重要。把"观察"做到极致，比做一个不稳定的"黑盒"更有价值。

### 4. 性能对比总结

通过这次重构，Opiniflow 完成了脱胎换骨的变化：

| 指标 | 1.0 旧架构 (单体脚本) | 2.0 新架构 (多进程) |
|------|---------------------|-------------------|
| 核心机制 | ThreadPoolExecutor (易阻塞) | Multi-Process (完全隔离) |
| OOM 风险 | 高 (内存泄漏会导致全盘崩溃) | 低 (进程隔离，崩溃自重启) |
| API 延迟影响 | 严重 (Gemini 慢 = 整个系统慢) | 零影响 (异步验证) |
| 数据源 | SQL 轮询 (IO 瓶颈) | Redis 纯内存操作 |
| 吞吐量 | 约 20 events/min | > 1200 events/min |

## 04. 写在最后

开发 Opiniflow 的过程，让我对全栈开发有了更深的理解。

以前做游戏客户端，我们关注的是帧率、渲染和本地状态；而做 Web 服务端，关注的是高并发、数据库锁和流量成本。但在架构设计上，两者的本质是通用的——**都是关于数据的流动与状态的同步**。


> Opiniflow 依然在迭代中，欢迎关注我