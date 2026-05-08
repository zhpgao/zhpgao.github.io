---
layout: post
title: "AI Agent Request 计费机制解析"
date: 2026-05-08
categories: [AI, LLM]
tags: [cursor, copilot, deepseek, zhipu, pricing]
---

# AI Agent "Request" 计费机制解析

> **日期：** 2026-05-08
> **覆盖平台：** Cursor、GitHub Copilot、智谱 GLM、DeepSeek V4

---

## 1. 什么是 "Request"

在 AI 编程助手的语境下，"request"（请求）是计费的基本单位。但不同平台对 "request" 的定义、计量方式和收费模型截然不同。

大致可以分为两种计费范式：

| 范式 | 代表 | 计量单位 | 特点 |
|------|------|---------|------|
| **Request-based** | Cursor (legacy 计划)、GitHub Copilot (截至 2026.05) | 次数 / 请求 | 简单直观，但不反映实际资源消耗差异 |
| **Token-based** | Cursor (新计划)、智谱、DeepSeek、GitHub Copilot (2026.06+) | Token 数 | 精确反映消耗，但用户难以预估费用 |

行业趋势是从 request-based 向 token-based 迁移，因为 Agent 模式下单个请求的复杂度差异极大——一个简单问答可能消耗 1K tokens，而一个复杂的代码重构可能消耗 100K+ tokens。

---

## 2. Cursor 的计费机制

Cursor 目前有**两种计费模式并存**：新的 token-based 计划和仍在使用的 legacy request-based 计划。

### 2.1 新计划：Token-Based（当前默认）

2025 年下半年起，Cursor 新注册用户默认采用 token-based 计费，分为两个使用池：

**Auto 模式**（固定费率，不随底层模型变化）：

| 项目 | 费率 |
|------|------|
| 输入 token | $1.25 / 1M tokens（含 cache write） |
| 输出 token | $6.00 / 1M tokens |
| 缓存读取 | $0.25 / 1M tokens |

**Premium / 指定模型**（按所选模型的 API 公开费率计费）：选择特定模型时走 API 池。

| 计划 | 月费 | 包含 API 额度 |
|------|------|--------------|
| Hobby | 免费 | 有限 |
| Pro | $20/月 | $20 |
| Pro+ | $60/月 | $70 |
| Ultra | $200/月 | $400 |

在 token-based 计划下，**不按消息条数收费**——你在一次会话中发 1 条消息还是 100 条消息，费用纯粹由实际消耗的 token 总量决定。

### 2.2 Legacy 计划：Request-Based（500 次/月）

部分早期用户和企业用户仍在使用 legacy request-based 计划：

- **Pro（旧版）：** 500 fast requests/月，超出后回退到 slow mode（排队处理）
- **Teams（旧版）：** 按用户分配 request 额度

**什么算 1 个 request：**

| 操作 | 是否消耗 request | 说明 |
|------|-----------------|------|
| 用户发送一条消息 | **是（1 个 request）** | 每次按"发送"都算 1 个 request |
| Agent 自动执行的工具调用 | 否 | 包含在当前 request 内 |
| 达到工具调用上限后点 "Continue" | **是** | 算一个新的 request |
| Tab 代码补全 | 否 | 单独计数，不消耗 request |

**核心结论：在 request-based 计划中，你在一次会话中发了 N 条消息，就消耗 N 个 request。** 每条消息内部无论 Agent 调用了多少次工具，都只算 1 个 request。

### 2.3 一条消息的内部处理流程

无论哪种计费模式，一条消息内部的处理流程是一样的：

```
用户发送一条消息
    │
    ▼
Cursor 构建系统提示
    │ +隐藏系统提示（数百 tokens）
    │ +当前文件内容、项目元数据
    │ +对话历史（包含本次会话中所有之前的消息）
    │
    ▼
第一次 LLM 调用 → 模型决定使用工具（如读取文件）
    │
    ▼
工具执行 → 返回结果 → 构建新上下文
    │ 重新发送：系统提示 + 原始问题 + 之前的对话 + 工具结果
    │
    ▼
第二次 LLM 调用 → 模型继续推理或使用更多工具
    │
    ▼
... 重复直到模型给出最终回复或达到工具调用次数上限 ...
    │
    ▼
最终回复返回给用户
```

**关键点：**
- 每次工具调用都会重新发送**完整上下文**，多次工具调用会让 token 消耗成倍膨胀
- 随着对话持续，对话历史越来越长，后续消息的 token 消耗（或处理时间）也会递增
- 在 token-based 计划中，这直接影响费用；在 request-based 计划中，这影响响应速度

### 2.4 Context 对费用的影响

对于 token-based 计划，上下文大小直接影响费用：

| 因素 | 影响 |
|------|------|
| 对话长度 | 每轮都发送完整对话历史，越长越贵 |
| 打开的文件数量 | 自动附加的文件内容增加输入 tokens |
| @引用的文件/文件夹 | 显式引用的内容全部计入输入 tokens |
| Skill/Instruction 文件 | 注入到系统提示中，每次调用都重复计入 |
| 工具调用次数 | 每次工具调用重新发送完整上下文 |
| 缓存命中率 | 缓存读取费率（$0.25/M）远低于非缓存读取（$1.25/M） |

---

## 3. GitHub Copilot 的计费机制

### 3.1 当前模型：Premium Request（截至 2026.05）

GitHub Copilot 目前使用**按请求计费**模型：

**基本规则：** 每个用户 prompt 算一个 premium request，乘以模型倍率。

| 计划 | 月费 | Premium Request 配额 |
|------|------|---------------------|
| Free | 免费 | 50 次/月 |
| Pro | $10/月 | 300 次/月 |
| Student | 免费 | 300 次/月 |
| Pro+ | $39/月 | 1,500 次/月 |

超出后可以按 $0.04/次 购买额外请求。

**模型倍率：** 不同模型消耗不同数量的 premium requests。包含模型（GPT-5 mini、GPT-4.1、GPT-4o）在付费计划下不消耗 premium requests。高级模型有 1x~20x 的倍率。使用 Auto 模式可获得 10% 倍率折扣。

**Agent 模式的特殊规则：**
- 只有用户主动发送的 prompt 计为 premium request
- Agent 自主执行的工具调用**不计费**
- Cloud Agent 每个 session 算一个 premium request（乘以模型倍率）
- session 期间的实时 steering comment 也各算一个 premium request

### 3.2 即将到来的转变：Usage-Based Billing（2026.06.01 起）

GitHub 已宣布从 2026 年 6 月 1 日起，Copilot 将从 request-based 切换到 **usage-based**（基于 token 消耗）计费。这意味着：
- 不再按"次"计费，而是按实际消耗的输入/输出/缓存 token 数量计费
- 按各模型的公开 API 费率收费
- 更精确地反映实际资源消耗

这个转变说明了行业共识：在 Agent 时代，按 request 计费不够合理——一个简单问答和一个需要 50 次工具调用的复杂重构，资源消耗天差地别但都算"1 次请求"。

### 3.3 Request 的内部结构

GitHub Copilot 的 request 内部结构与 Cursor 类似：

```
用户 prompt（= 1 premium request × 模型倍率）
    │
    ▼
Copilot 构建上下文
    │ +系统指令（copilot-instructions.md 等）
    │ +当前文件 + 工作区上下文
    │ +对话历史
    │
    ▼
LLM 推理 → 可能执行工具（文件编辑、终端、搜索等）
    │ Agent 自主的工具调用不算 premium request
    │
    ▼
最终回复返回给用户
```

**关键区别：** 在 Copilot 中，Agent 的自主工具调用**不计为 premium request**。只有用户发送的每个 prompt 才算。这比 Cursor 的 token-based 模型对用户更友好（不会因为 Agent 多调了几次工具就多收费）。

---

## 4. 智谱 GLM 的计费机制

### 4.1 纯 Token 计费

智谱采用最直接的 token 计费模型：按输入和输出的 token 数量分别收费。

| 模型 | 输入 ($/M tokens) | 输出 ($/M tokens) | 上下文窗口 |
|------|-------------------|-------------------|-----------|
| GLM-5.1 | $1.40 | $4.40 | 203K |
| GLM-5 | $1.00 | $3.20 | 203K |
| GLM-4.7 | $0.40 | $1.50 | 205K |
| GLM-4.7 Flash | $0.06 | $0.40 | 205K |
| GLM-4 Flash | 免费 | 免费 | 128K |

### 4.2 没有 "Request" 概念

智谱的 API 是纯粹的 LLM API，没有 "request" 的包装概念。每次 API 调用就是一次 token 消耗，费用 = 输入 tokens × 输入单价 + 输出 tokens × 输出单价。

如果你用智谱的 API 构建一个 Agent（比如用 LangChain/OpenAI SDK），Agent 的每次工具调用都会产生一次独立的 API 调用，每次都消耗 tokens。没有"包含在 request 内不额外收费"的概念。

### 4.3 订阅计划

智谱也提供面向开发者的包月计划：
- Lite: ~$10/月
- Pro: ~$30/月
- Max: ~$80/月

这些计划包含一定的 token 额度，超出后按上述 API 费率计费。

---

## 5. DeepSeek V4 的计费机制

### 5.1 Token 计费 + 缓存分层

DeepSeek 的计费模型在 token-based 的基础上增加了**缓存分层**——缓存命中和未命中有不同的价格：

#### DeepSeek V4 Flash

| 类型 | $/M tokens |
|------|-----------|
| 输入（缓存命中） | $0.0028 |
| 输入（缓存未命中） | $0.14 |
| 输出 | $0.28 |

#### DeepSeek V4 Pro（75% 折扣至 2026.05.31）

| 类型 | 折扣价 $/M tokens | 原价 $/M tokens |
|------|-------------------|----------------|
| 输入（缓存命中） | $0.003625 | $0.0145 |
| 输入（缓存未命中） | $0.435 | $1.74 |
| 输出 | $0.87 | $3.48 |

### 5.2 缓存命中率的重要性

DeepSeek 的缓存定价差异极大：

- V4 Flash：缓存命中 $0.0028 vs 未命中 $0.14 → **相差 50 倍**
- V4 Pro：缓存命中 $0.003625 vs 未命中 $0.435 → **相差 120 倍**

这意味着：
- **重复发送相同的系统提示和对话前缀**时，缓存可以大幅降低成本
- **Agent 的多轮工具调用**由于每次都重新发送完整上下文，前缀相同的部分会被缓存，后续调用的输入成本远低于首次
- 这就是为什么 Cursor 的 Auto 模式也有 "cache read" 的单独费率（$0.25/M vs 非缓存 $1.25/M）

### 5.3 同样没有 "Request" 概念

与智谱一样，DeepSeek 提供的是纯粹的 LLM API，按 token 计费。没有"一次请求包含多少工具调用"的概念。

---

## 6. 四个平台的计费对比

| 维度 | Cursor | GitHub Copilot (当前) | 智谱 GLM | DeepSeek V4 |
|------|--------|---------------------|---------|-------------|
| **计费单位** | Token（新）/ Request（legacy） | Premium Request | Token | Token |
| **"Request" 概念** | Legacy 有（每条消息 = 1 request）；新计划无 | 有（用户 prompt 为界） | 无 | 无 |
| **工具调用计费** | 按 token 累计 | Agent 自主调用不计费 | 独立 API 调用 | 独立 API 调用 |
| **缓存优惠** | 有（$0.25 vs $1.25/M） | 未公开 | 无 | 有（最高 120x 差异） |
| **模型倍率** | 无（Auto 固定费率） | 有（1x~20x） | 无（每模型独立定价） | 无 |
| **免费层** | Hobby（有限） | 50 次/月 | GLM-4 Flash 免费 | 无 |
| **趋势** | 已转 token-based | 2026.06 转 token-based | 始终 token-based | 始终 token-based |

---

## 7. 总结

### 核心概念

1. **Request** 是部分平台的计费抽象——用户每发一条消息算一个 request（GitHub Copilot 当前、Cursor legacy 计划）。Cursor 新计划已转为纯 token 计费
2. **Token** 是底层 LLM 的计费单位——实际消耗的计算资源度量
3. 行业正从 request-based 向 token-based 迁移，因为 Agent 时代下不同交互的复杂度差异过大——一个简单问答和一个需要多次工具调用的重构任务，资源消耗天差地别
4. **缓存**是降低成本的关键杠杆——重复的上下文前缀可以享受 5x~120x 的价格优惠

### 选型建议

| 使用场景 | 推荐平台 | 原因 |
|---------|---------|------|
| 日常编码助手 | Cursor Auto / Copilot 免费模型 | 固定费率或免费 |
| 重度 Agent 使用 | Cursor Pro+ / Ultra | 大额度 + token-based 可预测 |
| API 集成 / 自建 Agent | DeepSeek V4 Flash | 极低的缓存命中价格 |
| 轻度使用 / 学习 | 智谱 GLM-4 Flash | 完全免费 |
