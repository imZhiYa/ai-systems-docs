# AI Coding Engineering

关注 AI 参与软件开发后的工程问题、质量风险与治理实践。

## 文档导航

| 文档 | 内容 | 状态 |
| --- | --- | --- |
| [AI Coding 常见问题与治理要点](./01-problems/01-ai-coding-common-problems.md) | AI Coding 常见问题分类、工程影响与治理要点 | 初稿 |
| [AI Coding 验证体系：当开发过程变成黑盒](./02-verification/01-ai-coding-verification.md) | 测试保护、覆盖设计、可信的验证结果与效果评估 | 初稿 |
| [AI Coding 安全与合规：当 Agent 也成为攻击面](./03-security/01-ai-coding-security.md) | 生成代码安全、依赖治理、提示词注入防护、权限控制与审计 | 初稿 |
| [AI Coding 运行可靠性：AI 看不到的生产环境](./04-runtime/01-ai-coding-runtime-reliability.md) | 生产上下文左移、确定性验证与仿真、运行时防御与渐进发布、生产反馈闭环 | 初稿 |
| [AI Coding 需求工程：别让 AI 顺着错误的需求狂奔](./05-requirements/01-ai-coding-requirements.md) | 澄清优先、结构化规约、代码考古与规则账本、依赖血缘、红队事前验尸 | 初稿 |
| [AI Coding 工作流：既不是新瀑布，也不放任 Agent](./06-workflow/01-ai-coding-workflow.md) | 分级路由、产物接力、隔离红队、测试与编码隔离、执行循环的边界、六阶段闭环 | 初稿 |
| [AI Coding 代码质量门禁：别让“看起来对”通过审查](./07-quality-gate/01-ai-coding-quality-gate.md) | 四层漏斗、防作弊门禁、架构与契约门禁、增量变异测试、语义审计、结构化报错与误报熔断 | 初稿 |
| [AI Coding 上下文工程：既不能忘，也不能塞满](./08-context/01-ai-coding-context.md) | 四层存储金字塔、按角色配餐与隔离、交接简报、保鲜与降噪、知识回流、生产上下文安全与度量 | 初稿 |
| [AI Coding 架构约束：让正确的路成为最省力的路](./09-architecture/01-ai-coding-architecture.md) | 模块地图与生成的依赖规则、平台原语与构建依赖隔离、抽象预算、废弃流程与适应度函数、架构演进 | 初稿 |

## 内容范围

本主题从真实软件工程交付出发，关注需求理解、上下文管理、代码生成、测试验证、工程流程、团队协作和生产运行。

问题清单和 8 个治理方向的实践文章都已完成初稿。各篇中的对照实验尚未实测，示例脚本和配置除特别注明外也未在真实项目中运行；后续会在真实项目中验证，并根据结果修订。

## 阅读顺序

想了解全貌，可以先读问题清单，再按流水线的顺序阅读：

| 顺序 | 文章 | 在流水线中的位置 |
| --- | --- | --- |
| 1 | [常见问题与治理要点](./01-problems/01-ai-coding-common-problems.md) | 全局地图：36 个问题、治理要点与 8 个治理方向 |
| 2 | [工作流](./06-workflow/01-ai-coding-workflow.md) | 总编排：六阶段流水线，把其余各篇串在一起 |
| 3 | [需求工程](./05-requirements/01-ai-coding-requirements.md) | 写代码之前：事实考古、规约编译、红队验尸 |
| 4 | [上下文工程](./08-context/01-ai-coding-context.md) | 每个节点：给对的角色准备对的上下文 |
| 5 | [架构约束](./09-architecture/01-ai-coding-architecture.md) | 编码时：模块边界、平台原语与架构演进 |
| 6 | [代码质量门禁](./07-quality-gate/01-ai-coding-quality-gate.md) | 合并前：四层漏斗与防作弊门禁 |
| 7 | [验证体系](./02-verification/01-ai-coding-verification.md) | 合并前：测试保护与可信的验证结果 |
| 8 | [安全与合规](./03-security/01-ai-coding-security.md) | 全程：Agent 的权限、依赖与审计 |
| 9 | [运行可靠性](./04-runtime/01-ai-coding-runtime-reliability.md) | 上线后：生产约束、渐进发布与反馈闭环 |

遇到具体问题时，也可以直接在问题清单中找到对应的问题，按“详见”链接跳转到相关小节。
