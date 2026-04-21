# LLM 可观测性工具横向评测：LangSmith、Langfuse、Helicone、Phoenix 的真实短板

> 来源：[dev.to/soufian_azzaoui — I tried LangSmith, Langfuse, Helicone, and Phoenix](https://dev.to/soufian_azzaoui_85ea1c030/i-tried-langsmith-langfuse-helicone-and-phoenix-heres-what-each-gets-wrong-2cjk) · 2026-04-21

---

## 背景与问题

LLM 应用的可观测性（Observability）需求与传统软件截然不同：你不只需要知道请求成功/失败，还需要理解**为什么 Agent 做了这个决策**、**哪条推理路径导致了错误输出**、**各工具调用之间的因果关系**。

作者在构建 3 个月生产级 LLM 应用的过程中，系统评测了四款主流工具，并在发现所有工具都无法满足需求后，自己构建了 AgentLens。

---

## 四款工具深度评测

### LangSmith
**优势**：与 LangChain/LangGraph 深度集成，无缝追踪 Chain、Agent、Tool 调用。

**核心问题**：
1. **定价过激进**：$39/seat/month，5 人团队每月 $195，还没记录一条 trace。
2. **数据保留短**：默认 14 天。延长到 400 天需 $5/千次 trace，是基础价格的 10 倍，没有中间档。
3. **数据主权**：仅支持美国区域，欧盟团队面临 GDPR 合规风险，企业版才支持数据残留选择。
4. **框架锁定**：为 LangChain 设计，使用其他框架时集成体验显著下降。

**适合场景**：全面押注 LangGraph 的团队，且不在乎数据主权。

---

### Langfuse
**优势**：开源（25k+ GitHub stars）、可自托管、框架无关、定价透明。

**核心问题**：
1. **无 MCP 支持**：如果你在使用 Claude + MCP 工具，Langfuse 对 MCP 工具调用是盲的——这是 2026 年构建 Claude Agent 的致命缺陷。
2. **告警能力弱**：生产监控需要依赖 Datadog/Grafana 补充，增加集成复杂度。
3. **15% 延迟开销**：基准测试中可测量，对延迟敏感应用（实时对话）不友好。

**总体判断**：当前最值得推荐的开源选择，MCP gap 是唯一真正的阻塞项。

---

### Helicone
**优势**：极简接入——字面意义上一行代码，修改 base URL 即完成集成。

**核心问题**：
1. **代理架构的天花板**：Helicone 是 HTTP 代理，只能看到网络层流量。Agent 内部的决策过程、工具选择逻辑、多步推理路径——完全不可见。
2. **无 Span 级可见性**：想知道"Agent 为什么在第 3 步决定调用这个工具"？Helicone 无法回答。
3. **自托管有限**：云端为主，自托管方案不完整。

**适合场景**：简单 API 包装应用的成本追踪，不适合复杂 Agent。

---

### Phoenix（Arize）
**优势**：OpenTelemetry 原生、ML 可观测性根基扎实，适合有 Arize 基础设施的团队。

**核心问题**：
1. **面向 ML 工程师，不面向 LLM 开发者**：UI 和概念模型来自传统 ML 监控，对于构建 LLM 应用的后端工程师有学习曲线。
2. **设置复杂**：相比 Helicone 的一行代码，Phoenix 需要更多配置。
3. **生态依赖**：最大价值来自与 Arize 平台的结合，独立使用时价值有限。

**适合场景**：已有 Arize 基础设施的 ML 平台团队扩展到 LLM 监控。

---

## 关键结论

1. **MCP 支持是 2026 年的分水岭**：随着 Claude + MCP 成为主流开发范式，不支持 MCP 追踪的工具实际上对 Agent 内部是盲的。这将成为工具选型的关键标准。
2. **代理 vs 埋点架构的根本差异**：Helicone（代理）只能看到"什么请求被发出了"，Phoenix/Langfuse（埋点）能看到"为什么发出这个请求"。两者解决不同问题。
3. **定价模型的隐形成本**：LangSmith 的 per-seat 定价在小团队快速验证阶段是阻力，Langfuse 的按量计费更符合 LLM 应用的实际使用曲线。

---

## 技术细节深挖

### OpenTelemetry 在 LLM 可观测性中的位置
Phoenix 的 OTel 原生支持意味着你可以将 LLM trace 与现有的分布式追踪系统（Jaeger、Tempo）集成。对于混合架构（LLM + 传统微服务），这是真正的优势。

### 自托管的实际成本
Langfuse 自托管需要：PostgreSQL + Redis + Node.js 服务。在 AWS 上约 $30-80/月（小型部署），相比 LangSmith 的 $195+/月 有显著优势，但需要运维能力。

### AgentLens 的核心设计
作者自建方案的四个需求全部开源工具都未完整满足：
```python
import agentlens
agentlens.init()
agentlens.patch_anthropic()  # 自动追踪每次 Claude 调用
```
- ✅ 完整 Agent 行为追踪（每个工具调用、每个决策）
- ✅ 自托管（数据不出服务器）
- ✅ 按项目计费，非 per-seat
- ✅ 原生 MCP 支持（唯一做到的工具）

---

## 可观测性工具选型矩阵

| 工具 | MCP 支持 | 自托管 | 定价友好 | Agent 追踪深度 | 接入难度 |
|------|---------|--------|---------|-------------|---------|
| LangSmith | ❌ | ❌ | ❌ | 高（LangChain） | 低 |
| **Langfuse** | ❌ | ✅ | ✅ | 高 | 低 |
| Helicone | ❌ | 有限 | ✅ | 低（代理层） | 极低 |
| Phoenix | ❌ | ✅ | 中 | 高 | 高 |
| **AgentLens** | **✅** | **✅** | **✅** | **高** | 低 |

---

## 启发与思考

1. **可观测性工具滞后于框架演进**：MCP 在 2024 年底发布，到 2026 年中仍然没有主流可观测性工具完整支持——这是工具生态通常滞后于核心框架 6-18 个月的典型案例。
2. **构建 vs 购买的新决策点**：当现有工具的 gap 足够大（如 MCP 盲区），自建有时是唯一合理选项，但自建意味着你进入了"基础设施维护"业务。
3. **Langfuse 的社区护城河**：25k+ stars、框架无关、MIT 协议——即使有 MCP gap，Langfuse 的开源生态给了它持续迭代的能量。MCP 支持很可能在未来几个月内出现。

---

## 延伸阅读

- [Langfuse GitHub](https://github.com/langfuse/langfuse)
- [OpenTelemetry LLM Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [MCP 规范文档](https://modelcontextprotocol.io)
- [Arize Phoenix](https://github.com/Arize-ai/phoenix)
- [LLM Observability 综述 (2025)](https://arxiv.org/abs/2504.00000)
