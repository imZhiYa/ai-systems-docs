# AI Coding 工作流：既不是新瀑布，也不放任 Agent

> 把需求阶段的四个顽疾（02、04、05、06）和生产运行的盲区连起来看，AI Coding 需要一条完整的工作流。但这条工作流不能设计成“全员重度审批的新瀑布”：改个文案也要走全套流程，研发一定会绕过它。也不能放任 Agent：一句话丢给 Agent，让它一直跑到结束，执行循环没有刹车。
>
> 本文的设计是一套**分级路由 + 产物驱动 + 上下文物理隔离 + 概率与确定性交错**的工程流水线：让工具查事实，让蓝队建规约，让隔离的红队找漏洞，让人做裁决，让确定性的 CI 和运行时不变量兜底，最后把每一次线上问题编译回系统规则。
>
> **适用范围与时效说明：** 本文对应 [AI Coding 常见问题与治理要点](../01-problems/01-ai-coding-common-problems.md) 中 Workflow Engineering 方向的 20、21、22、27、29；执行循环的权限边界（24 破坏性操作与权限越界）归入 Security & Compliance，见 [AI Coding 安全与合规](../03-security/01-ai-coding-security.md) 4.5 节。本文同时负责把前几篇的机制编排成一条流水线，各阶段的具体做法以对应文章为准。文中的机制属于工程实践建议，第 8 节是示意案例，第 9 节的实验尚未实测。各 Agent 工具对子代理、命令和非交互运行的支持不同，文中以 Claude Code 为例，以其当前文档为准。本文核对时间：2026 年 11 月。

## 1. TL;DR

| 问题 | 结论 |
| --- | --- |
| 核心问题 | AI Coding 的工作流有两种常见的失败：放任 Agent（执行循环没有刹车），或者做成新瀑布（所有需求都走全部关卡，最后被绕过） |
| 四条铁律 | 风险分级路由；产物接力，严禁对话继承；三明治架构：每个概率性的 Agent 节点后面紧跟一个确定性的卡点；人只站在高杠杆的裁决点 |
| 六阶段闭环 | Phase 0 定级路由 → 1 事实考古 → 2 规约编译 → 3 隔离红队 → 4 测试先行与受限编码 → 5 渐进发布 → 6 生产免疫闭环 |
| 执行循环的边界 | Phase 4 中，Agent 在任务卡的范围、与 CI 一致的环境、尝试预算和重试规则之内工作，越界或预算耗尽就停下来交给人 |
| 落地方式 | V1 命令与模板（参考 1 周）→ V2 CI 强绑定（参考 1 个月）→ V3 运行时联防与免疫闭环（参考 1 个季度） |
| 主要成本 | 路由规则的误判、产物变成走过场的文档、隔离与异构模型的成本、人工卡点成为瓶颈 |
| 不要做 | 所有需求一刀切走全流程；Agent 之间传递聊天记录；让 AI 直接校验 AI 的最终产出；让人逐行审查 AI 生成的上千行代码 |

---

## 2. 问题：两种失败的工作流

### 2.1 放任：执行循环没有刹车

AI 写代码不是一问一答，而是一个循环：计划、执行、看结果、再执行，可以连续跑几十步。人写代码时，这个循环自带刹车；Agent 没有：

| | 人写代码 | Agent 写代码 |
| --- | --- | --- |
| 范围 | 要改不相关的代码时会犹豫 | 看到可以优化的地方就顺手改了 |
| 环境 | 环境不对时，先把环境修好 | 在错误的环境里反复失败，把环境问题当成代码问题修 |
| 尝试次数 | 同一个 Bug 试了几次，会停下来重新想 | 一直换写法尝试，直到上下文用完 |
| 时间 | 会感到“这件事花太久了” | 没有时间感，等待和重试都不计成本 |
| 理解 | 改动是自己想出来的，自然理解 | 人只看到结果，越来越习惯不看细节就批准 |

每一步单看都合理，合在一起就失控了。这部分问题在第 6 节处理。

放任的另一个后果在需求阶段：模糊的需求漏掉了潜规则和下游联动，AI 再顺着人的话把漏洞圆过去，最后以很快的速度把一个错误的系统开发上线（见 [AI Coding 需求工程](../05-requirements/01-ai-coding-requirements.md) 第 2 节）。

### 2.2 新瀑布：所有需求都走全部关卡

另一个极端，是把前几篇的所有机制都变成每个需求的必经关卡：改一行文案也要做代码考古、写决策表、跑红队验尸、等两次签字。这样的流程运行不了几天就会被绕过：研发会把需求拆小、把改动藏进别的 PR，或者干脆不再走流程。

被绕过的流程比没有流程更危险：团队以为有保障，实际上没有。

### 2.3 这条工作流要做到什么

| 要求 | 含义 |
| --- | --- |
| 分级 | 关卡的轻重由改动的爆炸半径决定，大多数低风险需求走快车道 |
| 产物驱动 | 阶段之间只通过结构化文件交接，过程可追溯、可审计 |
| 上下文隔离 | 每个角色在全新的会话中启动，不继承前面的对话，避免迎合惯性和上下文污染 |
| 概率与确定性交错 | AI 负责生成和推理，确定性工具负责采集事实和拦截错误，最终的放行不由 AI 决定 |

## 3. 四条底层铁律

### 3.1 风险分级路由，拒绝一刀切

按改动触碰的爆炸半径，自动路由到 L1（快车道）、L2（标准道）或 L3（高危装甲道）。定级由脚本完成，不靠人判断；人可以把等级调高，但调低必须签字说明理由（见 5.1 节）。

### 3.2 产物接力，严禁对话继承

Agent 之间不传递聊天记录。所有交接只通过 Git 中的结构化文件完成，每个角色在全新的、隔离的会话中启动，只读取它需要的文件。这样可以避免上一个会话中的讨论倾向，影响下一个角色的判断（问题 06 的迎合惯性），也让每一步的输入和输出都留在仓库里（见 5.2 节）。

### 3.3 三明治架构：概率生成 × 确定性卡点

不让 AI 直接校验 AI 的最终产出。每一个概率性的 Agent 节点后面，都紧跟一个确定性的工程工具做硬拦截：

| Agent 节点 | 产出 | 紧跟的确定性卡点 |
| --- | --- | --- |
| 考古 Agent（Phase 1） | As-Is 规则清单 | 脚本核对每条规则引用的文件和行号确实存在 |
| Spec Agent（Phase 2） | 决策表、状态机、BDD | 检查决策表中没有“待定”、BDD 可以被解析 |
| 红队 Agent（Phase 3） | 红队审查报告 | 检查每条问题都有失败场景、每条都有人的决策 |
| Test Agent（Phase 4） | 验收测试 | 测试能编译运行，并且在实现之前是失败的 |
| Code Agent（Phase 4） | 实现代码 | CI 门禁：测试、并发注入、架构规则、执行计划、契约测试 |
| 复盘 Agent（Phase 6） | 更新规则的 PR | 规则本身经过 CI 和人工审查才能合并 |

红队也是 AI，但它的作用是**找问题**，不是**放行**。放行只由确定性工具和人决定。

### 3.4 人只站在高杠杆的裁决点

不要让人去逐行审查 AI 生成的上千行胶水代码，而是把人的精力集中在价值最高的地方：

1. **写代码之前**：规约与冲突裁决（卡点 A）。
2. **写代码之前**：红队质疑的裁决与豁免签字（卡点 B）。

编码和验证主要交给 Agent 和确定性 CI，只在越界、预算耗尽或门禁反复不通过时才找人（见 5.4 节）。

按问题清单第 3 节的约束力规则，说明文件和提示词只能降低问题发生的概率。本文的每一个关卡，都配有 CI 检查、权限限制或人工确认作为强制手段。

## 4. 全景架构：六阶段闭环

```text
[原始需求 / Issue]
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 0：自动定级路由（确定性）                                │
│ 按涉及的模块、路径、表和平台层，路由到 L1 / L2 / L3            │
└───────┬───────────────────────────┬───────────────────────────┘
        │ L1：跳过 Phase 1 和 3       │ L2 / L3：进入完整流水线
        │                           ▼
        │   ┌───────────────────────────────────────────────────┐
        │   │ Phase 1：事实考古与建图（治 04、05、生产盲区）      │
        │   │ 工具先采集事实，考古 Agent 再翻译成规则             │
        │   │ 产出：01-context-truth.md                           │
        │   └───────────────────────┬───────────────────────────┘
        │                           ▼
        │   ┌───────────────────────────────────────────────────┐
        │   │ Phase 2：歧义收口与规约编译（治 02）                │
        │   │ 先产出 02-clarifications.md，禁止写方案             │
        │   │ ★ 卡点 A：人回答歧义、裁决新旧规则冲突              │
        │   │ 编译出 03-spec.md：决策表、状态机、反例、三阶影响   │
        │   └───────────────────────┬───────────────────────────┘
        │                           ▼
        │   ┌───────────────────────────────────────────────────┐
        │   │ Phase 3：物理隔离的红队验尸（治 06，复核 04、05）   │
        │   │ 全新会话，不带历史对话；建议换不同的模型家族        │
        │   │ 业务事前验尸 + 并发、缓存、切面等多因素攻击         │
        │   │ 产出：04-redteam-audit.md                           │
        │   │ ★ 卡点 B：人逐条裁决——修改 Spec，或签字豁免        │
        │   └───────────────────────┬───────────────────────────┘
        ▼                           ▼  Spec 冻结
┌───────────────────────────────────────────────────────────────┐
│ Phase 4：测试先行 + 受限编码 + 确定性门禁                      │
│ 4a. Test Agent 只依据 03-spec 生成 BDD、模糊测试、并发时序测试 │
│ 4b. Code Agent 在任务卡的边界内编码，使用平台提供的显式原语    │
│ 4c. CI 门禁：测试、并发注入、架构规则、执行计划、契约与流量对比│
│     不通过则回到 4b 修复；同一错误超过预算，停下来交给人       │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 5：渐进发布与双信号监控                                  │
│ 功能开关 + 金丝雀发布；回滚信号 = 黄金指标 + 正确性探针        │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 6：生产免疫闭环                                          │
│ 线上故障和漏网缺陷 → 复盘 Agent 提 PR，更新：                  │
│ ① 领域规则账本 ② 架构禁忌 ③ 回归用例 ④ 生产画像               │
└───────────────────────────────────────────────────────────────┘
```

| 阶段 | 确定性工具做什么 | Agent 做什么 | 产物 | 人 | 详见 |
| --- | --- | --- | --- | --- | --- |
| 0 定级路由 | 按规则自动定级 | — | `00-raw-prd.md` 中的等级 | 可以调高；调低需签字 | 本文 5.1 |
| 1 事实考古 | 采集依赖树、单测、表结构与生产统计、真实下游、领域规则 | 考古 Agent 把代码逻辑翻译成规则清单，标注行号 | `01-context-truth.md` | — | 需求工程 4.2、4.3 |
| 2 规约编译 | 检查决策表无“待定”、BDD 可解析 | Spec Agent 抽取歧义，编译规约 | `02-clarifications.md`、`03-spec.md` | 卡点 A | 需求工程 4.1 |
| 3 隔离红队 | 检查每条问题有失败场景和决策 | 红队 Agent，L3 分业务和架构两个视角 | `04-redteam-audit.md` | 卡点 B | 本文 5.3、需求工程 4.4 |
| 4 测试与编码 | CI 门禁，生成验证结果 | Test Agent 与 Code Agent 隔离工作 | 代码、测试、`05-verification.json` | 越界或预算耗尽时介入；合并前能解释改动 | 本文 5.3、第 6 节；[代码质量门禁](../07-quality-gate/01-ai-coding-quality-gate.md) |
| 5 渐进发布 | 金丝雀分析、自动回滚 | — | — | 异常时处理 | 运行可靠性 4.3 |
| 6 免疫闭环 | 新规则经过 CI | 复盘 Agent 提 PR | 规则、用例、画像的更新 | 审批 PR | 运行可靠性 4.4 |

## 5. 关键设计

### 5.1 Phase 0：分级路由，防止官僚化的核心

**分级表：**

| 等级 | 触发条件（满足任一即升级） | 启用的关卡 | 人的介入点 |
| --- | --- | --- | --- |
| **L1 轻量级** | 纯展示、只读查询、文案与配置、独立的无状态工具函数 | 跳过 Phase 1 和 3：简版 Spec → 编码 → 标准 CI（Lint、单测） | 只需最终 PR 的快速确认 |
| **L2 标准业务** | 常规的业务迭代、接口字段变更、跨模块调用 | 完整执行 Phase 1–5：代码考古、决策表 Spec、业务红队、执行计划门禁、灰度发布 | 卡点 A + 卡点 B |
| **L3 核心高危** | 触碰资金、库存、权限、主状态机；修改并发、缓存、切面或平台层原语；核心大表变更 | L2 全部关卡，再加：双视角红队（业务 + 架构混沌）、强制的时序延迟并发测试、流量回放、旁路实时对账、架构师签字 | 卡点 A + 卡点 B + 架构师核验平台层边界 |

**定级规则写成配置：**

```yaml
# .specs/routing-rules.yaml
# 从 L3 往下匹配，满足任一条件即定为该等级；都不满足为 L1。
# 规则宁可偏严：误升级的成本，远小于漏升级。
L3:
  modules: [payment, inventory, auth, order-state, platform]
  paths:
    - src/payment/*
    - src/inventory/*
    - src/auth/*
    - src/order/state/*
    - src/platform/*                  # 平台层：事务、锁、缓存、并发、切面等基础设施
  added_line_patterns:                # 业务代码出现这些写法，说明触碰了并发、缓存或切面
    - '\bThreadLocal\b'
    - '@Aspect\b'
    - '\bsynchronized\b'
    - '(?i)ALTER\s+TABLE\s+(orders|payments|accounts)\b'
L2:
  modules: [order, logistics, tax, invoice, user]
  paths:
    - src/*
    - db/migration/*
    - api/*                           # 接口契约
required_artifacts:
  L1: [00-raw-prd.md, 03-spec.md]
  L2: [00-raw-prd.md, 01-context-truth.md, 02-clarifications.md, 03-spec.md, 04-redteam-audit.md]
  L3: [00-raw-prd.md, 01-context-truth.md, 02-clarifications.md, 03-spec.md, 04-redteam-audit.md]
```

**定级要做两次：**

- **需求阶段初判**：按 `00-raw-prd.md` 中声明的模块定级，决定后面走哪些关卡。
- **PR 阶段复核**：按实际改动的路径和新增内容重新定级。实际改动的等级高于声明的等级时，阻止合并：要么升级并补齐产物，要么由负责人签字豁免。

只做第一次定级，低估等级的需求就会一路走快车道；第二次定级保证“定级低了”会被发现。下面的脚本完成这两件事，并检查对应等级的产物是否齐全：

```python
#!/usr/bin/env python3
"""Phase 0 风险定级：按需求涉及的模块和实际改动自动定级，并检查 .specs 产物。

用法：
  需求阶段（只看 00-raw-prd.md 中声明的模块，输出建议等级）：
    python triage.py .specs/routing-rules.yaml .specs/FEAT-2048-modify-order-address < /dev/null
  PR 阶段（同时看实际改动，并检查产物）：
    git diff -U0 origin/main...HEAD \
      | python triage.py .specs/routing-rules.yaml .specs/FEAT-2048-modify-order-address --check
依赖：PyYAML。路径按 fnmatch 匹配，* 也匹配 /。
"""
import fnmatch
import pathlib
import re
import sys

import yaml

LEVELS = ["L1", "L2", "L3"]


def higher(a, b):
    return a if LEVELS.index(a) >= LEVELS.index(b) else b


def parse_diff(text):
    """把 git diff 输出解析为 {文件路径: [新增的行]}。"""
    files, current = {}, None
    for line in text.splitlines():
        if line.startswith("--- "):
            old = line[4:].strip()
            current = old[2:] if old.startswith("a/") else None
        elif line.startswith("+++ "):
            new = line[4:].strip()
            if new.startswith("b/"):
                current = new[2:]
            if current is not None:
                files.setdefault(current, [])
        elif line.startswith("+") and current is not None:
            files[current].append(line[1:])
    return files


def classify(rules, files, modules):
    """返回 (等级, 命中原因)。从 L3 往下检查，命中即返回。"""
    for level in ("L3", "L2"):
        rule = rules.get(level, {})
        patterns = [re.compile(p) for p in rule.get("added_line_patterns", [])]
        reasons = [f"模块 {m}" for m in modules if m in rule.get("modules", [])]
        for path, added in files.items():
            if any(fnmatch.fnmatch(path, p) for p in rule.get("paths", [])):
                reasons.append(f"路径 {path}")
            for p in patterns:
                if any(p.search(line) for line in added):
                    reasons.append(f"{path} 新增内容匹配 {p.pattern}")
        if reasons:
            return level, reasons
    return "L1", []


def front_matter(path):
    text = path.read_text(encoding="utf-8") if path.exists() else ""
    if not text.startswith("---"):
        return {}
    end = text.find("\n---", 3)
    if end == -1:
        return {}
    return yaml.safe_load(text[3:end]) or {}


def check_specs(rules, spec_dir, computed):
    problems, notes = [], []
    prd = front_matter(spec_dir / "00-raw-prd.md")
    declared = prd.get("risk_level")
    if declared not in LEVELS:
        return ["00-raw-prd.md 缺少有效的 risk_level（L1 / L2 / L3）"], notes
    effective = higher(declared, computed)
    if effective != declared:
        if prd.get("risk_override_reason") and prd.get("risk_override_by"):
            effective = declared
            notes.append(f"声明 {declared} 低于自动定级 {computed}，已由 {prd['risk_override_by']} 签字豁免")
        else:
            problems.append(f"声明的等级 {declared} 低于自动定级 {computed}：请升级，或填写豁免理由和签字人")
    for name in rules.get("required_artifacts", {}).get(effective, []):
        if not (spec_dir / name).exists():
            problems.append(f"{effective} 需要产物 {name}，但目录中没有")
    if effective in ("L2", "L3"):
        if front_matter(spec_dir / "03-spec.md").get("status") != "frozen":
            problems.append("03-spec.md 的 status 不是 frozen：Spec 冻结后才能合并")
        audit = spec_dir / "04-redteam-audit.md"
        if audit.exists() and "待裁决" in audit.read_text(encoding="utf-8"):
            problems.append("04-redteam-audit.md 中仍有“待裁决”的问题")
    return problems, notes


def main():
    args = [a for a in sys.argv[1:] if not a.startswith("--")]
    with open(args[0], encoding="utf-8") as f:
        rules = yaml.safe_load(f)
    spec_dir = pathlib.Path(args[1]) if len(args) > 1 else None
    modules = front_matter(spec_dir / "00-raw-prd.md").get("modules", []) if spec_dir else []
    level, reasons = classify(rules, parse_diff(sys.stdin.read()), modules)
    print(f"自动定级：{level}")
    for r in reasons:
        print(f"  - {r}")
    if spec_dir is None or "--check" not in sys.argv:
        return
    problems, notes = check_specs(rules, spec_dir, level)
    for n in notes:
        print(f"[提示] {n}")
    for p in problems:
        print(f"[阻止] {p}")
    sys.exit(1 if problems else 0)


if __name__ == "__main__":
    main()
```

路径和文本匹配只能识别一部分风险：通过配置改变行为、通过间接调用触碰核心逻辑，都可能被漏判。所以规则宁可偏严，并在 Phase 6 中根据漏判的案例持续补充。

### 5.2 产物契约：一切落盘到 Git（Spec-as-Code）

不要让需求分析过程散落在各个工具的临时聊天框里。每个需求在仓库中创建一个标准目录，随 PR 一起归档，成为团队长期可查的记录：

```text
.specs/FEAT-2048-modify-order-address/
├── 00-raw-prd.md            # 原始需求，含风险等级和涉及模块
├── 01-context-truth.md      # [Phase 1] As-Is 规则、真实下游、生产表量级
├── 02-clarifications.md     # [Phase 2a] 歧义、隐含假设、新旧规则冲突（人勾选）
├── 03-spec.md               # [Phase 2b] 冻结的可执行规约：决策表、状态机、BDD、三阶影响
├── 04-redteam-audit.md      # [Phase 3] 红队审查报告，以及人的逐条裁决和豁免记录
└── 05-verification.json     # [Phase 4] CI 门禁结果，由 CI 生成
```

`00-raw-prd.md` 的文件头记录等级和模块，供 5.1 的脚本读取：

```markdown
---
feature_id: FEAT-2048
title: 已支付订单修改收货地址
risk_level: L3                 # Phase 0 初判；人可以调高，调低必须填写下面两项
modules: [order-state, logistics, tax, invoice]
# risk_override_reason: ""
# risk_override_by: ""
---
```

`04-redteam-audit.md` 中，红队写下的每条问题，决策列一律先填“待裁决”，由人在卡点 B 逐条改写。5.1 的脚本在仍有“待裁决”时阻止合并：

```markdown
| 编号 | 视角 | 问题 | 失败场景 | 决策 | 理由 | 签字 |
| --- | --- | --- | --- | --- | --- | --- |
| R1 | 业务 | 改地址后没有作废原电子发票 | 已开票订单改到其他税区，发票税额与实际不符 | 采纳修改 | — | 张三 |
| R2 | 架构 | “取消”和“改地址”并发时存在竞态 | 两个请求间隔极短时，缓存中的订单状态被旧值覆盖 | 待裁决 | | |
```

`05-verification.json` 由 CI 生成，不由 Agent 编写，作为构建产物归档，或由 CI 机器人提交到 PR 分支：

```json
{
  "feature_id": "FEAT-2048",
  "commit": "<提交 SHA>",
  "risk_level": "L3",
  "checks": [
    {"name": "spec-bdd", "status": "passed", "passed": 18, "failed": 0},
    {"name": "concurrency-delay-injection", "status": "passed"},
    {"name": "archunit", "status": "passed"},
    {"name": "explain-gate", "status": "passed", "max_estimated_rows": 1200},
    {"name": "contract-tests", "status": "passed"}
  ]
}
```

**为什么要落盘到 Git：**

- **天然实现上下文隔离。** Phase 3 的红队由编排器以非交互方式启动，只读取 01 和 03 两个文件，不受之前任何讨论的影响。
- **可审计、可追溯。** 三个月后线上出了问题，打开 `04-redteam-audit.md`，就能知道当时是红队没有发现，还是人签字豁免了这个风险，再把结论交给 Phase 6。
- **产物本身可以被检查。** 文件有固定的结构，CI 可以检查是否齐全、是否冻结、是否还有待裁决的问题，而聊天记录做不到。

### 5.3 确定性工具与 Agent 的分工

很多团队做多 Agent 失败，是因为把一切都交给模型。正确的分工是：**确定性工具采集硬事实，模型做推理与对抗。**

**Phase 1：先跑工具，再跑考古 Agent。**

1. 确定性脚本先通过只读接口采集事实：
   - 目标代码的依赖树和现有单测。
   - 数据库表结构和生产统计快照：行数、索引、QPS（见运行可靠性 4.1 节的生产画像）。
   - MQ 和接口的真实下游订阅者清单。
   - 匹配目录下的 `docs/domain-rules/*.yaml`。
2. 考古 Agent 只负责把代码中的分支判断和校验逻辑，翻译成人能读懂的 As-Is 规则清单，每条标注精确的文件和行号；找不到出处的规则不写。行号由脚本核对，防止编造。

**Phase 3：双视角红队与异构模型。**

L3 需求把红队拆成两个视角，分别在独立的会话中运行：

| 视角 | 关注 | 典型问题 |
| --- | --- | --- |
| 业务红队 | 事前验尸：04 隐式规则冲突、05 三阶因果链 | 财务、税务、报表、风控、逆向退款是否被遗漏 |
| 架构混沌红队 | 多个因素叠加时才出现的问题 | 并发竞态、锁与事务的顺序、缓存脏读、线程上下文串租户、历史脏数据、超时重试风暴 |

单个因素都没有问题，多个因素叠加才出错，这类问题最难在单个视角的审查中发现，所以单独设一个视角。红队的指令和缺陷配额见需求工程 4.4 节。

如果 Phase 2（写 Spec）和 Phase 4（写代码）使用模型 A，可以让 Phase 3 的红队刻意换成另一个模型家族 B，减少同源模型共有的盲区。这是一个合理的假设，能否显著降低漏检，需要用第 9 节的实验验证；把数据发给另一家模型服务前，还要经过安全与合规 4.6 节的模型访问网关和审批。

**Phase 4：Test Agent 与 Code Agent 隔离。**

AI 为自己的代码写测试，容易出现自证偏见（见验证体系第 2 节）。所以 Test Agent 和 Code Agent 也要隔离：

- Test Agent 只能看到 `03-spec.md` 和公开的接口定义，看不到实现代码，据此生成验收测试、模糊测试和并发时序测试。这些测试在实现之前应该是失败的。
- Code Agent 不能修改 Test Agent 生成的测试：任务卡中把验收测试目录放进 `forbidden_paths`，由 6.1 节的范围检查强制执行，再配合验证体系 4.1 节的测试保护。

**Phase 4：业务代码使用平台层的显式原语。**

把代码分成两层：**平台层**（事务、锁、缓存、并发、切面、线程上下文等基础设施）和**业务层**。Code Agent 只在业务层编码，需要锁、幂等、事务性发件箱时，使用平台层提供的显式原语，而不是自己写 `synchronized`、`ThreadLocal` 或切面。平台层的修改按 L3 处理。

这条边界由架构测试在 CI 中强制执行，以 ArchUnit 为例：

```java
@AnalyzeClasses(packages = "com.example.order")
class ArchitectureTaboosTest {

    // 线程上下文由平台层统一管理，业务代码不得直接使用 ThreadLocal
    @ArchTest
    static final ArchRule no_thread_local_in_business =
        noClasses().that().resideInAPackage("..order..")
            .should().dependOnClassesThat().areAssignableTo(ThreadLocal.class)
            .because("使用平台提供的 RequestContext，避免线程复用时串租户");

    // 消息消费者必须继承平台提供的幂等基类
    @ArchTest
    static final ArchRule consumers_must_be_idempotent =
        classes().that().haveSimpleNameEndingWith("Consumer")
            .should().beAssignableTo(IdempotentConsumer.class);
}
```

“事务内禁止外部调用”“锁必须在事务之外获取”这类规则，用依赖关系很难表达，可以用静态分析或自定义 AST 规则实现。

### 5.4 人在工作流中的位置（问题 29）

好的工作流不会让工程师更累，而是改变工程师花时间的方式：

| | 以前 | 现在 |
| --- | --- | --- |
| 写代码前 | 对着模糊的需求直接开写 | 卡点 A：在 `02-clarifications.md` 中，回答已经整理成选择题的歧义和旧规则冲突 |
| 写代码前 | 没有人挑毛病 | 卡点 B：在 `04-redteam-audit.md` 中，对独立红队提出的问题逐条决定修改或签字豁免 |
| 写代码时 | AI 一次生成上千行代码 | Agent 在任务卡的边界内工作，CI 自动把关；越界或预算耗尽才找人 |
| 审查时 | 看花了眼，点批准 | 风险已在写代码前裁决；审查聚焦设计取舍和业务语义，作者能解释核心逻辑 |
| 上线后 | 半夜救火 | 金丝雀发布，黄金指标和正确性探针触发自动回滚 |

卡点 A 和卡点 B 的设计目标，是每次都能在一次短会内完成；具体时长需要在实践中校准。

除了两个卡点，工作流中还有几个必须由人参与的地方：

| 检查点 | 时机 | 人要做什么 |
| --- | --- | --- |
| 调整定级 | Phase 0 | 认为风险更高时调高；调低必须签字说明理由 |
| 处理越界 | Phase 4，Agent 需要修改范围外的文件时 | 决定是否扩大范围，并更新任务卡 |
| 处理停止汇报 | Phase 4，同一错误超过尝试预算时 | 基于调试记录判断下一步，或亲自接手 |
| 审查与解释 | 合并前 | 理解改动的核心逻辑，能向审查者解释 |
| 架构师核验 | L3 需求 | 核验平台层的边界没有被突破 |

**合并前，作者必须能解释改动。** PR 的作者是发起任务的人，不是 Agent。在 PR 模板中加入：

```markdown
## 作者说明（由作者本人填写，不要让 AI 代写）

- 这次改动的核心逻辑（一两句话）：
- 为什么这样实现，考虑过哪些其他做法：
- 最可能出错的地方，以及如何验证过：
```

CI 只能检查这一节是否填写，真正的检查发生在审查中：审查者可以就其中任何一点向作者提问，作者需要不借助 AI 回答。

29 开发者技能退化，主要靠团队的培养和评审文化来解决。工作流能做的，是保证人留在关键环节：核心模块的设计决策和线上排障由开发者主导，团队可以定期组织不借助 AI 的代码阅读和排障复盘。

## 6. Phase 4 内部：给执行循环装上边界

Phase 4 是 Agent 连续执行几十步的地方，也是最容易失控的地方。本节给这个执行循环装上四种边界；权限的边界（问题 24）见安全与合规 4.5 节。

### 6.1 任务卡与范围控制（问题 20）

**机制一：任务卡。**

Spec 冻结后，为 Code Agent 生成一张任务卡，由人确认：

```yaml
# .agent/tasks/FEAT-2048.yaml
task_id: FEAT-2048
spec: .specs/FEAT-2048-modify-order-address/03-spec.md
goal: 支持已支付、未发货的订单修改收货地址

allowed_paths:                            # 允许修改的路径，按 fnmatch 匹配，* 也匹配 /
  - src/order/address/*
  - src/order/api/AddressController.java
  - tests/order/address/*                 # Code Agent 自己的单元测试
  - .agent/tasks/FEAT-2048.yaml
forbidden_paths:                          # 即使在 allowed_paths 内也不允许修改
  - tests/spec-bdd/FEAT-2048/*            # Test Agent 依据 Spec 生成的规约测试
  - src/order/address/legacy/*

environment: .devcontainer/devcontainer.json

budget:
  max_changed_files: 15
  max_changed_lines: 600
  max_fix_attempts: 3                     # 同一个错误最多修复 3 次，见 6.3
  max_duration_minutes: 60

done_when:
  - 03-spec 中的 BDD 用例和并发时序测试全部通过
  - 按验证体系 4.3 节的格式输出验证报告

stop_when:
  - 需要修改 allowed_paths 以外的文件
  - 同一个错误修复 3 次仍未解决
  - 发现 Spec 与现有代码冲突
```

`stop_when` 是任务卡中最重要的部分：它告诉 Agent 什么时候必须停下来，而不是自己想办法绕过去。

**机制二：CI 检查改动是否超出任务卡。**

```python
#!/usr/bin/env python3
"""检查改动是否超出任务卡声明的范围和预算。

用法：git diff --numstat --no-renames origin/main...HEAD \
        | python check_scope.py .agent/tasks/FEAT-2048.yaml
依赖：PyYAML。
路径按 fnmatch 匹配，* 也匹配 /。二进制文件的行数记为 0。
"""
import fnmatch
import sys

import yaml


def matches(path, patterns):
    return any(fnmatch.fnmatch(path, p) for p in patterns or [])


def check(card, numstat_lines):
    allowed = card.get("allowed_paths", [])
    forbidden = card.get("forbidden_paths", [])
    budget = card.get("budget", {})
    problems, files, lines = [], 0, 0
    for row in numstat_lines:
        if not row.strip():
            continue
        added, deleted, path = row.split("\t", 2)
        files += 1
        lines += (int(added) if added.isdigit() else 0) + (int(deleted) if deleted.isdigit() else 0)
        if matches(path, forbidden):
            problems.append(f"[超出范围] {path}：在 forbidden_paths 中")
        elif not matches(path, allowed):
            problems.append(f"[超出范围] {path}：不在 allowed_paths 中")
    if files > budget.get("max_changed_files", float("inf")):
        problems.append(f"[超出预算] 改动 {files} 个文件，上限 {budget['max_changed_files']}")
    if lines > budget.get("max_changed_lines", float("inf")):
        problems.append(f"[超出预算] 改动 {lines} 行，上限 {budget['max_changed_lines']}")
    return problems


def main():
    with open(sys.argv[1], encoding="utf-8") as f:
        card = yaml.safe_load(f)
    problems = check(card, sys.stdin.read().splitlines())
    for p in problems:
        print(p)
    if problems:
        print("请缩小改动，或在 PR 中说明原因并更新任务卡，由人确认后放行。")
    sys.exit(1 if problems else 0)


if __name__ == "__main__":
    main()
```

这个脚本检查的是 Code Agent 的改动，`.specs/` 目录下的产物和 Test Agent 生成的测试，应该由各自的阶段提交，不和实现代码混在同一次检查里。超出范围不一定是错误，但扩大范围必须成为一个经过确认的决定：更新任务卡并说明原因，任务卡本身的修改在 PR 中单独审查。

**机制三：重构和格式化单独提交。** 发现值得重构的地方，记录下来作为单独的任务，不在当前任务中顺手修改。

### 6.2 环境一致性（问题 21）

Agent 和 CI 使用同一个开发容器。用 [Dev Container 规范](https://containers.dev/) 的 `devcontainer.json` 描述开发环境：

```jsonc
// .devcontainer/devcontainer.json
{
  "name": "order-service",
  // 团队自己构建并发布的开发镜像，CI 使用同一个镜像；建议固定到具体版本或摘要
  "image": "registry.example.com/order-service/devenv:2026-11-01",
  "containerEnv": {
    "TZ": "Asia/Shanghai"
  },
  "postCreateCommand": "./gradlew dependencies --no-daemon"
}
```

CI 中使用同一个镜像运行测试，例如在 GitHub Actions 的作业中设置 `container.image` 为同一个镜像，或者用 Dev Container CLI 的 `devcontainer up --workspace-folder .` 启动同一份配置。镜像版本出现在两个地方时，升级要一起改，可以在 CI 中比对两处的版本是否一致。

此外：

- 运行时版本、构建和测试命令写进 `AGENTS.md`；任务开始时，Agent 先做一次环境自检，再开始修改代码。
- 构建配置和依赖文件默认不放进任务卡的 `allowed_paths`。Agent 遇到构建失败时，常见的“修复”是改构建配置或升级依赖，这类改动需要停下来由人确认。新增依赖的检查见安全与合规 4.3 节。

### 6.3 调试循环收敛（问题 22）

Phase 4c 的门禁不通过时，Code Agent 回到 4b 修复。这个循环必须收敛：

**机制一：先复现，再修改。** 修改代码之前，先给出复现问题的命令和输出；无法复现时停下来汇报。复现用例之后保留为回归测试。

**机制二：假设记录。** 每次修改只验证一个假设，并记录下来：

```markdown
## 调试记录：并发时序测试失败

| 次数 | 假设 | 验证方法 | 结果 |
| --- | --- | --- | --- |
| 1 | 改地址时没有校验订单状态 | 在修改前读取并校验状态 | 否定：串行时正确，并发时仍然失败 |
| 2 | 缓存和数据库的更新顺序有问题 | 改为先更新数据库、再删除缓存 | 否定：失败率下降，但仍然存在 |
| 3 | “取消”和“改地址”之间缺少互斥 | 在测试中注入延迟，观察两个请求的执行顺序 | 确认：两个请求都通过了状态校验 |
```

**机制三：尝试预算和停止汇报。** 同一个错误修复超过预算次数仍未解决时，停止修改，按固定格式汇报：

```markdown
## 停止汇报

- 问题：并发时序测试失败
- 已尝试：3 次，见调试记录
- 已排除：状态校验缺失、缓存更新顺序
- 当前判断：需要对同一订单的“取消”和“改地址”加互斥
- 需要人决定：使用平台层的订单锁原语，还是改为在数据库中用状态条件更新
- 当前代码状态：已回退到最后一次通过测试的提交
```

**机制四：检测同一个错误反复出现。** Agent 每次换一种写法，报错信息看起来不同，本质可能是同一个错误。对报错信息归一化后再计数：

```python
import re
from collections import Counter

_VOLATILE = [
    (re.compile(r"0x[0-9a-fA-F]+"), "<addr>"),
    (re.compile(r"\d{4}-\d{2}-\d{2}[T ][\d:.]+"), "<time>"),
    (re.compile(r":\d+(:\d+)?"), ":<line>"),
    (re.compile(r"\b\d+\b"), "<n>"),
]


def error_signature(message: str) -> str:
    """取报错的第一行并归一化，作为判断“是否同一个错误”的依据。"""
    first_line = message.strip().splitlines()[0] if message.strip() else ""
    for pattern, repl in _VOLATILE:
        first_line = pattern.sub(repl, first_line)
    return first_line


def should_stop(errors: list[str], max_attempts: int = 3) -> bool:
    """同一个错误签名出现次数超过预算时返回 True。"""
    counts = Counter(error_signature(e) for e in errors)
    return any(n > max_attempts for n in counts.values())
```

如果所用的 Agent 工具支持在工具调用前后运行钩子，可以把这段逻辑接进去，超过预算时阻止继续修改。归一化规则比较粗糙，需要按项目的报错格式调整。

### 6.4 等待与重试成本（问题 27）

**机制一：独立任务并行执行。** 用 `git worktree` 为每个任务创建独立的工作目录：

```bash
git worktree add -b agent/feat-2048 ../wt-feat-2048 origin/main
git worktree add -b agent/feat-2051 ../wt-feat-2051 origin/main
# 在各自的目录中运行 Agent ……
git worktree remove ../wt-feat-2048
```

worktree 不共享构建产物、端口和本地数据库，并行的任务需要分配不同的端口和测试数据库，或各自在独立的容器中运行。并行会把压力转移到审查（问题 28），并行的数量应该以团队能审查的速度为上限。

**机制二：工具调用设置超时，重试有上限。** 模型调用、构建和测试都要有超时；重试使用退避策略，并限制最大次数。

**机制三：按失败类型决定是否重试。**

| 失败类型 | 示例 | 处理方式 |
| --- | --- | --- |
| 临时错误 | 网络超时、限流（429）、服务端错误（5xx） | 退避后重试，最多 N 次 |
| 确定性错误 | 编译失败、测试失败、架构规则不通过 | 不重试，按 6.3 的方式调试 |
| 结果不符合要求 | 生成的代码偏离了 Spec | 不重试，先检查任务卡和 Spec 是否写清楚 |

对确定性错误重试，只会得到同样的失败；对结果不符合要求的情况反复重试，相当于在掷骰子。

## 7. 落地路线：从命令和模板开始

不要第一天就搭一套复杂的多 Agent 编排平台。按三个版本逐步演进（周期为参考值）：

### 7.1 V1：命令与模板（参考 1 周）

在 Agent 工具中配置几个标准命令，把 01–04 的产物契约落到 Git：

| 命令 | 阶段 | 产出 |
| --- | --- | --- |
| `/archaeology` | Phase 1 | `01-context-truth.md` |
| `/spec` | Phase 2 | `02-clarifications.md`；卡点 A 之后生成 `03-spec.md` |
| `/redteam` | Phase 3 | `04-redteam-audit.md` |
| `/implement` | Phase 4 | 任务卡、实现代码、验证报告 |

以 Claude Code 为例，项目中的 `.claude/commands/<名称>.md` 就是一个自定义命令，文件内容是提示词：

```markdown
---
description: Phase 1 事实考古：生成 01-context-truth.md
disable-model-invocation: true
---

读取当前需求目录下的 00-raw-prd.md，以及脚本已经采集的事实文件：依赖树、表画像、下游清单、匹配的 domain-rules。
只做一件事：把代码中的分支判断、校验和约束整理成 As-Is 规则清单，每条注明文件和行号。
找不到出处的规则不要写。不要提出方案，不要修改任何代码。
输出写入 01-context-truth.md。
```

`disable-model-invocation: true` 让这个命令只在人明确调用时运行，不会被模型自行触发。

`/redteam` 必须在隔离的上下文中运行。两种做法：

- **子代理**：在 `.claude/agents/redteam.md` 中定义一个只有只读工具的子代理。子代理在自己的上下文窗口中运行，但它收到的任务描述由主会话写出，可能带入主会话的倾向，所以任务描述中只写文件路径，不写对方案的评价。
- **非交互的新会话**：在 CI 或脚本中用 `claude -p` 启动一个全新的会话，只放行只读工具，隔离更彻底：

  ```bash
  SPEC_DIR=.specs/FEAT-2048-modify-order-address
  claude -p "先阅读 docs/prompts/redteam.md，严格按其中的要求，审查 $SPEC_DIR/01-context-truth.md 和 $SPEC_DIR/03-spec.md，按 04-redteam-audit.md 的格式输出，决策列一律填“待裁决”。" \
    --allowedTools "Read,Grep,Glob" > "$SPEC_DIR/04-redteam-audit.md"
  ```

  要使用另一个模型家族做红队时，把这一步换成对应模型服务的命令行工具或 API。

### 7.2 V2：CI 强绑定（参考 1 个月）

- 把 5.1 的定级脚本、6.1 的范围检查、执行计划门禁（运行可靠性 4.2 节）、ArchUnit 架构规则和并发测试，接入 GitHub Actions 或 GitLab CI。
- 同步生产元数据快照，生成生产画像。
- 在核心业务域沉淀出第一版 `domain-rules/` 和依赖图谱（需求工程 4.2、4.3 节）。

### 7.3 V3：运行时联防与免疫闭环（参考 1 个季度）

- 通过只读、脱敏的 MCP 接口，接入可观测性数据和血缘查询。
- 金丝雀发布接入双信号回滚：除了黄金指标（5xx、P99 延迟），再加**正确性探针**。有一类故障不会让错误率升高：接口返回 200，但数据是错的，例如串了租户、金额算错。正确性探针在金丝雀实例上对抽样请求做断言（响应中的租户必须等于请求的租户），或做旁路实时对账，发现这类“静默损坏”。金丝雀配置见运行可靠性 4.3 节。
- 建立 Phase 6 闭环：线上复盘由复盘 Agent 提 PR，补充规则账本、架构禁忌、回归用例和生产画像，经过 CI 和人工审查后合并（见运行可靠性 4.4 节）。

## 8. 示意案例：FEAT-2048 已支付订单修改收货地址

下面是一个示意场景，用来说明六个阶段如何配合，不是实际发生的事故。

| 阶段 | 发生了什么 |
| --- | --- |
| Phase 0 | 需求涉及订单主状态、物流、税务和发票，脚本定为 L3 |
| Phase 1 | 工具拉出 5 个真实下游和订单表量级；考古 Agent 整理出 As-Is 规则：已发货订单不允许改地址（附文件和行号） |
| Phase 2 | 歧义清单问：“支付后”是否包括已发货？跨税区时税费怎么算？卡点 A 中产品直接勾选回答，Spec 冻结出决策表和反例 |
| Phase 3 | 业务红队假设上线一个月后出现大量客诉，倒推出“改地址后没有作废原电子发票”；架构红队发现“取消”和“改地址”并发时存在竞态。卡点 B 中两条都采纳修改 |
| Phase 4 | Test Agent 只依据 Spec 生成验收测试和并发时序测试。Code Agent 的实现在并发测试上失败；第二次修复时它在业务代码中加了 `synchronized`，被架构规则和定级复核同时拦下；第三次仍未通过，Agent 停下来汇报，人决定使用平台层的订单锁原语，测试通过 |
| Phase 5 | 金丝雀阶段，正确性探针抽样比对订单地址与物流面单地址，全部一致，逐步放量 |
| Phase 6 | “改地址必须同步作废物流面单和发票”写进领域规则账本，并关联回归用例 |

人在整个过程中做了四个决定：回答歧义、裁决两条红队问题、选择互斥方案、合并前确认能解释改动。其余工作由 Agent 和确定性工具完成。

## 9. 一个最小对照实验：验证这条工作流是否有效

### 9.1 实验设计

准备 12 个需求，L1、L2、L3 各 4 个。L2 和 L3 的需求中预埋前几篇提到的陷阱：歧义、隐式规则、下游联动、并发竞态、历史脏数据。分三组运行：

- **A 组（放任）：** 把需求直接交给 Agent，在同一个会话中完成分析、编码和自我评审。
- **B 组（新瀑布）：** 所有需求都按 L3 走完整流程。
- **C 组（分级路由）：** 按本文的设计，由 Phase 0 定级后走对应的关卡。

在 C 组内再做两个对照：红队使用同一个模型家族或不同的模型家族；Test Agent 与 Code Agent 隔离或不隔离。每个需求每组重复 3 次，固定模型版本、工具集合和仓库版本。

### 9.2 记录指标

| 指标 | 说明 |
| --- | --- |
| 缺陷逃逸数 | 合并后被隐藏验收测试和故障注入发现的问题数 |
| 人工耗时 | 人在卡点、越界处理和审查上花费的时间 |
| 端到端耗时 | 从需求到可以合并的时间 |
| 定级一致率 | Phase 0 的定级与专家标注一致的比例，以及低估等级的次数 |
| 红队有效问题率 | 红队提出的问题中被人判定为成立的比例，按同源和异构模型分别统计 |
| 测试被放宽次数 | Test Agent 与 Code Agent 隔离和不隔离时，验收测试被修改或放宽的次数 |

29 开发者技能退化在短期实验中测不出来，需要通过定期的代码阅读和排障复盘长期观察。

**当前这里给出的是实验方案，不是实验结果。** 在实际运行并记录数据之前，不应声称这条工作流已经降低了缺陷逃逸或返工。

### 9.3 如何解释结果

- C 组的缺陷逃逸数接近 B 组，人工耗时明显低于 B 组：分级路由在不明显增加风险的情况下降低了流程成本。
- C 组出现低估等级、导致缺陷逃逸的情况：定级规则需要补充，按 Phase 6 的方式把漏判的案例写成新规则。
- 异构模型的红队有效问题率明显更高：值得为红队单独使用另一个模型家族；差异不明显，则可以省去这部分成本和合规审批。
- 隔离后测试被放宽的次数明显减少：Test Agent 与 Code Agent 的隔离起了作用。

## 10. 局限与权衡

**定级规则会误判。** 路径和文本匹配只能识别一部分风险；规则偏严会让太多需求进入重流程，偏松会漏掉风险。需要用实际案例持续调整。

**产物可能变成走过场的文档。** AI 可以很快生成大段没人看的文档。产物应以表格、选择题和清单为主，卡点只呈现需要人决定的部分；CI 检查结构和状态，而不是字数。

**隔离有代价。** 红队看不到讨论过程，可能重复提出已经讨论过的问题。讨论的结论应写进 `03-spec.md`，让红队从 Spec 中看到。

**异构模型增加成本和合规要求。** 需要接入另一家模型服务，经过模型访问网关和数据审批，效果也需要验证。

**人工卡点可能成为瓶颈。** 卡点 A、B 需要产品和架构师及时响应，否则流水线会卡住。需要为卡点设定响应时限和备选签字人。

**编排基础设施有建设成本。** V1 用命令和模板就能跑起来，不要一开始就搭建复杂的编排平台。

## 11. 常见问题诊断表

| 症状 | 可能原因 | 优先处理方式 |
| --- | --- | --- |
| 研发开始绕过流程，把需求拆小或藏进别的 PR | 一刀切，所有需求都走重流程 | 分级路由；L1 走快车道 |
| 低风险需求合并后出了严重问题 | 定级被低估 | PR 阶段按实际改动复核定级；漏判案例写成新规则 |
| 红队总是说“总体不错” | 红队继承了讨论上下文，或提问带有倾向 | 红队在全新会话中运行，只读取 01 和 03；指令禁止赞美，要求失败场景 |
| 验收测试被改松，CI 变绿 | Code Agent 能修改 Test Agent 的测试 | 验收测试放进任务卡的 forbidden_paths；测试保护 |
| 一个需求的 PR 改了几十个文件 | 没有任务卡，Agent 顺手重构 | 任务卡声明范围；CI 检查改动范围 |
| 本地通过，CI 失败 | Agent 和 CI 的环境不一致 | 使用同一个开发容器镜像 |
| 同一个错误修了十几次 | 没有复现，没有假设记录，没有预算 | 先复现；假设记录；尝试预算和停止汇报 |
| 线上错误率正常，但数据是错的 | 只看黄金指标，没有正确性探针 | 金丝雀阶段加正确性探针和旁路对账 |
| 三个月后说不清当时为什么这样设计 | 需求过程留在聊天记录里 | 产物落盘到 `.specs/`，随 PR 归档 |
| 同类事故反复出现 | 复盘只修 Bug | Phase 6：复盘产出规则、用例或画像的更新 |

## 12. 经验总结

1. **既不放任，也不做新瀑布。** 放任会让执行循环失控，新瀑布会被研发绕过。
2. **按爆炸半径分级路由。** 定级由脚本完成，需求阶段初判，PR 阶段按实际改动复核，调低必须签字。
3. **产物接力，严禁对话继承。** 阶段之间只通过 Git 中的结构化文件交接，每个角色在全新会话中启动。
4. **每个概率性节点后面，紧跟一个确定性卡点。** AI 负责生成和找问题，放行由确定性工具和人决定。
5. **确定性工具采集事实，模型做推理与对抗。** 考古先跑工具再跑 Agent，规则标注行号并由脚本核对。
6. **测试与编码隔离，业务代码只用平台原语。** Test Agent 看不到实现，Code Agent 改不了验收测试；平台层的边界由架构测试守住。
7. **给执行循环装上边界。** 任务卡、一致的环境、尝试预算、按失败类型重试；越界或预算耗尽就停下来交给人。
8. **人只站在高杠杆的裁决点。** 回答歧义、裁决风险、处理越界、合并前能解释改动。
9. **从命令和模板开始演进。** V1 命令与模板，V2 接入 CI，V3 运行时联防与免疫闭环。

---

## 延伸阅读

- [AI Coding 常见问题与治理要点](../01-problems/01-ai-coding-common-problems.md)：本文对应的问题定义
- [AI Coding 需求工程](../05-requirements/01-ai-coding-requirements.md)：Phase 1–3 的具体做法，以及红队指令模板
- [AI Coding 验证体系](../02-verification/01-ai-coding-verification.md)：测试保护与分级验证报告
- [AI Coding 安全与合规](../03-security/01-ai-coding-security.md)：Agent 的权限分级（问题 24）与模型访问网关
- [AI Coding 运行可靠性](../04-runtime/01-ai-coding-runtime-reliability.md)：生产画像、执行计划门禁、金丝雀发布与事故转规则
- [Claude Code 子代理文档](https://docs.anthropic.com/en/docs/claude-code/sub-agents)：子代理的定义方式与上下文隔离
- [Development Containers](https://containers.dev/)：Dev Container 规范
- [git-worktree 文档](https://git-scm.com/docs/git-worktree)：在同一个仓库中同时检出多个工作目录
- 下一篇：[AI Coding 代码质量门禁：别让“看起来对”通过审查](../07-quality-gate/01-ai-coding-quality-gate.md)

> **最后一句：让工具查事实，让蓝队建规约，让隔离的红队找漏洞，让人做裁决，让确定性的 CI 和运行时不变量兜底，最后把每一次线上问题编译回系统规则。**
