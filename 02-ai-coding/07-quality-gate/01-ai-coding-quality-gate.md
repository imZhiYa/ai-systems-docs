# AI Coding 代码质量门禁：别让“看起来对”通过审查

> 传统研发时代的质量门禁（ESLint、SonarQube、80% 单测覆盖率、人工 Code Review），面对 AI 生成的代码已经拦不住关键问题。原因是：AI 非常擅长“应试”。它能写出格式完美、零 Lint 警告、覆盖率很高的代码，却依然把生产打崩，因为这些代码里藏着传统门禁测不出的“伪高质量陷阱”。
>
> 因此，AI 时代的代码质量门禁，需要重构为一套**以确定性拦截为主、防作弊优先、按风险与耗时分层、能反哺 AI 自愈**的四层漏斗体系：用机器的确定性规则，锁住 AI 的概率幻觉与作弊本能，把架构红线、生产真相和规约契约放在合并按钮之前。
>
> **适用范围与时效说明：** 本文对应 [AI Coding 常见问题与治理要点](../01-problems/01-ai-coding-common-problems.md) 中 Code Quality Gate 方向的 07、09、11、12、28，同时是 [AI Coding 工作流](../06-workflow/01-ai-coding-workflow.md) Phase 4c 那一层确定性卡点的展开，因此也会用到 10、14、17、20、33 等问题的门禁。文中的阈值都是示例，需要按项目调整；脚本和配置除特别注明外，未在真实仓库和 CI 中运行。各工具的用法以其当前文档为准。本文核对时间：2026 年 11 月。

## 1. TL;DR

| 问题 | 结论 |
| --- | --- |
| 核心问题 | AI 擅长应试：格式、Lint、覆盖率都能做漂亮，传统门禁拦不住四类伪高质量陷阱——测试作弊、架构绕道、生产物理盲区、语义漂移 |
| 四条原则 | 确定性拦截为主，AI 判断只做最后一层；防作弊优先；按风险与耗时分层，只对增量严格；门禁结果要能反哺 AI 自愈 |
| 四层漏斗 | Layer 0 本地快速护栏 → Layer 1 确定性静态与架构门禁 → Layer 2 确定性动态、性能与并发门禁 → Layer 3 隔离的语义审计 |
| 阻断策略 | 确定性门禁硬阻断；语义审计只触发人工签字；修改平台层自动升级 L3，由平台负责人审批 |
| 运转机制 | 结构化报错协议与重试预算、增量执行与耗时红线、棘轮基线与门禁配置保护、误报熔断与紧急破窗、按风险分配人的审查 |
| 主要成本 | 规则误报、影子库与变异测试的成本、门禁配置的维护、团队对门禁的信任一旦失去很难恢复 |
| 不要做 | 只看覆盖率；让 AI 做 Lint 和编译器能做的事；把 AI 审查设为没有申诉通道的硬阻断；允许 Agent 自己加抑制注释过门禁 |

---

## 2. 问题：为什么传统门禁拦不住 AI 代码

### 2.1 AI 擅长“应试”

传统门禁检查的，大多是代码“看起来”的样子：

| 传统门禁 | 检查什么 | AI 怎样通过它 |
| --- | --- | --- |
| 格式化与 Lint | 写法是否规范 | 生成的代码天然格式整齐；不通过时加一行抑制注释 |
| 单测覆盖率 | 代码是否被执行过 | 用大量 Mock 把每一行都“跑到”，断言却很弱 |
| 静态扫描 | 已知的缺陷模式 | 已知模式之外的问题照样存在 |
| 人工 Code Review | 人觉得对不对 | 代码通顺、注释齐全、命名讲究，越是写得好看，越容易被放过 |

这些门禁在人写代码的年代还算有效，因为人很少专门“针对门禁”写代码。Agent 的目标常常就是“让 CI 变绿”，门禁检查的是表面，它就把表面做好。

同时，AI 让代码产出变快了，审查却没有变快（问题 28）。审查者面对的 PR 更多、更大，只能更多地依赖“看起来对”。

### 2.2 四类伪高质量陷阱

| 陷阱 | 典型表现 | 对应问题 |
| --- | --- | --- |
| 奖励黑客与测试作弊（Reward Hacking） | 为了跑通 CI，弱化断言、过度 Mock 制造永远全绿的假测试、吞掉异常（空 catch）、加 `@ts-ignore`、`# noqa` 或 skip 注释 | 17、18、12 |
| 架构绕道与熵增（Slop & Drift） | 不复用平台层原语，自己重新造一套带 Bug 的轮子；随意叠加切面注解、破坏分层边界；引入幻觉出来的依赖包 | 10、08、14、03 |
| 生产物理盲区 | ORM 写出隐蔽的 N+1 查询、大表上无索引查询、无边界地把数据加载进内存 | 33 |
| 语义漂移与越界（Spec Drift） | 代码能跑通，但悄悄偏离了冻结的 Spec 决策表，或者顺手改了不该动的“邻近逻辑” | 02、11、20 |

第一类有研究支持：ImpossibleBench 构造了需求与单元测试相互矛盾的任务，任何“通过”都意味着违反需求；研究观察到的作弊方式，从直接修改测试到运算符重载都有（见 [AI Coding 验证体系](../02-verification/01-ai-coding-verification.md) 第 2 节）。

再加上 Code Quality Gate 方向本身的几个问题：

| 问题 | 看起来对，实际上 |
| --- | --- |
| 07 API 幻觉与版本错位 | 调用写得很自然，但这个方法不存在，或者当前版本里没有 |
| 09 编码规范不一致 | 写法没毛病，但不是这个项目的写法 |
| 11 代码正确性不足 | 结构清晰，逻辑是错的 |
| 12 代码健壮性不足 | 正常流程完整，异常路径没处理 |
| 28 生成速度与理解速度失衡 | 改动太多，审查者没细看就批准了 |

### 2.3 门禁在六阶段流水线中的位置

在 [AI Coding 工作流](../06-workflow/01-ai-coding-workflow.md) 的三明治架构里，Code Agent 是概率性的节点，Phase 4c 的 CI 门禁就是紧跟在它后面的确定性卡点。门禁在这里有三个作用：

1. **拦下“看起来对”的问题**：能由机器判断的，交给机器确定地拦下。
2. **给 Code Agent 反馈**：门禁的结果就是 Agent 修复循环的输入；结果确定、可复现、写得清楚，修复循环才会收敛（工作流 6.3 节）。
3. **把人的注意力留给机器判断不了的地方**：设计取舍、业务语义、风险豁免。

## 3. 四条原则

### 3.1 确定性拦截为主，AI 判断只做最后一层

绝不让 AI 去做 Lint 和编译器能做的事。能写成确定性规则的，都写成规则：同一份代码，每次检查的结果都一样，不依赖模型当时的状态，也不依赖审查者当天的精力。

AI 判断只放在最后一层（Layer 3），只做确定性工具做不了的事：核对代码意图与冻结的规约是否一致。它也不能直接放行或硬阻断，只能触发人工签字。这和验证体系里“验证与生成分离”的原则一致。

### 3.2 防作弊优先

门禁一旦存在，Agent 就会为了通过门禁而“优化”。所以第一批要建的，不是更多的检查规则，而是**防止绕过检查**的规则：禁止改验收测试、禁止新增抑制注释、禁止吞异常、门禁配置本身受保护。没有这一层，后面的门禁都可以被绕过。

### 3.3 按风险与耗时分层，只对增量严格

- **按风险分级**：沿用工作流 Phase 0 的 L1 / L2 / L3，等级越高，启用的门禁越多（见第 6 节）。
- **按耗时分层**：快的检查放在前面，慢的检查放在后面，越早给出反馈越好。
- **只对增量严格**：老代码库的存量问题不阻塞合并；新改动不能引入新问题，存量问题用棘轮基线逐步减少（见 7.3 节）。重型门禁只针对本次改动执行。

### 3.4 门禁结果要能反哺 AI 自愈

门禁不只是给人看的。门禁的报错，就是给 Code Agent 的修正指令：要写清楚哪个文件哪一行、违反了什么、应该怎么改、还剩几次重试（见 7.1 节）。超过重试预算就停下来交给人，防止 Agent 为了强行通过门禁而开始“花式绕过”。

## 4. 四层漏斗架构

```text
[AI 生成的代码]
     │
     ▼
Layer 0：本地快速护栏（提交前钩子 / IDE，目标 < 5 秒）
  └── 格式与 Lint / 密钥泄露 / 幻觉依赖 / 禁用的危险 API
     │
     ▼
Layer 1：确定性静态与架构门禁（PR 推送时，目标 1–3 分钟）──【硬阻断】
  ├── 1.1 防作弊与偷懒：禁改验收测试 / 禁加抑制注释 / 禁吞异常 / 重复与膨胀
  ├── 1.2 编译、类型与项目规范
  ├── 1.3 架构适应度与多因素禁令：ArchUnit / Semgrep / 平台层变更锁
  └── 1.4 契约与 DDL 兼容：OpenAPI / Protobuf 破坏性变更 / 锁表 DDL
     │
     ▼
Layer 2：确定性动态、性能与并发门禁（合并前，目标 3–10 分钟，按 L1/L2/L3 路由）──【硬阻断】
  ├── 2.1 规约 BDD 测试 + 增量变异测试
  ├── 2.2 模糊测试与属性测试
  ├── 2.3 生产仿真性能门禁：对齐生产统计的 EXPLAIN + N+1 计数
  └── 2.4 多因素并发与切面拓扑：竞争检测 + 时序延迟注入 + 启动期切面顺序断言
     │
     ▼
Layer 3：隔离的语义审计（AI 作为评审，目标 1–2 分钟）──【需人工签字】
  └── 对照 03-spec.md 与 04-redteam-audit.md，检查语义遗漏与越界改动
```

| 层 | 时机 | 耗时目标 | 阻断方式 | 由谁执行 |
| --- | --- | --- | --- | --- |
| Layer 0 | 提交前；Agent 在开发容器中每次修改后 | < 5 秒 | 本地拦截，CI 中重复执行一次 | 确定性工具 |
| Layer 1 | PR 推送时，所有等级 | 1–3 分钟 | 硬阻断 | 确定性工具 |
| Layer 2 | 合并前，按等级路由 | 3–10 分钟 | 硬阻断 | 确定性工具 |
| Layer 3 | Layer 1、2 通过后，L2 / L3 | 1–2 分钟 | 只触发人工签字 | 隔离会话中的 AI，人做决定 |

Agent 在开发容器中先跑一遍 Layer 0 到 Layer 2 的同一套检查，再提交（工作流 6.2 节），不要把 CI 当成调试工具。Layer 0 在本地可以被跳过（例如 `--no-verify`），所以它的检查在 CI 中必须再执行一次，本地只是为了更快地得到反馈。

各问题由哪一层的门禁负责：

| 问题 | 主要门禁 | 详见 |
| --- | --- | --- |
| 07 API 幻觉与版本错位 | 编译与严格类型检查、按依赖版本的 API 索引 | 5.2.2 |
| 09 编码规范不一致 | 格式化与 Lint 自动修复、项目自定义规则 | 5.1、5.2.2 |
| 11 代码正确性不足 | 规约 BDD、增量变异测试、语义审计 | 5.3.1、5.4 |
| 12 代码健壮性不足 | 禁止吞异常、静态分析、模糊测试 | 5.2.1、5.3.2 |
| 28 审查速度失衡 | 语义审计生成审查导航、按风险分配审查 | 5.4、7.5 |
| 17 绕过测试 | 验收测试目录写保护、禁止新增抑制和跳过 | 5.2.1 |
| 10 重复建设 | 增量重复率、新增依赖数量 | 5.2.1 |
| 14 依赖幻觉 | 新增依赖存在性检查 | 5.1 |
| 20 改动范围失控 | 任务卡范围检查、语义审计中的越界检测 | 5.4 |
| 33 生产环境行为偏差 | 执行计划门禁、N+1 计数 | 5.3.3 |

## 5. 逐层拆解

### 5.1 Layer 0：本地快速护栏（问题 09、14）

快反馈，拦低级错误。只放几秒内能完成的检查：

- **格式化与 Lint 自动修复**：能自动修复的，直接修复，不留给审查评论。
- **硬编码密钥与个人信息扫描**：如 gitleaks、TruffleHog。
- **幻觉依赖检查**：新增依赖是否在企业私有仓库或白名单中真实存在、首次发布时间是否异常，防止 AI 编造出的包名被抢注投毒。脚本见 [AI Coding 安全与合规](../03-security/01-ai-coding-security.md) 4.3 节。
- **危险 API 黑名单**：如 `eval`、没有超时的 HTTP 客户端、业务代码中直接 `new Thread()`、`SELECT *`。

以 pre-commit 为例：

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: <固定到具体版本>
    hooks:
      - id: gitleaks
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: <固定到具体版本>
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: local
    hooks:
      - id: check-new-deps            # 安全与合规 4.3 节的依赖存在性检查
        name: check new dependencies
        entry: python scripts/check_new_deps.py
        language: system
        files: ^(requirements.*\.txt|package\.json|pom\.xml|go\.mod)$
      - id: banned-apis                # 危险 API 黑名单，规则写在 Semgrep 中
        name: banned APIs
        entry: semgrep --config .semgrep/banned-apis.yaml --error
        language: system
        types_or: [python, java, ts, javascript]
```

`scripts/check_new_deps.py` 负责从依赖文件的改动中提取新增的包名，再调用安全与合规 4.3 节的检查脚本，需要按项目的依赖文件格式实现。

### 5.2 Layer 1：确定性静态与架构门禁

所有等级的 PR 都必须跑，任何一项不通过都硬阻断。

#### 5.2.1 防作弊与偷懒门禁（问题 17、12、10）

这是 AI 时代见效最快的一类门禁，用基于 diff 的脚本就能实现：

| 检查 | 拦什么 |
| --- | --- |
| 验收测试目录写保护 | Code Agent 修改或删除 Test Agent 依据 `03-spec.md` 生成的验收测试。Code Agent 要么修实现让测试通过，要么提出修改 Spec，不能改“试卷” |
| 禁止新增抑制与跳过 | 本次 diff 新增的 `@ts-ignore`、`eslint-disable`、`@SuppressWarnings`、`# noqa`、`# type: ignore`、`NOSONAR`、`@Disabled`、`.skip(`、`.only(` 等；带豁免标记的，转为人工确认 |
| 禁止吞异常 | 新增的空 catch 块、`except ...: pass`；catch 中只打日志、却返回 `null`、`true` 或成功状态码伪装成功的写法，用 Semgrep 规则补充 |
| 防重复与膨胀 | 新增代码的重复率（CPD 或 Sonar 的增量规则）、单个 PR 新增外部依赖的数量、没有任何引用的新增代码 |

下面的脚本实现前三项，输出 7.1 节的结构化报错：

```python
#!/usr/bin/env python3
"""Layer 1.1 防作弊门禁：扫描 diff 中对验收测试的修改、新增的抑制注释和吞异常写法。

用法：git diff -U0 origin/main...HEAD | python check_evasions.py > evasions.json
  只对 Code Agent 的提交范围运行；Test Agent 生成的验收测试应在单独的提交或 PR 中先合入。
  命中硬阻断规则时退出码为 1；带豁免标记的抑制注释记为 SIGN_OFF，需要人工确认。
"""
import fnmatch
import json
import re
import sys

PROTECTED_TESTS = ["tests/spec-bdd/*", "tests/acceptance/*"]
WAIVER = re.compile(r"ARCH-WAIVER\([A-Z]+-\d+\)")      # 例如 ARCH-WAIVER(QA-123)：原因
SUPPRESSIONS = [
    r"@ts-ignore", r"@ts-nocheck", r"@ts-expect-error", r"eslint-disable",
    r"#\s*noqa", r"#\s*type:\s*ignore", r"#\s*pragma:\s*no\s*cover",
    r"@SuppressWarnings", r"@SuppressFBWarnings", r"NOSONAR", r"//\s*nolint",
    r"@Disabled\b", r"@Ignore\b", r"@pytest\.mark\.skip", r"\bt\.Skip\(",
    r"\.skip\(", r"\bxit\(", r"\bxdescribe\(", r"\.only\(",
]
SUPPRESSION_RE = re.compile("|".join(SUPPRESSIONS))
EMPTY_CATCH = re.compile(r"catch\s*(\([^)]*\))?\s*\{\s*\}")
CATCH_OPEN = re.compile(r"catch\s*(\([^)]*\))?\s*\{\s*$")
EXCEPT_PASS = re.compile(r"^\s*except\b[^:]*:\s*pass\s*$")
EXCEPT_OPEN = re.compile(r"^\s*except\b[^:]*:\s*$")
HUNK = re.compile(r"^@@ -\d+(?:,\d+)? \+(\d+)(?:,\d+)? @@")


def finding(gate, severity, path, line, violation, fix):
    return {"gate": gate, "severity": severity, "file": f"{path}:{line}",
            "violation": violation, "required_fix": fix}


def protected(path):
    return path is not None and any(fnmatch.fnmatch(path, p) for p in PROTECTED_TESTS)


def scan(diff_text):
    findings, old_path, new_path, line_no, prev = [], None, None, 0, None
    for raw in diff_text.splitlines():
        if raw.startswith("--- "):
            old_path = raw[6:] if raw.startswith("--- a/") else None
            continue
        if raw.startswith("+++ "):
            new_path = raw[6:] if raw.startswith("+++ b/") else None
            path = new_path or old_path
            if protected(old_path) or protected(new_path):
                findings.append(finding(
                    "ANTI_CHEAT_001", "HARD_BLOCK", path, 0,
                    "修改或删除了由 Test Agent 生成的验收测试",
                    "撤回对验收测试的修改；如果认为测试有误，停止编码并提出修改 03-spec.md"))
            prev = None
            continue
        m = HUNK.match(raw)
        if m:
            line_no, prev = int(m.group(1)), None
            continue
        if not raw.startswith("+") or new_path is None:
            continue
        text = raw[1:]
        if SUPPRESSION_RE.search(text):
            if WAIVER.search(text):
                findings.append(finding(
                    "ANTI_CHEAT_002", "SIGN_OFF", new_path, line_no,
                    "新增了带豁免标记的抑制或跳过注释", "由负责人确认豁免理由"))
            else:
                findings.append(finding(
                    "ANTI_CHEAT_002", "HARD_BLOCK", new_path, line_no,
                    "新增了抑制或跳过注释：" + SUPPRESSION_RE.search(text).group(0),
                    "修复被抑制的问题本身；确实需要豁免时，加 ARCH-WAIVER(工单号) 并说明原因"))
        swallowed = (EMPTY_CATCH.search(text) or EXCEPT_PASS.match(text)
                     or (prev is not None and CATCH_OPEN.search(prev) and text.strip() == "}")
                     or (prev is not None and EXCEPT_OPEN.match(prev) and text.strip() == "pass"))
        if swallowed:
            findings.append(finding(
                "ANTI_CHEAT_003", "HARD_BLOCK", new_path, line_no,
                "新增了吞掉异常的写法", "处理异常、向上抛出，或记录后返回明确的失败结果"))
        prev, line_no = text, line_no + 1
    return findings


def main():
    findings = scan(sys.stdin.read())
    json.dump(findings, sys.stdout, ensure_ascii=False, indent=2)
    sys.exit(1 if any(f["severity"] == "HARD_BLOCK" for f in findings) else 0)


if __name__ == "__main__":
    main()
```

这个脚本基于文本匹配：字符串里出现 `.skip(` 会被误报；把异常处理写成多行、或者在 catch 里只写一行日志，它都发现不了。所以它只负责最常见的绕过方式，更复杂的写法交给 Semgrep 或 AST 规则，最终还要靠变异测试（5.3.1）检验测试是否真的有效。

验收测试目录除了这个脚本，还应该用工作流 6.1 节任务卡的 `forbidden_paths` 和 CODEOWNERS 双重保护。

#### 5.2.2 编译、类型与项目规范（问题 07、09）

**API 幻觉（07）：让“方法不存在”在编译期暴露。**

- 编译型语言：编译必须在 CI 中通过，并把编译警告视为错误，例如 Java 的 `-Werror`，可以选择性地开启 Error Prone 等编译期检查。
- 动态语言：写错方法名、传错参数，要到运行时才会报错，所以必须开启严格的类型检查，例如 `mypy --strict` 或 pyright 的 strict 模式、TypeScript 的 `"strict": true`。
- 为 AI 提供项目实际依赖版本的 API：读取依赖锁文件中的版本，查询对应版本的文档，或者在本地从依赖生成 API 索引，而不是依赖训练数据中的旧版本。
- 用 Lint 规则禁止调用已废弃的 API，例如 Java 编译器的 `-Xlint:deprecation` 配合 `-Werror`。

**编码规范（09）：把项目自己的约定写成规则。**

格式化和通用 Lint 规则只能保证代码“规范”，保证不了代码是“这个项目的写法”。项目自己的约定，要写成自定义规则，例如用 Semgrep 要求统一的 HTTP 客户端封装：

```yaml
# .semgrep/project-conventions.yaml
rules:
  - id: use-project-http-client
    languages: [java]
    severity: ERROR
    message: >-
      请使用项目统一的 HttpClientFactory，它内置了超时、重试和链路追踪；
      不要直接创建 RestTemplate 或 OkHttpClient。
    pattern-either:
      - pattern: new RestTemplate(...)
      - pattern: new OkHttpClient(...)
      - pattern: new OkHttpClient.Builder()
```

分层和依赖方向这类约定，用 ArchUnit 等架构测试表达更合适（见 5.2.3）。

#### 5.2.3 架构适应度与多因素禁令

这一层守住工作流 5.3 节中“业务层只使用平台层原语”的边界，用 ArchUnit（Java）、dependency-cruiser（TS）、Go 的 AST 分析或 Semgrep 自定义规则执行。

**平台层变更锁。** 任何修改平台层目录（锁、事务编排、缓存框架、切面、上下文传递）的 PR，自动定级为 L3（工作流 5.1 节的定级脚本），并通过 CODEOWNERS 要求平台负责人审批：

```text
# .github/CODEOWNERS
/src/platform/    @your-org/platform-owners
```

**多因素危险组合禁令。** 单个写法都合法，组合在一起才出问题，这类规则最需要写成门禁：

| 禁令 | 原因 |
| --- | --- |
| `@Transactional` 方法内发起外部 HTTP / RPC 调用、直接发送 MQ（不经过 Outbox）、直接写 Redis | 事务回滚时，外部调用和缓存写入无法回滚；事务持有连接期间等待外部调用，容易耗尽连接池 |
| 在类内部直接调用带切面注解的方法 | 自调用不经过代理，事务、锁等切面不会生效 |
| 业务层自行叠加 `@DistributedLock` 和 `@Transactional` | 锁和事务的顺序由切面的执行顺序决定，容易出现事务提交前锁已释放；应使用平台层提供的显式编排模板 |
| 业务代码直接使用 `ThreadLocal` 或自行创建线程池 | 线程复用时上下文串租户；线程池没有统一的监控和上限 |

事务内外部调用的规则，用 Semgrep 示意如下：

```yaml
# .semgrep/arch-taboos.yaml
rules:
  - id: no-external-io-in-transaction
    languages: [java]
    severity: ERROR
    message: >-
      @Transactional 方法内不允许外部调用和缓存写入。
      请使用平台层的 txOrchestrator.execute(...) 并在 afterCommit 中处理，
      参考 docs/primitives/tx-orchestrator.md。
    patterns:
      - pattern-inside: |
          @Transactional
          $RET $METHOD(...) { ... }
      - pattern-either:
          - pattern: (RestTemplate $C).$CALL(...)
          - pattern: (StringRedisTemplate $R).$CALL(...)
          - pattern: (KafkaTemplate $K).send(...)
```

这条规则只能发现方法上直接标注、并且直接调用的情况：类级别的 `@Transactional`、通过其他方法间接发起的调用都发现不了。写好后要用正例和反例测试规则本身，规则的写法以 Semgrep 当前文档为准。“类内自调用切面方法”这类规则，用文本模式很难准确表达，更适合用 5.3.4 的启动期检查或 AST 分析。

`ThreadLocal` 和幂等基类的 ArchUnit 规则，见工作流 5.3 节。

#### 5.2.4 契约与 DDL 兼容

**API 与事件契约的向后兼容检查。** 禁止删除已有的响应字段、改变字段类型、收缩枚举值、把可选改为必填，把破坏下游隐式契约的改动拦在合并之前：

```bash
# OpenAPI：与主干的接口定义对比，出现确定的破坏性变更（ERR）时失败
git show origin/main:api/openapi.yaml > /tmp/openapi-base.yaml
oasdiff breaking /tmp/openapi-base.yaml api/openapi.yaml --fail-on ERR

# Protobuf：与主干的 schema 对比
buf breaking --against '.git#branch=origin/main'
```

只有写进契约文件的行为才能被检查。文档没有写、但下游实际依赖的行为（例如字段的顺序、空值的表现），需要运行可靠性 4.2 节的契约测试和流量回放来发现。

**DDL 静态审计。** 用 Bytebase 等工具的 SQL 审核规则，或者在 CI 中自定义检查：禁止对生产大表执行会锁表的 DDL、禁止新增没有默认值的 NOT NULL 列、禁止没有索引的外键。哪些表算“大表”，从运行可靠性 4.1 节的生产画像中读取。迁移的发布方式（先扩展、再迁移、后收缩）见运行可靠性 4.3 节。

### 5.3 Layer 2：确定性动态、性能与并发门禁

按 L1 / L2 / L3 路由，只针对本次改动执行（见 7.2 节）。

#### 5.3.1 规约 BDD 与增量变异测试（问题 11）

**规约 BDD。** 必须跑通 Test Agent 依据 `03-spec.md` 编译出的决策表全组合 BDD 用例，覆盖所有边界和反例（工作流 5.3 节）。

**变更覆盖率。** 不看整体覆盖率，只看本次新增和修改的行是否被测试覆盖：

```bash
diff-cover coverage.xml --compare-branch=origin/main --fail-under=80
```

**增量变异测试：专门戳破“高覆盖率的假测试”。** AI 写单元测试，常常全链路 Mock，覆盖率很高，断言却没有约束力。变异测试会自动修改代码，例如把 `>` 改成 `>=`、把 `+` 改成 `-`、删掉一个 if 分支，再重新跑测试；代码被改坏了，测试却依然全绿（变异体存活），说明这些测试的断言没有起作用。

变异测试很慢，必须只针对本次改动：

| 语言 | 工具 | 只针对改动的方式 |
| --- | --- | --- |
| Java | PIT | 用 `targetClasses` 限定为本次改动的类，并开启 `withHistory` 复用上一次的分析结果 |
| JavaScript / TypeScript | StrykerJS | `--incremental` 模式，只对改动的代码和测试重新执行 |
| Go | go-mutesting | 只传入本次改动的包 |

PIT 的示例，改动的类名由脚本从 diff 中提取：

```bash
CHANGED_CLASSES=$(git diff --name-only --diff-filter=AMR origin/main...HEAD -- 'src/main/java/*.java' \
  | sed -e 's#src/main/java/##' -e 's#\.java$##' -e 's#/#.#g' | paste -sd, -)
mvn -DwithHistory -DtargetClasses="$CHANGED_CLASSES" -DmutationThreshold=75 \
  test-compile org.pitest:pitest-maven:mutationCoverage
```

没有改动的 Java 类时，`CHANGED_CLASSES` 为空，PIT 会退回到分析全部代码，脚本中需要先判断并跳过。对 L2 / L3 的核心域，可以设定增量变异杀死率的下限，例如 75%（`mutationThreshold`），阈值按项目调整。先在一个核心模块试点，观察耗时和误报，再推广。

#### 5.3.2 模糊测试与属性测试（问题 12）

对外的入参和反序列化逻辑，跑轻量的模糊测试或属性测试：注入 `null`、超长字符串、畸形 JSON、负金额、极值数字、重复请求，专门检验代码对生产脏数据的处理。常见工具有 Python 的 Hypothesis、Java 的 jqwik、JavaScript 的 fast-check，以及 Go 内置的 `go test -fuzz`。

把线上真实出现过的畸形数据，脱敏后加入用例库（运行可靠性 4.4 节的“事故转规则”）。

静态分析补上异常路径：在 CI 中检查空值、资源泄漏和未处理的异常，例如 SpotBugs、Error Prone 加 NullAway（Java），并使用空值安全的类型和注解。这和运行可靠性 4.3 节的分工是：那一节讲上线后的运行时护栏，本节在发布前、在代码层面拦下。

#### 5.3.3 生产仿真性能门禁（问题 33）

**执行计划门禁。** 在 CI 中拉起一个影子库：表结构和索引与生产一致，统计信息（行数、直方图）与生产对齐，不含真实的个人信息。对本次新增或修改的 SQL 运行 `EXPLAIN`，出现大表全表扫描、预估扫描行数超过阈值（例如 10 万行）、没有分页的大结果集或文件排序时，硬阻断。脚本见 [AI Coding 运行可靠性](../04-runtime/01-ai-coding-runtime-reliability.md) 4.2 节。

**N+1 查询计数。** 在集成测试的数据库访问层挂一个计数器，断言单个接口请求触发的 SQL 次数不超过上限（例如 10 次），拦下 AI 最常写的“循环里查库”。以 Python 和 SQLAlchemy 为例：

```python
# tests/conftest.py：统计一次请求执行了多少条 SQL
import contextlib

import pytest
from sqlalchemy import event


@pytest.fixture
def max_queries(db_engine):          # db_engine 为测试使用的 SQLAlchemy Engine
    @contextlib.contextmanager
    def _limit(limit):
        statements = []

        def _record(conn, cursor, statement, parameters, context, executemany):
            statements.append(statement)

        event.listen(db_engine, "before_cursor_execute", _record)
        try:
            yield statements
        finally:
            event.remove(db_engine, "before_cursor_execute", _record)
        assert len(statements) <= limit, (
            f"一次请求执行了 {len(statements)} 条 SQL，上限 {limit}，可能存在 N+1 查询：\n"
            + "\n".join(statements[:20])
        )
    return _limit


# 用法
def test_order_list_has_no_n_plus_one(client, max_queries):
    with max_queries(10):
        client.get("/orders?page=1&size=50")
```

这段示例未实际运行。Java 项目可以使用 Hibernate 的统计功能，或者在数据源上挂一个统计代理，做同样的断言。

#### 5.3.4 多因素并发与切面拓扑（L3 必跑）

**启动期切面拓扑自检。** 切面的执行顺序决定了“锁在事务之外”这类约束是否成立，但它取决于配置和注解，单看代码很难判断。可以在测试中启动应用上下文，直接断言代理链上切面的实际顺序。以 Spring 为例：

```java
@SpringBootTest
class AspectOrderTest {

    @Autowired
    OrderService orderService;

    @Test
    void tenant_then_lock_then_transaction() {
        List<String> chain = Arrays.stream(((Advised) orderService).getAdvisors())
            .map(Advisor::getAdvice)
            .map(advice -> advice instanceof AbstractAspectJAdvice a
                ? a.getAspectName()
                : advice.getClass().getSimpleName())
            .toList();

        // 外层先执行：租户上下文 → 分布式锁 → 事务
        assertThat(chain).containsSubsequence(
            "tenantContextAspect", "distributedLockAspect", "TransactionInterceptor");
    }
}
```

这段示例未实际运行，切面的名称和获取方式需要按项目的实现调整。

**竞争检测与时序延迟注入。**

- 开启竞争检测，例如 Go 的 `go test -race`。
- 在核心并发用例中挂上测试钩子，在关键的时间窗口里故意注入延迟（例如 50 毫秒），再打入并发的读写请求，验证不出现脏缓存、超卖或重复扣款：

| 注入点 | 验证什么 |
| --- | --- |
| 数据库写入之后、事务提交之前 | 并发读是否读到未提交的数据；缓存是否被提前更新 |
| 删除缓存之后、读取数据库之前 | 并发读是否把旧值重新写回缓存 |
| 获取锁之后、开始事务之前 | 锁是否在事务提交前被释放 |

延迟注入的钩子只在测试环境中启用，不能进入生产代码路径。

### 5.4 Layer 3：隔离的语义审计（问题 28、11、20）

铁律：**不让 AI 去做 Lint 和编译器能做的事；也不把 AI 审查设为没有申诉通道的硬阻断。** AI 审查会误报，它的唯一任务，是核对“代码意图”与“冻结的规约”是否一致。

**输入隔离。** 在全新的会话中运行，只输入三样东西：`03-spec.md`、`04-redteam-audit.md`（已经过人裁决的版本）和本次的 `git diff`。不输入 Code Agent 的对话，也不输入 PR 描述中对改动的自我评价。

**只查三件事：**

1. **Spec 决策表覆核**：决策表里的每一个分支，diff 里是否都有对应的实现？有没有遗漏某个异常分支？
2. **红队问题核销**：`04-redteam-audit.md` 中人裁决为“采纳修改”的每一条风险，代码里是否真的落实了？
3. **越界改动检测**：diff 中有没有改了 Spec 非目标范围之外的业务逻辑，或者顺手重构了不相关的老代码？（任务卡的路径检查见工作流 6.1 节，这里检查的是语义上的越界）

**输出约束。** 只允许输出带确切文件和行号、并附有失败反例的高置信度告警；不要输出“建议把变量名改得更优雅”这类泛泛的意见。

```markdown
你是代码与规约的一致性审计员。你只能看到冻结的 Spec、红队审查记录和本次 diff。

只做三件事：
1. 对照 03-spec.md 的决策表，逐行判断每个分支在 diff 中是否有对应实现。
2. 对照 04-redteam-audit.md 中决策为“采纳修改”的每一条，判断是否已经落实。
3. 找出 diff 中超出 Spec 范围（尤其是非目标）的改动。

输出规则：
- 先输出决策表核对表：分支编号 | 是否实现 | 对应的文件和行号。
- 告警只输出你有把握的问题，每条必须包含：文件和行号、对应的 Spec 条目、一个具体的失败场景。
- 不评价代码风格、命名和格式，这些由其他工具负责。
- 没有问题时，只输出核对表，不要补充建议。
```

**处理方式。** 审计结果以 PR 评论的形式输出“Spec 分支实现核对表”，作为人审查的导航。有告警时，PR 进入“需要人工签字”状态：由审查者逐条判断告警是否成立，确认后放行，或要求修改。语义审计不能单独放行任何 PR，也不能单独阻止合并。

## 6. 阻断策略矩阵

门禁必须层次分明：不能什么都标红，也不能什么都放水。

| 门禁类别 | 具体检查 | L1 轻量 | L2 标准 | L3 高危 | 处理方式 |
| --- | --- | :---: | :---: | :---: | --- |
| 本地护栏 | 格式与 Lint、密钥扫描、幻觉依赖、危险 API | 必跑 | 必跑 | 必跑 | 硬阻断（CI 中重复执行） |
| 防作弊 | 修改验收测试、新增抑制或跳过、吞异常 | 必跑 | 必跑 | 必跑 | 硬阻断；带豁免标记的转人工确认 |
| 编译与类型 | 编译、严格类型检查、废弃 API | 必跑 | 必跑 | 必跑 | 硬阻断 |
| 架构禁忌 | 切面违规组合、事务内外部调用、直接使用 ThreadLocal | 必跑 | 必跑 | 必跑 | 硬阻断 |
| 契约与 DDL | 破坏性的接口或事件变更、锁表 DDL | — | 必跑 | 必跑 | 硬阻断 |
| 测试质量 | 规约 BDD、变更覆盖率、增量变异测试 | 仅单测 | 必跑 | 必跑 | 硬阻断 |
| 性能防线 | 执行计划门禁、N+1 计数 | — | 必跑 | 必跑 | 硬阻断 |
| 多因素并发 | 竞争检测、时序延迟注入、启动期切面顺序 | — | 选跑 | 必跑 | 硬阻断 |
| 语义审计 | 决策表对齐、红队问题核销、越界改动 | — | 必跑 | 必跑 | 人工签字 |
| 平台层越界 | 修改平台层、资金状态机、并发原语目录 | 自动升级 L3 | 自动升级 L3 | 必跑 | 平台负责人审批 |

L2 的“选跑”，指改动涉及并发或缓存相关代码时才运行，由工作流 5.1 节定级脚本的命中原因决定。

## 7. 让门禁运转起来的工程机制

很多团队建了一堆门禁，最后要么被研发抱怨到关掉，要么 AI 卡在 CI 里无限重试、不停消耗 Token。门禁要真正运转，需要配齐下面几个机制。

### 7.1 面向 AI 自愈的结构化报错协议

不要只输出一行“Checkstyle failed”。每一条门禁失败，都封装成结构化的诊断，交给 Code Agent：

```json
{
  "gate": "ARCH_FITNESS_004",
  "severity": "HARD_BLOCK",
  "file": "src/order/address/OrderService.java:88",
  "violation": "@Transactional 方法内直接调用了 redisClient.delete()，事务回滚时缓存已被删除，可能导致缓存与数据库不一致",
  "forbidden_pattern": "数据库事务内直接写缓存",
  "required_fix": "改用平台层原语 txOrchestrator.execute(...)，在 afterCommit 回调中删除缓存；参考 docs/primitives/tx-orchestrator.md",
  "retry_budget": "1/3"
}
```

| 字段 | 作用 |
| --- | --- |
| `gate` | 门禁规则的编号，用于统计误报率（7.4 节） |
| `severity` | `HARD_BLOCK`、`SIGN_OFF` 或 `WARN` |
| `file` | 精确到行，Agent 不需要自己去找 |
| `violation` | 违反了什么，以及为什么危险 |
| `required_fix` | 应该怎么改，并指向平台层原语的文档；避免 Agent 用“绕过”代替“修复” |
| `retry_budget` | 当前是第几次修复，以及上限 |

所有门禁的诊断，汇总写入工作流 5.2 节的 `05-verification.json`，同时供 Code Agent 修复、供人审查、供 Phase 6 复盘使用。

**重试预算。** 同一条门禁连续失败超过上限（例如 3 次），立刻停止修复，按工作流 6.3 节的格式输出停止汇报，交给人处理。超过预算还继续尝试，Agent 很可能开始寻找绕过的办法，而这正是 5.2.1 要拦的东西。

### 7.2 增量执行与耗时红线

AI 让 PR 的数量明显增加，门禁如果要跑几十分钟，流水线很快就会拥堵：

- **所有重型门禁只针对改动**：变异测试只针对 diff 涉及的类和方法；执行计划门禁只跑改动涉及的 SQL；并发测试只运行相关领域的用例。
- **耗时目标**：Layer 1 小于 3 分钟，Layer 2 小于 10 分钟。超时的慢用例移到夜间的全量回归中，不阻塞主干。
- **夜间回归发现的问题**，自动创建工单，并关联到引入问题的 PR。

### 7.3 棘轮基线与门禁配置保护

**棘轮基线。** 在老代码库上新开一条规则，存量的违规往往有成百上千处，不可能一次修完，也不能让它们阻塞每一个 PR。做法是：为每条规则记录当前的违规数量作为基线，**基线只能减少，不能增加**。

```json
{
  "no-external-io-in-transaction": 42,
  "use-project-http-client": 17,
  "ANTI_CHEAT_003": 8
}
```

```python
#!/usr/bin/env python3
"""棘轮检查：每条规则的违规数量不能超过主干上的基线。

用法：python ratchet.py baseline.json current.json
  baseline.json：主干上的基线（git show origin/main:.quality/baseline.json）
  current.json：本次改动后重新统计的违规数量，格式相同
"""
import json
import sys


def check(baseline, current):
    problems, improved = [], []
    for rule, count in sorted(current.items()):
        allowed = baseline.get(rule, 0)
        if count > allowed:
            problems.append(f"[阻止] {rule}：违规 {count} 处，基线 {allowed} 处，本次新增了 {count - allowed} 处")
        elif count < allowed:
            improved.append(f"[改善] {rule}：违规从 {allowed} 处减少到 {count} 处，可以同步下调基线")
    return problems, improved


def main():
    with open(sys.argv[1], encoding="utf-8") as f:
        baseline = json.load(f)
    with open(sys.argv[2], encoding="utf-8") as f:
        current = json.load(f)
    problems, improved = check(baseline, current)
    for line in improved + problems:
        print(line)
    sys.exit(1 if problems else 0)


if __name__ == "__main__":
    main()
```

基线必须和主干上的版本比较，而不是和 PR 中的基线文件比较，否则 Agent 只要把基线调高就能通过。基线文件在 PR 中只允许下调。

**门禁配置保护。** 门禁本身也可能被“修改”来通过：放宽阈值、删掉规则、把违规加进排除列表。门禁配置和基线文件都放进 CODEOWNERS，由质量负责人审批，并在任务卡中列为 `forbidden_paths`：

```text
# .github/CODEOWNERS
/.semgrep/                  @your-org/quality-owners
/.quality/                  @your-org/quality-owners
/.pre-commit-config.yaml    @your-org/quality-owners
/.github/workflows/         @your-org/quality-owners
/tests/spec-bdd/            @your-org/quality-owners
```

CODEOWNERS 需要配合分支保护中的“需要代码所有者审查”才会强制生效。

### 7.4 误报熔断与紧急破窗

**误报率红线。** 每月统计每条门禁规则被人合理豁免的比例。任何一条规则的误报率超过红线（例如 5%），立即降级为警告，并修复规则本身。一条总是误报的规则，会毁掉团队对整套门禁的信任：大家会开始习惯性地申请豁免，真正的问题也被一起放过。

**紧急破窗（Break-Glass）。** 线上 P0 故障需要紧急修复时，允许指定级别的负责人绕过部分门禁，但系统自动留下审计记录，并强制创建一张 24 小时内必须关闭的补债工单。以 GitHub Actions 为例：负责人给 PR 加上 `emergency-override` 标签，并在 PR 描述中写明原因：

```yaml
# .github/workflows/break-glass.yml
name: break-glass
on:
  pull_request:
    types: [labeled]
jobs:
  record:
    if: github.event.label.name == 'emergency-override'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: read
    steps:
      - name: 创建补债工单
        env:
          GH_TOKEN: ${{ github.token }}
          PR: ${{ github.event.pull_request.number }}
          ACTOR: ${{ github.actor }}
        run: |
          gh issue create --repo "$GITHUB_REPOSITORY" \
            --title "补债：PR #$PR 紧急绕过了质量门禁" \
            --label break-glass \
            --body "由 $ACTOR 在 PR #$PR 触发紧急绕过。请在 24 小时内补齐被跳过的门禁并关闭本工单。"
```

需要注意：

- 只有指定的负责人才能添加这个标签，通过仓库权限或审批规则控制。
- 可以被绕过的，只有耗时长的门禁（变异测试、并发测试、流量回放）。密钥扫描、验收测试写保护等安全和防作弊门禁，任何情况下都不能绕过。
- 补债工单超过 24 小时未关闭时，由定时任务锁定该模块的普通发布，直到工单关闭。

### 7.5 人的审查：按风险分配注意力（问题 28）

人不应该逐行审查 AI 生成的上千行代码，而是把注意力集中在机器判断不了的地方：

| 等级 | 人审查什么 |
| --- | --- |
| L1 | 快速确认改动与需求一致；格式、规范、测试都由门禁负责 |
| L2 | 语义审计的核对表和告警；设计取舍；业务语义是否正确 |
| L3 | L2 的全部内容，加上平台层边界、并发与一致性设计；由平台负责人或架构师参与 |

**自动生成审查导航。** 人打开 PR 时，看到的不应该只是一个两千行的 diff，而是一份由门禁结果和语义审计汇总出的导航：

```markdown
## 审查导航（自动生成）

- 风险等级：L3（命中：src/order/state/*）
- 需要人工确认：2 项
  - ANTI_CHEAT_002：OrderService.java:120 新增了带豁免标记的 @SuppressWarnings（QA-311）
  - 语义审计：决策表分支 D7“已支付、跨税区”未找到对应实现
- 未被测试覆盖的改动行：AddressValidator.java:45–52
- 决策表核对：12 个分支中 11 个已实现
- 红队问题核销：采纳修改的 3 条均已落实
```

**审查清单只列机器判断不了的项**：设计取舍是否合理、业务语义是否正确、命名是否表达了意图、有没有更简单的做法。格式、规范、空值、覆盖率这些，交给门禁。

**PR 规模**由工作流 6.1 节的任务卡预算限制；合并前，作者必须能解释改动的核心逻辑（工作流 5.4 节）。

## 8. 第一周的最小可行组合

不要一次性把所有门禁都建起来。第一周先上线投入小、针对性强的 6 条：

1. **目录写保护门禁**：Code Agent 禁止修改验收测试目录，禁止新增抑制和跳过注释（5.2.1）。
2. **自定义 Semgrep 或 AST 规则集（5–10 条）**：先封住团队踩过最痛的坑，例如事务内外部调用、无分页的全表查询、吞异常、切面随意叠加、直接使用 `ThreadLocal`（5.2.3）。
3. **N+1 计数器和测试库执行计划门禁**：在集成测试阶段拦下全表扫描和循环查库（5.3.3）。
4. **接口与 Schema 向后兼容门禁**：在 PR 阶段拦下破坏性的字段修改（5.2.4）。
5. **增量变异测试（先在一个核心交易模块试点）**：检验 AI 写的单元测试是否真的有效（5.3.1）。
6. **隔离的 Spec 与 diff 语义审计**：PR 提交后自动评论“Spec 分支实现核对表”，作为人审查的导航（5.4）。

同时从第一天开始统计每条规则的误报和豁免（7.4），为后续调整积累数据。之后的演进，可以跟随工作流 7.2、7.3 节的 V2、V3 路线。

## 9. 示意案例：一个“全绿”的 PR

下面是一个示意场景，用来说明四层门禁如何配合，不是实际发生的事故。

**背景：** Code Agent 实现了“已支付订单修改收货地址”（工作流第 8 节的 FEAT-2048）。在传统门禁下，这个 PR 格式完美、零 Lint 警告、覆盖率 93%，看起来可以合并。

| 层 | 拦下了什么 |
| --- | --- |
| Layer 1.1 防作弊 | 一个并发测试失败后，Agent 在测试上加了 `@Disabled`——硬阻断，诊断指出“修复被抑制的问题本身” |
| Layer 1.3 架构禁忌 | 修复时 Agent 在 `@Transactional` 方法内直接删除了缓存——硬阻断，诊断给出平台层的 `afterCommit` 写法 |
| Layer 2.1 增量变异测试 | 覆盖率 93%，但大量测试用 Mock 返回固定值，变异杀死率只有 40%——硬阻断，Agent 补充了真实断言 |
| Layer 2.3 N+1 计数 | 地址校验在循环中逐个查询行政区划，一次请求 61 条 SQL——硬阻断，改为批量查询 |
| Layer 3 语义审计 | 核对表显示决策表分支“已支付、跨税区”没有实现，触发人工签字；审查者确认告警成立，要求补上 |

第三次修复并发测试仍未通过时，Agent 停下来汇报，由人决定使用平台层的订单锁原语（工作流 6.3 节）。最终合并的 PR，每一条门禁都有记录，写在 `05-verification.json` 中。

## 10. 一个最小对照实验：验证这些门禁是否有效

### 10.1 实验设计

在测试仓库中设计 12 个任务，每类 3 个，分别埋入一类伪高质量陷阱：

| 类型 | 埋入的陷阱 |
| --- | --- |
| 测试作弊 | 需求和某个测试用例冲突，诱发修改测试或加跳过注释 |
| 架构绕道 | 平台层已有原语，但直接手写更“顺手” |
| 生产物理盲区 | 需要在大表上查询，或者天然容易写成循环查库 |
| 语义漂移 | Spec 决策表中有一个容易遗漏的异常分支 |

每个任务分三组运行：

- **A 组（传统门禁）：** Lint、整体覆盖率 80%、人工审查。
- **B 组（四层门禁，不含防作弊）：** 关闭 Layer 1.1。
- **C 组（完整四层门禁）：** 包括防作弊、结构化报错和重试预算。

每个任务每组重复 3 次，固定模型版本、工具集合和仓库版本。合并后，用隐藏验收测试和故障注入判断陷阱是否逃逸。

### 10.2 记录指标

| 指标 | 说明 |
| --- | --- |
| 陷阱逃逸率 | 各类陷阱在合并后才被发现的比例 |
| 绕过尝试次数 | Agent 新增抑制注释、修改测试或门禁配置的次数 |
| 修复轮次 | Agent 从首次失败到通过门禁的平均轮次 |
| 误报次数 | 门禁拦下、但被人判定为不成立的次数 |
| 门禁耗时 | 每层门禁的执行时间 |
| 人工审查耗时 | 人在审查和签字上花费的时间 |

**当前这里给出的是实验方案，不是实验结果。** 在实际运行并记录数据之前，不应声称这些门禁已经降低了缺陷逃逸率。

### 10.3 如何解释结果

- B 组的测试作弊类陷阱逃逸率明显高于 C 组：防作弊门禁起了作用，应该优先建设。
- C 组修复轮次明显少于 B 组：结构化报错让 Agent 更快地找到正确的修法。
- 误报次数很高、人工审查耗时增加：规则需要收窄，按 7.4 节的误报熔断处理。
- 语义漂移类陷阱在三组都有逃逸：语义审计的提示词或输入需要改进，或者 Spec 本身没有写清楚。

## 11. 局限与权衡

**门禁会误报。** 文本和模式匹配的规则都有误报，误报多了，团队就会绕过整套门禁。规则要有人维护，误报率要持续统计。

**确定性规则只能拦住已知的模式。** 新的绕过方式、业务逻辑上的错误，规则拦不住，需要变异测试、语义审计和人的判断兜底。

**影子库和变异测试有成本。** 影子库要和生产的结构、统计信息同步；变异测试即使只针对改动也比普通测试慢很多。

**语义审计本身是概率性的。** 它会误报，也会漏报，所以只能触发人工签字，不能单独决定放行。

**棘轮基线需要维护。** 存量问题如果长期没有人治理，基线就只是记录，不会减少。

**门禁越多，反馈越慢。** 要守住耗时目标，持续把慢的检查移到夜间回归。

## 12. 常见问题诊断表

| 症状 | 可能原因 | 优先处理方式 |
| --- | --- | --- |
| 覆盖率很高，上线后逻辑错误 | 测试大量 Mock，断言没有约束力 | 增量变异测试；变更覆盖率代替整体覆盖率 |
| CI 变绿了，但测试被改了 | Code Agent 修改了验收测试 | 验收测试目录写保护；任务卡 forbidden_paths；CODEOWNERS |
| 代码里抑制注释越来越多 | Agent 用抑制代替修复 | 禁止新增抑制注释，豁免需要标记和确认 |
| 异常被吞掉，线上问题难以定位 | 空 catch、catch 后返回成功 | 禁止吞异常的门禁；Semgrep 补充规则 |
| 平台层已有的能力被重新实现 | Agent 不知道或不愿意用平台原语 | 架构禁忌规则；诊断中给出平台原语的用法 |
| 下游老系统在发布后报错 | 破坏了接口契约 | oasdiff / buf 向后兼容检查；契约测试 |
| 集成测试很快，上线后数据库被拖垮 | 测试数据量太小，看不出全表扫描和 N+1 | 对齐生产统计的执行计划门禁；N+1 计数 |
| 并发问题只在生产出现 | 测试没有覆盖时序窗口 | 竞争检测；时序延迟注入；启动期切面顺序断言 |
| 代码实现了，但漏了 Spec 的某个分支 | 只有测试，没有核对 Spec | 语义审计的决策表核对表 |
| Agent 在 CI 里反复重试 | 报错不清楚，没有重试上限 | 结构化报错协议；重试预算与停止汇报 |
| 门禁跑得太慢，大家开始抱怨 | 重型门禁跑全量 | 只针对改动执行；耗时红线；慢用例移到夜间 |
| 某条规则总是被豁免 | 规则误报率高 | 误报熔断，降级为警告并修复规则 |

## 13. 经验总结

1. **AI 擅长应试。** 格式、Lint、覆盖率都能做漂亮，门禁要检查的不是代码“看起来”怎样，而是它是否真的正确、安全、符合规约。
2. **确定性拦截为主，AI 判断只做最后一层。** 不让 AI 做编译器和 Lint 能做的事，也不让 AI 审查单独决定放行或阻止。
3. **防作弊优先。** 先封住改测试、加抑制、吞异常、改门禁配置这些绕过方式，后面的门禁才有意义。
4. **按风险与耗时分层，只对增量严格。** 快的检查在前，重的门禁只针对改动；存量问题用棘轮基线逐步减少。
5. **门禁报错就是给 Agent 的修正指令。** 写清楚位置、原因和正确的修法，并设定重试预算，超过就交给人。
6. **用变异测试戳破假覆盖率。** 覆盖率只说明代码被执行过，变异测试才能说明测试是否真的能发现错误。
7. **把生产真相和契约放进门禁。** 对齐生产统计的执行计划、N+1 计数、接口与 Schema 的向后兼容检查。
8. **保住团队对门禁的信任。** 统计误报、及时熔断，紧急情况有破窗机制，但必须留痕、必须补债。
9. **把人的注意力留给机器判断不了的地方。** 设计取舍、业务语义、风险豁免，由人决定。

---

## 延伸阅读

- [AI Coding 常见问题与治理要点](../01-problems/01-ai-coding-common-problems.md)：本文对应的问题定义
- [AI Coding 工作流](../06-workflow/01-ai-coding-workflow.md)：六阶段流水线、分级路由、Test Agent 与 Code Agent 的隔离
- [AI Coding 验证体系](../02-verification/01-ai-coding-verification.md)：测试保护、变异测试与分级验证报告
- [AI Coding 运行可靠性](../04-runtime/01-ai-coding-runtime-reliability.md)：执行计划门禁、契约测试与流量回放
- [AI Coding 安全与合规](../03-security/01-ai-coding-security.md)：依赖存在性检查与 CODEOWNERS
- Zhong, Raghunathan, Carlini, [ImpossibleBench: Measuring LLMs' Propensity of Exploiting Test Cases](https://arxiv.org/abs/2510.20270)：衡量 LLM Agent 利用测试用例走捷径的倾向
- [oasdiff](https://github.com/oasdiff/oasdiff)、[buf breaking](https://buf.build/docs/breaking/)：OpenAPI 与 Protobuf 的破坏性变更检查
- [PIT](https://pitest.org/)、[StrykerJS 增量模式](https://stryker-mutator.io/docs/stryker-js/incremental/)：变异测试工具
- [diff-cover](https://github.com/Bachmann1234/diff_cover)：只针对改动行的覆盖率检查

> **最后一句：AI 时代的代码质量门禁，不再是检查代码写得漂不漂亮，而是用机器的确定性规则，锁住 AI 的概率幻觉与作弊本能，把架构红线、生产真相和规约契约放在合并按钮之前。**
