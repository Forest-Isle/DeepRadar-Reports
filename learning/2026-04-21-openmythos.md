# OpenMythos：从第一原理重构 Claude Mythos 架构

> 来源：[kyegomez/OpenMythos](https://github.com/kyegomez/OpenMythos) · 2026-04-21

---

## 背景与问题

Claude Mythos 是 Anthropic 尚未公开完整技术细节的下一代模型架构。社区对其核心设计充满好奇，但缺乏可复现的开源实现。OpenMythos 项目基于公开研究文献，从第一原理对 Mythos 的假想架构进行理论重构，旨在让研究者可以在本地运行、实验并理解这类计算自适应的深度可变推理模型。

核心问题：**如何在 Transformer 基础上构建"可变计算深度"的推理机制，使模型能对困难问题自动投入更多计算？**

---

## 核心方法：三阶段 Recurrent-Depth Transformer（RDT）

OpenMythos 的架构分为三个串行阶段：

### 1. Prelude（前导）
标准 Transformer 块，负责初步特征提取与上下文编码。参数量由 `prelude_layers` 控制。

### 2. Recurrent Block（循环块）——核心创新
这是与传统 Transformer 最大的区别。该模块执行最多 `max_loop_iters` 次迭代，每次迭代都在相同参数上运行，类似于隐式深度网络（implicit deep network）。

关键约束：循环块内包含一个注入矩阵 `A`，其**谱半径 ρ(A) 必须严格 < 1**，以保证循环收敛（Lipschitz 稳定性条件）。若 ρ(A) ≥ 1，推理将发散。

```python
A = model.recurrent.injection.get_A()
print(f"Spectral radius ρ(A) max: {A.max().item():.4f}  # 必须 < 1")
```

推理时可动态指定循环次数（`n_loops`），困难问题投入更多迭代，简单问题减少迭代——这正是"计算自适应"的本质。

### 3. Coda（尾段）
最终几层 Transformer 块，将循环块的输出投影到词表空间。

---

## 注意力机制：MLA vs GQA 可切换

OpenMythos 同时支持两种注意力变体：

| 机制 | 说明 | 适用场景 |
|------|------|----------|
| **GQA**（分组查询注意力） | 多个 Query 头共享一组 K/V 头，降低 KV Cache 内存 | 推理内存受限 |
| **MLA**（多头潜注意力） | DeepSeek 提出，通过低秩压缩 K/V，引入 RoPE 位置编码，支持百万上下文 | 超长文本推理 |

MLA 配置需额外指定 `kv_lora_rank`、`q_lora_rank`、`qk_rope_head_dim` 等参数，本质是将 K/V 投影到低秩潜空间后再展开，减少显存占用同时保留表达能力。

---

## 前馈层：稀疏 MoE（混合专家）

每个 Transformer 块的 FFN 替换为稀疏 MoE：
- `n_experts`：总专家数（如 128）
- `n_shared_experts`：对所有 token 激活的共享专家数
- `n_experts_per_tok`：每个 token 路由到的专家数（稀疏激活）

这使得参数量可扩展到 1T 量级，而单次前向传播的实际计算量远低于稠密模型。

---

## 关键结论

1. **深度可变推理**：通过控制 `n_loops` 参数，同一模型可在推理时灵活权衡速度与质量，无需训练多个模型。
2. **稳定性是硬约束**：ρ(A) < 1 不是建议，是数学保证——违反此约束会导致梯度爆炸和推理发散。
3. **MLA + RDT 组合**：MLA 解决长上下文 KV Cache 问题，RDT 解决计算深度问题，两者正交互补。

---

## 技术细节深挖

### 训练脚本中的关键设计
3B 模型在 FineWeb-Edu 上的训练支持单 GPU 和 `torchrun` 多 GPU。训练时 `n_loops` 通常设为固定值（如 4-8），推理时可超出训练范围（课程学习效应）。

### 模型规模对比
| 变体 | 参数规模 | Loop iters | 上下文 |
|------|---------|------------|--------|
| mythos_1b | 2048 dim | 16 | 4k |
| mythos_100b | 8192 dim | 32 | 1M |
| mythos_1t | 16384 dim | 64 | 1M |

规模越大，循环次数越多，上下文越长——体现了"大模型需要更深推理"的直觉。

---

## 横向对比

| 架构 | 计算深度 | 参数效率 | 代表模型 |
|------|---------|---------|---------|
| 标准 Transformer | 固定 | 基准 | GPT-4, LLaMA |
| MoE Transformer | 固定 | 高（稀疏激活） | Mixtral, DeepSeek-V3 |
| **RDT (OpenMythos)** | **可变** | 高（循环复用） | 假想 Claude Mythos |
| Universal Transformer | 可变 | 低（全层循环） | 学术研究 |

RDT 与 Universal Transformer 的核心区别：RDT 只有中间的 Recurrent Block 循环，Prelude/Coda 固定，避免了全层循环的低效。

---

## 启发与思考

1. **推理时计算扩展（Test-Time Compute Scaling）**：o1/o3 用自回归思维链扩展计算，RDT 用循环深度扩展——两种路径殊途同归，但 RDT 不依赖显式 CoT token，更适合非文本任务。
2. **ρ(A) < 1 的工程含义**：训练时需要监控谱半径，可作为模型健康度指标。若谱半径趋近 1，说明梯度流动正在退化。
3. **开源价值**：即使 OpenMythos 对 Claude Mythos 的猜测不完全准确，它提供了一个研究"深度可变 Transformer"的清晰实现框架。

---

## 延伸阅读

- [Universal Transformers (Dehghani et al., 2018)](https://arxiv.org/abs/1807.03819)
- [DeepSeek-V3 MLA 机制](https://arxiv.org/abs/2412.19437)
- [Mixture of Experts 综述](https://arxiv.org/abs/2407.06204)
- [Test-Time Compute Scaling](https://arxiv.org/abs/2408.03314)
