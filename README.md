# AI Systems Engineering Playbook

面向生产环境的 AI 系统工程实践知识库，记录从模型调用到 Agent 系统落地过程中遇到的真实工程问题、架构设计与实践方法。

> **Models are probabilistic; Engineering must be deterministic.**

模型输出具有概率性，但系统的边界、流程、权限、状态与质量保障必须尽可能确定、可验证、可观测。

## 这个仓库关注什么

这里不以罗列模型或框架用法为目标，而是从工程问题出发，讨论 AI 系统如何构建、演进并稳定运行。

主要关注方向：

- **Agent Engineering**：Agent 架构、Harness、任务执行、上下文管理与能力编排。
- **AI Coding Engineering**：AI 参与软件开发后的工程问题、质量风险与治理实践。
- **Context Engineering**：上下文构建、压缩、懒加载与信息边界。
- **Memory Engineering**：记忆写入、检索、更新、冲突处理与防止记忆污染。
- **RAG Engineering**：文档处理、分块策略、检索质量与评测。
- **AI Runtime & Infrastructure**：工具调用、流式输出、高并发 SSE、可靠性与可观测性。

## 内容导航

### Agent Engineering

围绕 Agent 能力如何组织、加载、执行和评估展开。

| 主题 | 内容 | 状态 |
| --- | --- | --- |
| Skill Engineering | 从 Prompt 膨胀到可维护的能力模块：理解 Skill 的价值、边界与引入时机 | 已发布 |

**当前文章**

- [Agent Skill 实战：从 Prompt 膨胀到可维护的能力模块](./01-agent-engineering/01-skill/01-why-skill.md)

### AI Coding Engineering

围绕 AI 如何可靠地参与软件工程交付展开。

主题导航：[AI Coding Engineering](./02-ai-coding/README.md)

| 主题 | 内容 | 状态 |
| --- | --- | --- |
| AI Coding Problems | AI Coding 常见问题、风险分类与治理要点 | 初稿 |
| AI Coding Verification | 测试保护、可信的验证结果与效果评估 | 初稿 |
| AI Coding Security | 生成代码安全、依赖治理、提示词注入防护、权限控制与审计 | 初稿 |
| AI Coding Runtime Reliability | 生产上下文左移、确定性验证、运行时防御与反馈闭环 | 初稿 |
| AI Coding Requirements | 澄清优先、结构化规约、代码考古、依赖血缘与红队评审 | 初稿 |
| AI Coding Workflow | 任务卡与范围控制、环境一致性、调试收敛、并行与重试、人工检查点 | 初稿 |

**当前文章**

- [AI Coding 常见问题与治理要点](./02-ai-coding/01-problems/01-ai-coding-common-problems.md)
- [AI Coding 验证体系：当开发过程变成黑盒](./02-ai-coding/02-verification/01-ai-coding-verification.md)
- [AI Coding 安全与合规：当 Agent 也成为攻击面](./02-ai-coding/03-security/01-ai-coding-security.md)
- [AI Coding 运行可靠性：AI 看不到的生产环境](./02-ai-coding/04-runtime/01-ai-coding-runtime-reliability.md)
- [AI Coding 需求工程：别让 AI 顺着错误的需求狂奔](./02-ai-coding/05-requirements/01-ai-coding-requirements.md)
- [AI Coding 工作流：给 Agent 的执行循环装上边界](./02-ai-coding/06-workflow/01-ai-coding-workflow.md)

## 后续规划

以下是计划持续补充的主题，不代表相关内容已经全部实现或验证：

- **Skill Engineering**：真实任务提取 Skill、Description 路由与评测、`SKILL.md` 结构、References / Scripts 的组织方式、安全边界与迭代机制。
- **Agent Architecture**：Agent Harness、任务状态管理、工具执行治理、多 Agent 协作与失败恢复。
- **AI Coding Engineering**：需求理解、代码复用、质量门禁、自动化验证与生产可靠性。
- **Context Engineering**：上下文预算、渐进式披露、压缩策略与上下文质量评估。
- **Memory Engineering**：记忆生命周期、冲突消解、过期清理与记忆质量评测。
- **RAG Engineering**：分块、索引、召回与重排，以及端到端评测。
- **Runtime Engineering**：流式响应、高并发连接管理、超时控制、背压与可观测性。

## Engineering Principles

- **Architecture before implementation**：先明确问题、边界与约束，再选择技术方案。
- **Explicit boundaries**：明确模块职责、数据边界、权限边界与失败边界。
- **Evaluation over intuition**：用可复现的评测验证效果，不用主观感受代替证据。
- **Reliability by design**：把超时、重试、幂等、降级与可观测性纳入设计。
- **Trade-offs over silver bullets**：说明方案的适用条件、成本与局限，不追求脱离场景的“最佳实践”。

## 适合谁阅读

适合有软件工程经验、希望构建可靠 AI 应用与 Agent 系统的开发者和架构师。内容默认读者具备一定的后端开发基础，重点讨论设计决策、工程权衡和生产实践。

## 关于本文档

这是一个持续演进的个人技术知识库。文章会尽量区分概念解释、个人实践、实验设计与已验证结论；尚未实测的内容会明确标注，不以推测代替实验结果。
