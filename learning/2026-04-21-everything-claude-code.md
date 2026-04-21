# Everything Claude Code：AI Agent 性能优化系统的工程实践

> 来源：[affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) · 2026-04-21

---

## 背景与问题

开发者在使用 Claude Code 等 AI Agent 工具时面临一个系统性困境：默认配置无法满足生产级需求。Token 消耗过高、跨会话上下文丢失、安全边界不清、模型选择随意——这些问题单独看都可以手动应对，但合在一起构成了一个需要系统性解决的"Agent 性能问题"。

Everything Claude Code（ECC）在 10 个月密集日常使用构建真实产品的过程中演化出了一套完整的解决方案体系，并在 Anthropic Hackathon 中获奖。

---

## 核心方法：六维性能优化框架

### 1. Token 优化（Token Optimization）
- **模型路由**：`/model-route` 命令根据任务复杂度动态选择模型，避免用 Sonnet 做能用 Haiku 完成的任务
- **系统提示精简**：移除冗余描述，保留结构化指令
- **背景进程隔离**：监控等低优先级任务使用廉价模型，核心推理用高能力模型

### 2. 内存持久化（Memory Persistence）
跨会话上下文保留是最常见的痛点。ECC 的解法：
- **Hooks 架构**：在 `SessionStart`/`SessionEnd` 事件中自动保存/加载上下文
- 不依赖模型记忆，而是将关键上下文写入文件系统，下次会话启动时注入
- 支持 `ECC_HOOK_PROFILE=minimal|standard|strict` 控制 hook 粒度

### 3. 持续学习（Continuous Learning）
- **Instinct 系统**：从每次会话中自动提取可复用的"本能"（Instinct），带置信度评分
- `/instinct-import` 命令将成功模式提炼为结构化规则
- v1.4.1 修复了关键 bug：`parse_instinct_file()` 之前会静默丢弃 Action/Evidence/Examples 节后的所有内容

### 4. 验证循环（Verification Loops）
- Checkpoint 评估 vs 持续评估的权衡
- Grader 类型：LLM-as-judge、单元测试、集成测试
- pass@k 指标：对同一任务运行 k 次，计算至少 1 次成功的概率

### 5. 并行化（Parallelization）
- **Git Worktrees 方法**：为每个并行任务创建独立工作树，避免文件系统冲突
- **Cascade Method**：主 Agent 分解任务 → 多个 Sub-Agent 并行执行 → 主 Agent 汇总
- 判断何时扩展实例：当任务间没有共享状态依赖时

### 6. Sub-Agent 编排（Subagent Orchestration）
- **上下文问题**：每个 Sub-Agent 携带完整对话历史，大任务容易超出窗口
- **迭代检索模式**：Sub-Agent 不携带完整历史，而是按需从文件系统检索相关上下文

---

## 关键结论

1. **ECC 是"harness 性能系统"而非配置包**：v1.8.0 明确定义了这个定位——目标是让 Agent 框架本身运行得更好，而非只是提供更好的提示词。
2. **Hook 可靠性是基础设施**：SessionStart root fallback、Stop 阶段会话摘要、脚本化 hooks 替代内联 one-liner——这些都是生产环境的必要条件。
3. **12 语言生态系统**：TypeScript、Python、Go、Java、PHP、Perl、Kotlin、C++、Rust、Swift……ECC 试图成为"所有语言的 Agent 最佳实践集合"。

---

## 技术细节深挖

### ECC 2.0 Alpha：Rust 控制平面
`ecc2/` 目录包含一个 Rust 重写的控制平面原型，暴露以下命令：
```
dashboard | start | sessions | status | stop | resume | daemon
```
Rust 的选择出于性能和稳定性考虑：Hook 代码在每次 Agent 操作时都会执行，Python 的启动开销在高频场景下不可忽略。

### SQLite 状态存储
v1.9.0 引入 SQLite 作为状态存储：
- 记录哪些组件已安装（防止重复安装）
- 会话历史查询 CLI
- Skill 演化基础设施（为自我改进 Skill 铺路）

### AgentShield 安全扫描
`/security-scan` 直接从 Claude Code 运行 AgentShield，包含 1282 个测试、102 条规则，覆盖：提示注入、工具调用越界、数据泄露等攻击向量。

---

## 横向对比

| 维度 | ECC | 裸 Claude Code | Cursor Rules | OpenCode Plugin |
|------|-----|---------------|-------------|----------------|
| 内存持久化 | Hook 自动化 | 手动 | 无 | 部分 |
| Token 优化 | 系统化 | 手动 | 无 | 无 |
| 安全扫描 | AgentShield 集成 | 无 | 无 | 无 |
| 多语言支持 | 12 种 | 无 | 无 | 无 |
| 跨 harness | 6 种（CC/Cursor/Codex等） | 仅 CC | 仅 Cursor | 仅 OpenCode |

---

## 启发与思考

1. **"配置即代码"的极致**：ECC 的核心洞察是，AI Agent 的行为可以像软件一样被工程化管理——版本控制、测试、持续集成。这与传统"调提示词"的直觉相反。
2. **Hook 架构的普适性**：ECC 的 Hook 系统本质上是 Agent 的 AOP（面向切面编程）——在不修改核心逻辑的情况下，在关键事件点注入横切关注点（日志、安全、内存）。
3. **自我改进的飞轮**：Instinct 系统 + Continuous Learning + Skill 演化，构成了一个"Agent 越用越聪明"的正向循环。这是 ECC 最有前瞻性的设计。

---

## 延伸阅读

- [ECC 官方指南（Shorthand）](https://x.com/affaanmustafa/status/2012378465664745795)
- [Claude Code Hooks 文档](https://docs.anthropic.com/claude-code/hooks)
- [AgentShield 安全框架](https://github.com/affaan-m/everything-claude-code/tree/main/security)
- [Git Worktrees 并行 Agent 模式](https://git-scm.com/docs/git-worktree)
