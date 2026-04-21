# Browser Harness：自愈型 LLM 浏览器框架的设计哲学

> 来源：[browser-use/browser-harness](https://github.com/browser-use/browser-harness) · 2026-04-21

---

## 背景与问题

主流 LLM 浏览器自动化框架（Playwright、Selenium 包装层、browser-use 等）普遍存在"框架脆弱性"问题：开发者预定义一套工具函数，LLM 只能在这个固定集合中选择。一旦遇到框架未覆盖的场景（如特殊文件上传、验证码处理、动态 iframe），Agent 就会失败——因为它缺少合适的工具，却无法自己创造工具。

Browser Harness 提出了一个激进的解法：**让 LLM 在任务执行中途自己编写缺失的工具函数**。

---

## 核心方法：自愈型架构（Self-Healing Harness）

### 架构极简性
整个框架只有 ~592 行 Python：
- `run.py`（36 行）：入口，带预加载 helpers 运行纯 Python
- `helpers.py`（~195 行）：初始工具集，**Agent 可以直接编辑这个文件**
- `admin.py` + `daemon.py`（~361 行）：CDP WebSocket 桥接

直接建立在 **Chrome DevTools Protocol (CDP)** 上，没有任何中间层。一个 WebSocket 连接到 Chrome，没有其他依赖。

### 自愈机制的工作流

```
  ● Agent 需要上传文件
  │
  ● helpers.py → upload_file() 不存在
  │
  ● Agent 编辑 helpers.py，写入 upload_file()   192 → 199 行
  │
  ✓ 文件上传成功
```

关键洞察：Agent 不仅是工具的**使用者**，也是工具的**创造者**。当遇到障碍时，Agent 的第一反应不是报错，而是扩展自己的能力集。

### Domain Skills：沉淀可复用知识

框架内置了 `domain-skills/` 目录，包含 GitHub、LinkedIn、Amazon 等网站的技能文件。这些文件**由 Agent 自动生成**，而非手工编写：

> "Skills are written by the harness, not by you. Just run your task with the agent — when it figures something non-obvious out, it files the skill itself."

这形成了一个正向飞轮：
1. Agent 首次遇到某网站 → 摸索成功 → 自动写入 domain skill
2. 下次遇到同一网站 → 读取 skill → 直接执行，无需重新摸索

---

## 关键结论

1. **最小框架原则**：框架越薄，Agent 越自由。592 行比 5000 行的框架给 Agent 更大的能力空间。
2. **自修改代码是合法的**：让 Agent 编辑 `helpers.py` 看起来风险很高，但在隔离环境中这正是正确的设计——Agent 能力的边界应该由任务决定，而非由开发者预设。
3. **知识沉淀自动化**：与其靠人工维护工具文档，不如让 Agent 在实际执行中自动提炼 skill。这是一种经验驱动的知识库构建方式。

---

## 技术细节深挖

### CDP 直连 vs 框架包装
| 方式 | 延迟 | 能力上限 | 可调试性 |
|------|------|---------|---------|
| CDP 直连 | 最低 | Chrome 全能力 | 高（直接看 WebSocket 帧） |
| Playwright | 低 | 覆盖 90% 场景 | 中 |
| Selenium | 中 | 覆盖 80% 场景 | 低 |

CDP 直连的能力上限是 Chrome 的全部能力，包括：网络拦截、内存快照、性能追踪、Service Worker 控制等。

### 远程浏览器支持
框架支持连接 `cloud.browser-use.com` 提供的免费远程浏览器（3 个并发，无需信用卡），使 Sub-Agent 场景可行：主 Agent 编排，多个 Sub-Agent 各自操控一个浏览器实例。

---

## 横向对比

| 框架 | 工具可扩展性 | 自愈能力 | 代码量 | 适用场景 |
|------|------------|---------|------|---------|
| Browser Harness | **Agent 自写** | **有** | ~600 行 | 复杂/未知任务 |
| browser-use | 固定 + 插件 | 无 | ~5000 行 | 通用场景 |
| Playwright MCP | 固定 | 无 | 外部依赖 | 结构化任务 |
| Stagehand | 固定 | 无 | ~3000 行 | 企业场景 |

---

## 启发与思考

1. **"框架即约束"的反思**：大多数 Agent 框架在帮助开发者的同时，也在限制 Agent 的能力边界。Browser Harness 提出了一个问题：如果去掉框架，Agent 能走多远？
2. **自修改系统的风险管理**：在生产环境中，Agent 自动修改代码文件需要严格的沙箱隔离（独立进程、文件系统隔离）。Browser Harness 假设在受控开发环境中运行，这个前提需要在部署时认真审视。
3. **Domain Skills 的商业价值**：各网站的 skill 文件本质上是"如何操作这个网站"的结构化知识。这类知识库有潜在的商业价值，类似于 RPA 领域的机器人模板市场。

---

## 延伸阅读

- [Chrome DevTools Protocol 文档](https://chromedevtools.github.io/devtools-protocol/)
- [browser-use 苦涩教训博客](https://browser-use.com/posts/bitter-lesson-agent-frameworks)
- [Web Agents That Actually Learn](https://browser-use.com/posts/web-agents-that-actually-learn)
- [Stagehand: An Open and Easy to Use Framework for Web Agents](https://arxiv.org/abs/2501.09140)
