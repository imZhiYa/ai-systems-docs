# AI Coding 架构约束：让正确的路成为最省力的路

> AI 写代码只看局部：完成眼前的任务，用最顺手的写法。架构却是一个整体性质，靠的是很多次“多走一步”：复用已有的组件、通过模块的公开接口调用、不为一个场景造一层抽象、顺手把临时代码清理掉。人写代码慢，架构也是慢慢腐化的；AI 让写代码几乎没有成本，架构腐化的速度也跟着被放大了。
>
> 靠“叮嘱 AI 遵守架构”守不住架构。本文的做法是两件事：**架构必须可执行**——边界、依赖方向和平台原语写成机器可读的定义，由门禁检查；**让正确的路最省力**——使用平台原语是最容易的写法，绕开它反而更费劲，还会被拦下。
>
> **适用范围与时效说明：** 本文对应 [AI Coding 常见问题与治理要点](../01-problems/01-ai-coding-common-problems.md) 中 Architecture Constraints 方向的 03、08、10、23。前几篇已经讲过业务层只能使用平台原语、架构禁忌门禁和上下文中的分层边界，本文只补上还没讲的部分：边界本身怎么定义、规则从哪里生成、平台原语怎么设计、技术债怎么持续偿还、架构怎么演进。文中的阈值都是示例；脚本除特别注明外，未在真实仓库中运行。各工具的用法以其当前文档为准。本文核对时间：2026 年 11 月。

## 1. TL;DR

| 问题 | 结论 |
| --- | --- |
| 核心问题 | AI 只看局部，架构是整体：每一次“局部最优”都合理，累积起来就是边界被绕过、抽象越来越多、同一能力多处实现、补丁越积越多 |
| 核心思路 | 架构必须可执行；让正确的路最省力 |
| 五条原则 | 架构即代码，一份定义是唯一来源；铺好的路；新增抽象要有理由；熵只降不升；改架构要一起改四处 |
| 关键机制 | 模块地图 `modules.yaml` 与生成的依赖规则、构建依赖隔离、可照抄的平台原语与使用率统计、抽象预算与人工确认、废弃流程与有边界的迁移任务、适应度函数、带“同步更新清单”的 ADR |
| 一份定义，多处生成 | 从 `modules.yaml` 生成门禁规则、上下文中的模块地图、CODEOWNERS 和风险定级中按模块判断的路径，CI 检查生成结果与定义一致 |
| 主要成本 | 模块地图的维护、边界划错时约束反而成为阻碍、抽象预算可能被机械执行、废弃流程会拖慢迭代 |
| 不要做 | 只在文档里画架构图；在多处手写同一条依赖规则；让 AI 在没有边界的任务里“顺便重构”；只禁止新增债务、从不偿还 |

---

## 2. 问题：AI 只看局部，架构是整体

### 2.1 四种“局部最优、整体变差”

| 问题 | 局部看是合理的 | 整体上的后果 |
| --- | --- | --- |
| 03 跨模块理解不足 | 改动在当前模块里是对的，直接调用另一个模块的内部类最快 | 下游被破坏，模块边界被绕过，模块之间越来越紧密地缠在一起 |
| 08 过度工程化 | 多一层接口、多一个工厂“更灵活” | 复杂度持续上升，没有人敢改 |
| 10 重复建设 | 自己写一个最快，不用去找已有的实现 | 同一能力有多个实现，行为不一致，修复不同步 |
| 23 技术债累积 | 先用临时方案让功能跑起来 | 补丁越积越多，职责越来越乱 |

这四个问题有一个共同点：**每一次决策单独看都没有错。** 问题清单中对它们的治理要点（检索已有实现、追问第二个使用场景、TODO 关联问题单）都是对的，但都依赖“每一次都做对”。人写代码时，这些“多走一步”的成本分摊在缓慢的开发过程中；AI 一次生成大量代码，每一次省掉的那一步都会被放大。

### 2.2 为什么叮嘱没有用

在说明文件里写“请复用已有组件”“请遵守分层”，只能降低问题发生的概率（问题清单第 3 节的约束力规则）。架构规则有三个特点，让它特别不适合靠叮嘱：

- **规则是全局的，而 Agent 的上下文是局部的。** “订单模块不能直接访问库存模块的数据表”这条规则，只有在看到整个模块结构时才有意义。
- **违反规则的代码看起来完全正常。** 绕过公开接口的调用、只有一个实现的接口，单看都是干净的代码。
- **代价是延迟的。** 违反规则不会让测试失败，后果要在几个月后的某一次修改中才显现。

所以架构必须**可执行**：写成机器可读的定义，由门禁在合并前检查。同时，只靠“禁止”还不够：如果正确的做法比错误的做法更麻烦，Agent 和人都会想办法绕过去。所以还要**让正确的路最省力**。

### 2.3 前几篇已经覆盖的部分

| 已经讲过 | 在哪里 | 本文补什么 |
| --- | --- | --- |
| 业务层只能使用平台层原语，ArchUnit 示例 | [工作流](../06-workflow/01-ai-coding-workflow.md) 5.3 | 边界本身怎么定义：模块怎么划分、公开接口是什么（4.1） |
| 架构禁忌门禁、平台层变更锁 | [代码质量门禁](../07-quality-gate/01-ai-coding-quality-gate.md) 5.2.3 | 规则从哪里来：由一份模块定义生成，不在各处手写（4.1、第 5 节） |
| C0 中的分层边界、按路径挂载的规则 | [上下文工程](../08-context/01-ai-coding-context.md) 4.1、4.2 | 同一份定义同时生成上下文和门禁（第 5 节） |
| 让可复用代码容易被检索到 | 问题 10 的治理要点 | 平台原语怎么设计，AI 才会优先用它（4.2） |
| 棘轮基线：存量违规只能减少 | 代码质量门禁 7.3 | 技术债怎么持续偿还，而不只是“不让变多”（4.4） |
| 用平台原语处理幂等、锁和缓存 | [运行可靠性](../04-runtime/01-ai-coding-runtime-reliability.md) 4.1、4.3 | 原语的使用率和“自己手写”的次数怎么度量（4.2、4.4） |

## 3. 五条原则

### 3.1 架构即代码：一份定义是唯一来源

模块的划分、每个模块的公开接口、允许的依赖方向，写成一份机器可读的定义（`architecture/modules.yaml`）。门禁规则、上下文中的模块地图、CODEOWNERS，以及风险定级中按模块判断的路径，都从这份定义**生成**，而不是在各处分别手写。手写在多处的同一条规则，迟早会互相不一致。

### 3.2 铺好的路：让正确的做法最省力

平台原语要比绕开它更容易使用：一行就能调用、有可以直接照抄的示例、名字容易被搜到。同时，让错误的做法变难：业务模块的构建依赖中，根本没有直接访问 Redis、HTTP、消息队列客户端的库，绕过原语的代码连编译都过不了。

### 3.3 新增抽象要有理由

新增接口、新模块、新的外部依赖，都会长期增加系统的复杂度。它们应该是**计划内的决定**：在任务卡中声明，计划外的新增由人确认。一个模式出现第三次再抽象，而不是第一次就抽象。

### 3.4 熵只降不升

用**适应度函数**（fitness function）度量架构的健康程度：越界依赖数、循环依赖、重复率、废弃 API 的调用次数、平台原语的使用率等。这个概念来自《演进式架构》（Building Evolutionary Architectures）：用可以客观评估的方式，判断架构离目标有多远。每个指标都设棘轮基线，只能变好；同时安排持续的偿还，让存量真正减少。

### 3.5 改架构要一起改四处

架构不是不能改，但改的时候，四样东西必须在同一个 PR 中一起更新：ADR、模块定义、生成的门禁规则、上下文中的说明。只改了其中一部分，规则和现实就会脱节。

## 4. 工程机制

### 4.1 模块地图与依赖方向（问题 03）

**机制一：用 `modules.yaml` 描述模块。**

每个模块写明：代码在哪里、谁负责、哪些包是公开接口、允许依赖哪些模块、风险等级。示例以 Python 项目为例（Java 和 TypeScript 项目的规则见机制三）：

```yaml
# architecture/modules.yaml —— 模块边界的唯一来源，修改需要架构负责人审批
root_package: shop

modules:
  platform:
    package: shop.platform
    owner: "@your-org/platform-owners"
    risk: L3                        # 用于工作流 Phase 0 的风险定级
    public: [shop.platform.api]     # 其他模块只能通过这些包调用
    may_depend_on: []

  inventory:
    package: shop.inventory
    owner: "@your-org/inventory-team"
    risk: L3
    public: [shop.inventory.api]
    may_depend_on: [platform]

  payment:
    package: shop.payment
    owner: "@your-org/payment-team"
    risk: L3
    public: [shop.payment.api]
    may_depend_on: [platform]

  order:
    package: shop.order
    owner: "@your-org/order-team"
    risk: L2
    public: [shop.order.api]
    may_depend_on: [platform, inventory, payment]

  cms:
    package: shop.cms
    owner: "@your-org/content-team"
    risk: L1
    public: [shop.cms.api]
    may_depend_on: [platform]
```

**三条规则：**

1. 一个模块只能依赖 `may_depend_on` 中列出的模块。
2. 跨模块的调用只能通过对方的 `public` 包，不能直接引用内部的类、仓储或表结构。
3. 模块之间的依赖关系不能有环。

**机制二：用脚本检查依赖，并从同一份定义生成其他规则。**

下面的脚本读取 `modules.yaml`，检查 Python 代码中的 `import` 是否违反这三条规则，并能生成 import-linter 配置、上下文中的模块地图和 CODEOWNERS。检查结果使用质量门禁 7.1 节的结构化报错格式：

```python
#!/usr/bin/env python3
"""从 architecture/modules.yaml 检查模块依赖，并生成门禁配置、模块地图和 CODEOWNERS。

用法：
  python arch.py check architecture/modules.yaml src > arch-findings.json
  python arch.py gen-importlinter architecture/modules.yaml > .importlinter
  python arch.py gen-context architecture/modules.yaml > docs/generated/module-map.md
  python arch.py gen-codeowners architecture/modules.yaml > .github/CODEOWNERS.modules
依赖：PyYAML。check 只分析 Python 的 import 语句；
动态导入（importlib、字符串形式的导入）无法被发现。
"""
import argparse
import ast
import json
import pathlib
import sys

import yaml


def load(path):
    with open(path, encoding="utf-8") as f:
        return yaml.safe_load(f)


def owner_module(name, modules):
    """按最长前缀，返回包名所属的模块；不属于任何模块（标准库、第三方库）时返回 None。"""
    best = None
    for mod, conf in modules.items():
        pkg = conf["package"]
        if (name == pkg or name.startswith(pkg + ".")) and (
                best is None or len(pkg) > len(modules[best]["package"])):
            best = mod
    return best


def is_public(name, conf):
    return any(name == p or name.startswith(p + ".") for p in conf.get("public", []))


def find_cycle(modules):
    """在声明的依赖关系中查找环，返回环上的模块列表；没有环时返回 None。"""
    state, stack = {}, []

    def visit(mod):
        state[mod] = "visiting"
        stack.append(mod)
        for dep in modules[mod].get("may_depend_on", []):
            if state.get(dep) == "visiting":
                return stack[stack.index(dep):] + [dep]
            if dep in modules and dep not in state:
                cycle = visit(dep)
                if cycle:
                    return cycle
        state[mod] = "done"
        stack.pop()
        return None

    for mod in modules:
        if mod not in state:
            cycle = visit(mod)
            if cycle:
                return cycle
    return None


def module_name(path, src_root):
    parts = list(path.relative_to(src_root).with_suffix("").parts)
    if parts[-1] == "__init__":
        return ".".join(parts[:-1]), True
    return ".".join(parts), False


def imported_names(tree, current, is_package):
    """返回 (行号, 被导入的完整名称) 列表；相对导入解析为绝对名称。"""
    base = current.split(".") if is_package else current.split(".")[:-1]
    names = []
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            names.extend((node.lineno, alias.name) for alias in node.names)
        elif isinstance(node, ast.ImportFrom):
            if node.level:
                prefix = base[:len(base) - node.level + 1] if node.level > 1 else base
                target = ".".join(prefix + ([node.module] if node.module else []))
            else:
                target = node.module or ""
            names.extend((node.lineno, f"{target}.{alias.name}" if alias.name != "*" else target)
                         for alias in node.names)
    return names


def finding(gate, path, line, violation, fix, severity="HARD_BLOCK"):
    return {"gate": gate, "severity": severity, "file": f"{path}:{line}",
            "violation": violation, "required_fix": fix}


def check(manifest, src_root):
    modules = manifest["modules"]
    findings = []
    for mod, conf in modules.items():
        for dep in conf.get("may_depend_on", []):
            if dep not in modules:
                findings.append(finding("ARCH_MAP_000", "architecture/modules.yaml", 0,
                                        f"{mod} 声明依赖了不存在的模块 {dep}", "修正 modules.yaml"))
    cycle = find_cycle(modules)
    if cycle:
        findings.append(finding("ARCH_MAP_003", "architecture/modules.yaml", 0,
                                "模块依赖存在环：" + " -> ".join(cycle),
                                "调整模块划分或依赖方向，并记录 ADR"))
    root = manifest.get("root_package")
    for path in sorted(pathlib.Path(src_root).rglob("*.py")):
        current, is_package = module_name(path, pathlib.Path(src_root))
        own = owner_module(current, modules)
        if own is None:
            if root and (current == root or current.startswith(root + ".")) and current != root:
                findings.append(finding("ARCH_MAP_004", path, 1,
                                        f"{current} 不属于 modules.yaml 中的任何模块",
                                        "放进已有模块，或在 modules.yaml 中登记新模块（需要人工确认）",
                                        severity="SIGN_OFF"))
            continue
        tree = ast.parse(path.read_text(encoding="utf-8"), filename=str(path))
        for line, name in imported_names(tree, current, is_package):
            target = owner_module(name, modules)
            if target is None or target == own:
                continue
            if target not in modules[own].get("may_depend_on", []):
                findings.append(finding(
                    "ARCH_MAP_001", path, line, f"{own} 不允许依赖 {target}（导入了 {name}）",
                    f"通过允许的依赖完成，或在 ADR 中论证后修改 {own}.may_depend_on"))
            elif not is_public(name, modules[target]):
                findings.append(finding(
                    "ARCH_MAP_002", path, line, f"{own} 绕过了 {target} 的公开接口（导入了 {name}）",
                    f"改为使用 {', '.join(modules[target].get('public', []))} 中的接口"))
    return findings


def gen_importlinter(manifest):
    modules = manifest["modules"]
    lines = ["# 由 architecture/modules.yaml 生成，请勿手工修改", "[importlinter]",
             f"root_package = {manifest['root_package']}", ""]
    for mod, conf in modules.items():
        forbidden = [modules[m]["package"] for m in modules
                     if m != mod and m not in conf.get("may_depend_on", [])]
        if not forbidden:
            continue
        deps = ", ".join(conf.get("may_depend_on", []))
        lines += [f"[importlinter:contract:{mod}-dependencies]",
                  f"name = {mod} 只能依赖 {deps}" if deps else f"name = {mod} 不依赖其他模块",
                  "type = forbidden", "source_modules =", f"    {conf['package']}",
                  "forbidden_modules ="] + [f"    {p}" for p in forbidden] + [""]
    return "\n".join(lines)


def gen_context(manifest):
    rows = ["<!-- 由 architecture/modules.yaml 生成，请勿手工修改 -->", "",
            "| 模块 | 包 | 负责人 | 风险 | 公开接口 | 允许依赖 |", "| --- | --- | --- | --- | --- | --- |"]
    for mod, c in manifest["modules"].items():
        rows.append(f"| {mod} | `{c['package']}` | {c['owner']} | {c.get('risk', '')} | "
                    f"{', '.join(f'`{p}`' for p in c.get('public', []))} | "
                    f"{', '.join(c.get('may_depend_on', [])) or '—'} |")
    return "\n".join(rows) + "\n"


def gen_codeowners(manifest, src_prefix="src"):
    lines = ["# 由 architecture/modules.yaml 生成，请勿手工修改",
             "/architecture/ @your-org/architecture-owners"]
    for c in manifest["modules"].values():
        lines.append(f"/{src_prefix}/{c['package'].replace('.', '/')}/ {c['owner']}")
    return "\n".join(lines) + "\n"


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("command", choices=["check", "gen-importlinter", "gen-context", "gen-codeowners"])
    parser.add_argument("manifest")
    parser.add_argument("src", nargs="?", default="src")
    args = parser.parse_args()
    manifest = load(args.manifest)
    if args.command == "check":
        findings = check(manifest, args.src)
        json.dump(findings, sys.stdout, ensure_ascii=False, indent=2)
        sys.exit(1 if any(f["severity"] == "HARD_BLOCK" for f in findings) else 0)
    output = {"gen-importlinter": gen_importlinter, "gen-context": gen_context,
              "gen-codeowners": lambda m: gen_codeowners(m, args.src)}[args.command](manifest)
    sys.stdout.write(output)


if __name__ == "__main__":
    main()
```

生成的 import-linter 配置可以直接用 `lint-imports` 运行，作为第二道检查；契约的更多选项以 import-linter 的文档为准。“只能通过公开接口调用”这一条，import-linter 的 forbidden 契约不好直接表达，由上面的 `check` 负责。

**机制三：Java 和 TypeScript 项目，从同一份定义生成对应的规则。**

Java 项目用 ArchUnit。每个模块生成一条“只能依赖这些包”的规则，再加一条全局的无环规则：

```java
// 由 architecture/modules.yaml 生成，请勿手工修改
@AnalyzeClasses(packages = "com.example")
class ModuleBoundariesTest {

    @ArchTest
    static final ArchRule order_dependencies =
        classes().that().resideInAPackage("com.example.order..")
            .should().onlyDependOnClassesThat().resideInAnyPackage(
                "com.example.order..",
                "com.example.platform.api..",
                "com.example.inventory.api..",
                "com.example.payment.api..",
                "java..", "javax..", "org.springframework..")   // 第三方包的白名单按项目调整
            .because("order 只能通过公开接口依赖 platform、inventory、payment");

    @ArchTest
    static final ArchRule no_module_cycles =
        slices().matching("com.example.(*)..").should().beFreeOfCycles();
}
```

TypeScript 项目用 dependency-cruiser，在 `forbidden` 中为每个模块生成规则，例如：

```json
{
  "forbidden": [
    {
      "name": "order-only-public-apis",
      "comment": "由 architecture/modules.yaml 生成：order 只能通过公开接口依赖 platform、inventory、payment",
      "severity": "error",
      "from": { "path": "^src/order/" },
      "to": {
        "path": "^src/(?!order/)",
        "pathNot": "^src/(platform|inventory|payment)/api/"
      }
    }
  ]
}
```

规则怎么写，以 ArchUnit 和 dependency-cruiser 的文档为准；重要的是它们**都从 `modules.yaml` 生成**，CI 检查生成结果与定义是否一致（4.5）。

**机制四：查找依赖时，用模块地图代替猜测。** 生成的模块地图放进上下文工程的 C0 索引地图和 C1 中（第 5 节）；需要知道“谁在调用我”时，使用代码导航工具（上下文工程 4.3 节）。问题 03 的治理要点中提到的“服务依赖清单和接口契约文件”，对应的是 `modules.yaml` 管不到的跨服务依赖，见需求工程 4.3 节的依赖图谱。

### 4.2 平台原语与铺好的路（问题 10，兼顾 08）

问题 10 的治理要点解决了“已有的实现找不到”：统一命名、加标识、自动生成组件清单。本节解决另一半：**找到了，Agent 也愿意用**。

**机制一：原语的设计原则。**

| 原则 | 含义 | 反例 |
| --- | --- | --- |
| 一件事只有一种明显的做法 | 同一个能力只提供一个原语，旧的替代写法标记废弃 | 项目里有三个 HTTP 客户端封装，Agent 不知道用哪个 |
| 正确的用法最短 | 最常见的场景一行就能写完，默认值就是安全的值 | 用原语要写十几行配置，自己写一个反而更快 |
| 名字容易被搜到 | 用业务和技术上最直白的名字，统一后缀和标识 | 原语叫 `Coordinator`，没人想到“幂等消费”要搜这个词 |
| 有可以直接照抄的示例 | 每个原语都配一个最小的、能运行的示例 | 只有接口签名，没有示例 |
| 失败时给出明确的指引 | 用错时的报错直接说明应该怎么改 | 用错时只抛一个通用异常 |

**机制二：每个原语一页说明，作为 C1 的一部分。**

````markdown
# 原语：IdempotentConsumer

ref: src/platform/messaging/IdempotentConsumer.java#IdempotentConsumer
review_by: 2027-03-01
负责人：@your-org/platform-owners

## 解决什么
同一条消息被重复投递时，只处理一次。

## 什么时候用
所有消息消费者都必须继承它（门禁规则：consumers_must_be_idempotent）。

## 怎么用（直接照抄）
```java
@Component
public class PaymentSucceededConsumer extends IdempotentConsumer<PaymentSucceeded> {
    @Override
    protected void handle(PaymentSucceeded event) {
        orderApi.markPaid(event.orderId());
    }
}
```

## 不要这样写
- 自己维护去重表
- 在 handle 里先查询是否处理过，再写入（并发时会重复处理）
````

`ref:` 锚点和 `review_by` 由上下文工程 6.2 节的保鲜检查负责：原语改名或删除时，说明会被发现过期。原语说明放在 `docs/primitives/`，由组件清单（问题 10）索引，按路径挂载到相关目录的 C1 中。

**机制三：让错误的路变难——构建依赖隔离。**

最有力的“铺好的路”，是让绕开原语的代码根本无法编译：业务模块的构建依赖中，不直接引入 Redis、HTTP、消息队列等客户端库，只有平台模块引入它们。以 Gradle 为例：

```groovy
// platform/build.gradle：平台模块自己使用这些库，但不对外暴露
plugins { id 'java-library' }
dependencies {
    api project(':platform-api')                        // 对外暴露的只有原语接口
    implementation 'org.springframework.data:spring-data-redis'
    implementation 'com.squareup.okhttp3:okhttp'
    implementation 'org.apache.kafka:kafka-clients'
}

// order/build.gradle：业务模块只能看到平台原语
dependencies {
    implementation project(':platform')
}
```

`java-library` 插件中，`implementation` 声明的依赖不会出现在使用方的编译类路径上，`api` 声明的才会。这样，订单模块的代码里写 `new OkHttpClient()`，编译就会失败。版本号和插件配置按项目调整。

Python 和 TypeScript 项目做不到编译期隔离，可以用 4.1 的依赖规则，禁止业务模块直接导入这些第三方库（把它们当作只允许 platform 依赖的“模块”）。

**机制四：脚手架。** 常见的新增场景（新的消息消费者、新的接口、新的定时任务）提供生成命令，例如 `make new-consumer NAME=PaymentSucceeded`，生成的代码已经使用了正确的原语。Agent 和人都会优先使用最省事的方式，脚手架让正确的方式成为最省事的方式。

**机制五：度量原语的使用。**

| 指标 | 怎么统计 |
| --- | --- |
| 原语使用率 | 使用原语的调用点，占同类调用点（原语 + 被门禁拦下或豁免的手写实现）的比例 |
| 手写次数 | 架构禁忌门禁（质量门禁 5.2.3）和重复检测中，“没用原语、自己实现”的命中次数 |
| 原语缺口 | 被豁免的手写实现中，有多少是因为原语不支持这个场景 |

原语缺口说明原语本身需要改进。如果一个原语经常被绕开，先检查它是不是太难用，而不是只加更多禁令。

### 4.3 抽象预算（问题 08）

**机制一：在任务卡中声明计划新增的抽象。**

在工作流 6.1 节的任务卡中增加两项：

```yaml
# .agent/tasks/FEAT-2048.yaml（节选）
planned_abstractions: []               # 计划新增的接口、抽象类、新模块，需要写明理由
planned_dependencies: []               # 计划新增的外部依赖
```

**机制二：计划外的新增，转人工确认。**

下面的脚本扫描本次的 diff，找出新增的接口、抽象类、Python 的 `ABC` / `Protocol` 子类和新的外部依赖，与任务卡对比：不在计划内的，标记为需要人工确认（`SIGN_OFF`），而不是直接阻断。同一个脚本也检查问题 23 中“没有关联问题单的 TODO”，这一项直接阻断：

```python
#!/usr/bin/env python3
"""熵检查：计划外新增的抽象和外部依赖转人工确认；没有关联问题单的 TODO 直接阻断。

用法：git diff -U0 origin/main...HEAD | python check_entropy.py .agent/tasks/FEAT-2048.yaml > entropy.json
依赖：PyYAML。只识别 Java、TypeScript 的 interface / abstract class，以及 Python 继承 ABC 或 Protocol 的类；
依赖文件只识别 requirements*.txt、package.json、pom.xml、go.mod 中新增的行，结果是近似的。
"""
import fnmatch
import json
import re
import sys

import yaml

ABSTRACTION = [
    (("*.java", "*.kt"), re.compile(r"^\s*(?:public\s+|protected\s+)?(?:sealed\s+)?(?:interface|abstract\s+class)\s+(\w+)")),
    (("*.ts", "*.tsx"), re.compile(r"^\s*(?:export\s+)?abstract\s+class\s+(\w+)")),
    (("*.py",), re.compile(r"^\s*class\s+(\w+)\s*\([^)]*\b(?:ABC|Protocol)\b")),
]
DEPENDENCY = [
    (("requirements*.txt",), re.compile(r"^\s*([A-Za-z0-9_.\-]+)")),
    (("package.json",), re.compile(r'^\s*"([@\w./\-]+)"\s*:\s*"[\^~<>=]?\d')),
    (("pom.xml",), re.compile(r"<artifactId>([\w.\-]+)</artifactId>")),
    (("go.mod",), re.compile(r"^\s*(?:require\s+)?([\w.\-]+\.[a-z]{2,}/[\w./\-]+)\s+v\d")),
]
NOT_DEPENDENCIES = {"name", "version", "node", "npm", "yarn", "pnpm", "groupId"}   # 依赖文件中不是依赖的字段
TODO = re.compile(r"\b(TODO|FIXME|HACK|XXX)\b")
ISSUE = re.compile(r"\b[A-Z][A-Z0-9]+-\d+\b|#\d+\b")
HUNK = re.compile(r"^@@ -\d+(?:,\d+)? \+(\d+)(?:,\d+)? @@")


def basename(path):
    return path.rsplit("/", 1)[-1]


def match_rules(path, rules):
    return [regex for globs, regex in rules if any(fnmatch.fnmatch(basename(path), g) for g in globs)]


def scan(diff_text, card):
    planned_abs = set(card.get("planned_abstractions") or [])
    planned_deps = set(card.get("planned_dependencies") or [])
    findings, path, line_no = [], None, 0
    for raw in diff_text.splitlines():
        if raw.startswith("+++ "):
            path = raw[6:] if raw.startswith("+++ b/") else None
            continue
        m = HUNK.match(raw)
        if m:
            line_no = int(m.group(1))
            continue
        if not raw.startswith("+") or raw.startswith("+++") or path is None:
            continue
        text = raw[1:]
        for regex in match_rules(path, ABSTRACTION):
            hit = regex.search(text)
            if hit and hit.group(1) not in planned_abs:
                findings.append({"gate": "ENTROPY_001", "severity": "SIGN_OFF", "file": f"{path}:{line_no}",
                                 "violation": f"计划外新增的抽象 {hit.group(1)}",
                                 "required_fix": "说明现在是否已有第二个使用场景；确有必要时写进任务卡的 planned_abstractions"})
        for regex in match_rules(path, DEPENDENCY):
            hit = regex.search(text)
            if hit and hit.group(1) not in planned_deps and hit.group(1) not in NOT_DEPENDENCIES:
                findings.append({"gate": "ENTROPY_002", "severity": "SIGN_OFF", "file": f"{path}:{line_no}",
                                 "violation": f"计划外新增的依赖 {hit.group(1)}",
                                 "required_fix": "优先使用已有的依赖和平台原语；确有必要时写进 planned_dependencies"})
        if TODO.search(text) and not ISSUE.search(text):
            findings.append({"gate": "ENTROPY_003", "severity": "HARD_BLOCK", "file": f"{path}:{line_no}",
                             "violation": "新增的 TODO 没有关联问题单",
                             "required_fix": "创建问题单并在注释中写上编号，例如 TODO(ORDER-123)"})
        line_no += 1
    return findings


def main():
    with open(sys.argv[1], encoding="utf-8") as f:
        card = yaml.safe_load(f) or {}
    findings = scan(sys.stdin.read(), card)
    json.dump(findings, sys.stdout, ensure_ascii=False, indent=2)
    sys.exit(1 if any(f["severity"] == "HARD_BLOCK" for f in findings) else 0)


if __name__ == "__main__":
    main()
```

`SIGN_OFF` 的结果进入质量门禁 7.5 节的审查导航，由审查者追问：**现在有没有第二个使用场景？** 没有的话，先写成具体的实现，等同一个模式出现第三次再抽象。TypeScript 的 `interface` 常用来描述数据结构，默认不计入；是否纳入，按项目的写法决定。

**机制三：跟踪复杂度的变化。** 对本次改动的文件，统计圈复杂度、文件数和类数的变化，作为审查导航的一部分。目的不是设一个硬阈值，而是让“这个改动让系统变复杂了多少”变得可见。

### 4.4 技术债与熵预算（问题 23）

棘轮基线（质量门禁 7.3 节）保证债务不再增加，但只要没有人偿还，基线就只是一份记录。本节讲怎么让债务真正减少。

**机制一：技术债登记。** 每一个 TODO 都关联一个问题单（4.3 的脚本强制执行），问题单中写明：为什么现在先这样做、正确的做法是什么、影响范围。没有问题单的 TODO，几个月后就没有人知道它为什么在那里。

**机制二：废弃流程。** 要替换的旧实现，按固定的阶段推进：

| 阶段 | 做什么 | 由什么保证 |
| --- | --- | --- |
| 1. 标记废弃 | 加 `@Deprecated` 或同等标记，说明替代的原语，写 ADR | 代码审查 |
| 2. 禁止新增调用 | 门禁规则禁止新的调用点；存量调用记入棘轮基线 | 门禁 + 棘轮基线 |
| 3. 迁移 | 把存量调用迁移到新原语，拆成有边界的小任务 | 任务卡 + 测试 |
| 4. 删除 | 调用点为 0 后，删除旧实现，并删除对应的门禁规则 | 棘轮基线归零 |

这是绞杀者模式（Strangler Fig）的思路：新实现逐步替代旧实现，而不是一次性重写。

**机制三：迁移交给 Agent，但要有边界。** 把几十个调用点从旧实现迁移到新原语，是重复、机械、规则明确的工作，**AI 很适合在约束下做这类事**，这也是 AI 能帮忙偿还技术债的地方。前提是任务有边界：

```yaml
# .agent/tasks/MIGRATE-HTTPUTIL-01.yaml
task_id: MIGRATE-HTTPUTIL-01
goal: 把 order 模块中 LegacyHttpUtil 的调用迁移到平台原语 HttpClientFactory
allowed_paths:
  - src/order/*
forbidden_paths:
  - src/platform/*                    # 不修改原语本身
  - tests/spec-bdd/*
budget:
  max_changed_files: 10               # 每批最多 10 个文件，便于审查
done_when:
  - order 模块中 LegacyHttpUtil 的调用点为 0
  - 现有测试全部通过，行为不变
  - 棘轮基线中 deprecated-legacy-http-util 的计数相应下降
stop_when:
  - 某个调用点的行为无法用新原语等价实现（记录下来，交给人决定）
```

一批迁移一个模块、一个 PR。行为不变由现有测试和契约测试保证；某个调用点无法等价迁移，说明新原语缺了能力（4.2 的“原语缺口”），停下来交给人。

**机制四：适应度函数看板。** 把架构健康程度写成一组可以持续统计的指标，每次合并后更新，并保留趋势：

| 适应度函数 | 数据来源 | 期望 |
| --- | --- | --- |
| 越界依赖数 | 4.1 的依赖检查 | 0 |
| 模块依赖环 | 4.1 的依赖检查、ArchUnit 无环规则 | 0 |
| 未登记的代码 | 4.1 的依赖检查（不属于任何模块的包） | 0 |
| 废弃 API 的调用点 | 棘轮基线 | 持续下降，直到归零 |
| 平台原语使用率 | 4.2 机制五 | 持续上升 |
| 新增代码的重复率 | 质量门禁 5.2.1 | 不高于基线 |
| 带问题单的 TODO 数量和平均存在时间 | 技术债登记 | 数量不增加，存在时间不增长 |

这些指标的计数，可以和质量门禁 7.3 节的棘轮基线放在同一份文件中，由同一个检查脚本保证只降不升。

**机制五：定期偿还。** 在迭代中固定预留一部分时间偿还技术债，优先处理适应度函数中退化最快的项；偿还工作按机制三拆成 Agent 可以执行的迁移任务。

### 4.5 架构演进

架构不是一成不变的。边界划错了、业务变了，就需要调整。本节保证调整的时候，规则和现实一起变。

**机制一：ADR 模板中加入“需要同步更新的约束”。** 以 Michael Nygard 提出的 ADR 格式（标题、状态、背景、决策、后果）为基础，增加一节：

```markdown
# ADR-0042：库存预占从订单模块迁移到库存模块

## 状态
已接受（2026-11-24）

## 背景
订单模块直接读写库存预占表，库存规则变化时需要同时修改两个模块。

## 决策
库存预占由库存模块负责，通过 shop.inventory.api.ReservationApi 对外提供。

## 后果
- 订单模块不再访问库存的内部包；迁移期间旧的直接访问记入棘轮基线
- 库存模块的公开接口增加一个

## 需要同步更新的约束
- [x] architecture/modules.yaml：inventory.public 增加 shop.inventory.api.reservation
- [x] 重新生成门禁规则、模块地图和 CODEOWNERS
- [x] 门禁：新增规则禁止 order 访问 shop.inventory.repository
- [x] 上下文：更新 src/order/AGENTS.md 中库存相关的说明
- [x] 迁移任务：MIGRATE-RESERVATION-01、02
```

**机制二：改边界的 PR，四样东西一起改。** 修改 `modules.yaml` 的 PR，必须同时包含：ADR、重新生成的规则和模块地图、相关目录的上下文说明。`architecture/` 目录由 CODEOWNERS 指定架构负责人审批。

**机制三：CI 检查生成结果与定义一致。** 生成的文件不允许手工修改，CI 重新生成一遍并对比：

```bash
python arch.py gen-importlinter architecture/modules.yaml | diff -u .importlinter -
python arch.py gen-context architecture/modules.yaml | diff -u docs/generated/module-map.md -
python arch.py gen-codeowners architecture/modules.yaml | diff -u .github/CODEOWNERS.modules -
```

任何一项有差异，说明有人手工改了生成的文件，或者改了 `modules.yaml` 却没有重新生成，CI 失败。GitHub 只读取一个 CODEOWNERS 文件，所以 `.github/CODEOWNERS.modules` 需要由脚本合并进 `.github/CODEOWNERS`，合并后的文件同样做一致性检查。

## 5. 串联：一份定义，多处生成

前几篇分别用到了模块边界：门禁里的架构规则、上下文里的分层边界和路径规则、工作流里的风险定级、CODEOWNERS。它们应该来自同一份定义：

```text
                       architecture/modules.yaml  +  docs/adr/
                                    │
     ┌───────────────┬──────────────┼──────────────┬────────────────┐
     ▼               ▼              ▼              ▼                ▼
  门禁规则        上下文          CODEOWNERS     风险定级规则      组件与原语清单
  import-linter   C0 索引地图                    routing-rules     docs/primitives/
  ArchUnit        C1 路径规则                    的 L3 路径        docs/components.md
  dependency-     C2 模块图谱
  cruiser
```

| 生成物 | 用在哪里 | 详见 |
| --- | --- | --- |
| 依赖规则（import-linter、ArchUnit、dependency-cruiser） | 质量门禁 Layer 1 的架构适应度门禁 | 本文 4.1；质量门禁 5.2.3 |
| 模块地图 | C0 的索引地图、C1 的按路径规则、C2 的模块图谱 | 上下文工程 4.1–4.3 |
| CODEOWNERS | 平台层变更锁、模块负责人审批 | 质量门禁 5.2.3 |
| 风险定级规则中的 L3 路径 | 工作流 Phase 0 的分级路由 | 工作流 5.1 |
| 组件与原语清单 | Agent 检索已有实现、C1 中的原语说明 | 本文 4.2；问题 10 |

并不是所有内容都能从 `modules.yaml` 生成：

- 风险定级规则中按代码内容判断的部分（例如新增 `synchronized`、`ThreadLocal`，见工作流 5.1 节），仍然在 `routing-rules.yaml` 中手写；本文的 `arch.py` 也还没有生成定级路径的命令，需要时可以仿照 `gen-codeowners` 实现。
- 组件清单按命名约定从代码生成（问题 10），原语说明页（4.2）是手写的。

改一次 `modules.yaml`，能生成的部分一起变，由 4.5 的一致性检查保证没有遗漏；需要手写的部分，在 ADR 的“需要同步更新的约束”中逐项确认。这样，“架构”就不再是一张会过时的图，而是一份在每次合并时都被执行的定义。

## 6. 示意案例：订单模块需要查询库存

下面是一个示意场景，用来说明这些约束如何配合，不是实际发生的情况。

**任务：** 下单前检查库存是否充足，订单模块属于 L2。

**Agent 最顺手的写法：**

1. 直接导入 `shop.inventory.repository.StockRepository` 查询库存表；
2. 写一个带重试的 HTTP 调用工具类，调用外部的仓储系统；
3. 为“库存检查”新增 `InventoryChecker` 接口、一个实现类和一个工厂。

**各层约束起到的作用：**

| 约束 | 起到的作用 |
| --- | --- |
| 上下文：C1 中的模块地图和原语说明 | Agent 读到订单模块允许依赖 inventory，但只能通过 `shop.inventory.api`；HTTP 调用应使用 `HttpClientFactory` |
| 依赖检查（ARCH_MAP_002） | 第一版代码导入了 `shop.inventory.repository`，被判定为绕过公开接口，诊断指向 `shop.inventory.api.StockApi` |
| 构建依赖隔离 | 第二版代码直接创建 HTTP 客户端，订单模块的编译类路径上没有这个库，编译失败 |
| 熵检查（ENTROPY_001） | `InventoryChecker` 接口不在任务卡的 `planned_abstractions` 中，转人工确认；审查者问“有没有第二个实现”，答案是没有，改为直接调用 `StockApi` |
| 适应度函数 | 合并后，越界依赖数仍为 0，原语使用率不变，没有新增重复代码 |

另一个场景：项目中有 40 处调用废弃的 `LegacyHttpUtil`。按 4.4 的废弃流程，先用门禁禁止新增调用，再拆成 4 个迁移任务，每个任务处理一个模块、不超过 10 个文件，由 Agent 完成迁移、人审查。4 个 PR 合并后，棘轮基线归零，删除旧实现和对应的门禁规则。

## 7. 一个最小对照实验：验证这些约束是否有效

### 7.1 实验设计

在一个按 `modules.yaml` 划分好模块的测试仓库中，设计 4 类、共 12 个诱发任务：

| 类型 | 诱发的问题 | 数量 |
| --- | --- | --- |
| 旁边有现成的原语 | 自己实现一遍更顺手 | 3 |
| 可以绕过公开接口 | 直接访问其他模块的内部类更快 | 3 |
| 容易过度抽象 | 需求描述中出现“以后可能会扩展” | 3 |
| 需要迁移 | 把一批废弃调用迁移到新原语 | 3 |

分三组运行：

- **A 组（只靠说明文件）：** 在说明文件中写明架构规则，没有门禁。
- **B 组（手写门禁规则）：** 有门禁，但规则分别手写，不从统一的定义生成，也没有构建依赖隔离和原语说明。
- **C 组（本文的做法）：** 规则由 `modules.yaml` 生成，有构建依赖隔离、原语说明、抽象预算和有边界的迁移任务。

每个任务每组重复 3 次，固定模型版本、工具集合和仓库版本。

### 7.2 记录指标

| 指标 | 说明 |
| --- | --- |
| 原语使用率 | 应该使用原语的地方，实际使用了原语的比例 |
| 越界依赖数 | 合并后仍然存在的越界依赖和绕过公开接口的调用 |
| 新增抽象数 | 新增的接口和抽象类中，只有一个实现的数量 |
| 迁移完成率与行为回归 | 迁移任务中调用点被迁移的比例，以及迁移后失败的测试数 |
| 首次通过率 | 第一次提交就通过架构门禁的比例 |
| 规则不一致次数 | B 组中，不同地方的规则互相矛盾或遗漏的次数 |

**当前这里给出的是实验方案，不是实验结果。** 在实际运行并记录数据之前，不应声称这些约束已经降低了架构腐化的速度。

### 7.3 如何解释结果

- C 组的原语使用率明显高于 B 组：铺好的路和构建隔离起了作用，而不只是禁令。
- B 组出现规则不一致：说明规则应该从统一的定义生成。
- C 组首次通过率高，但新增抽象数没有下降：抽象预算的人工确认流于形式，需要审查时认真追问第二个使用场景。
- 迁移任务出现行为回归：迁移前的测试覆盖不足，或者新原语和旧实现的行为并不等价，需要先补测试。

## 8. 局限与权衡

**模块地图需要维护。** 模块划分、公开接口和依赖关系，都需要有人负责；模块太细，`modules.yaml` 本身会变成负担。

**边界划错时，约束会变成阻碍。** 可执行的架构，会把错误的划分也严格执行下去。边界需要按 4.5 的流程定期复核和调整。

**抽象预算可能被机械执行。** 有些抽象确实应该提前做（例如明确的扩展点）；预算的目的是让抽象成为有理由的决定，而不是禁止抽象。

**废弃流程会拖慢迭代。** 标记、禁止、迁移、删除，每一步都需要时间；只适用于值得替换的旧实现。

**构建依赖隔离不是每种语言都能做到。** 动态语言只能依赖导入检查，反射和动态导入仍然可能绕过。

**依赖检查有盲区。** 动态导入、反射、通过配置注入的依赖，静态分析发现不了。

## 9. 常见问题诊断表

| 症状 | 可能原因 | 优先处理方式 |
| --- | --- | --- |
| 改了一个模块，另一个模块出错 | 绕过公开接口，直接依赖了内部实现 | 模块地图声明公开接口；依赖检查拦截绕过 |
| 模块之间互相调用，越来越难拆分 | 依赖方向没有约束，出现了环 | `may_depend_on` 声明依赖方向；无环规则 |
| 同一个能力在多处被重新实现 | 原语找不到，或者太难用 | 原语说明与示例；构建依赖隔离；统计原语缺口 |
| 平台原语经常被绕开 | 原语不支持某些场景 | 先改进原语，再加禁令 |
| 代码里接口和工厂越来越多 | 抽象没有理由也没有预算 | 任务卡声明计划内的抽象；计划外的转人工确认 |
| TODO 越来越多，没人知道为什么在那里 | TODO 没有关联问题单 | 熵检查阻断没有问题单的 TODO |
| 废弃的旧实现一直删不掉 | 只禁止新增，没有安排迁移 | 废弃流程；把迁移拆成有边界的 Agent 任务 |
| 门禁规则、架构图和说明文件互相矛盾 | 规则在多处手写 | 从 `modules.yaml` 统一生成；CI 检查一致性 |
| 架构调整后，Agent 仍然按旧的边界写代码 | 上下文没有同步更新 | ADR 的同步更新清单；改边界的 PR 必须同时更新上下文 |
| 新目录里的代码不受任何规则约束 | 没有登记到模块地图 | 依赖检查报告未登记的代码 |

## 10. 经验总结

1. **AI 只看局部，架构是整体。** 每一次“局部最优”都合理，累积起来就是腐化；AI 放大了这种速度。
2. **架构必须可执行。** 边界、公开接口和依赖方向写成机器可读的定义，由门禁在合并前检查。
3. **一份定义，多处生成。** 门禁规则、上下文、CODEOWNERS 和风险定级都从 `modules.yaml` 生成，CI 检查一致性。
4. **让正确的路最省力。** 原语一行就能用、有可照抄的示例、名字容易搜到；让错误的路编译不过。
5. **原语经常被绕开，先改原语。** 统计原语缺口，比只加禁令更有效。
6. **新增抽象要有理由。** 计划内的写进任务卡，计划外的由人确认；第三次出现再抽象。
7. **熵只降不升，而且要主动偿还。** 适应度函数加棘轮基线，配合废弃流程和定期偿还。
8. **让 AI 帮忙还债。** 有边界的机械迁移，正是 AI 擅长、也最安全的工作。
9. **改架构要一起改四处。** ADR、模块定义、生成的规则和上下文，在同一个 PR 中更新。

---

## 延伸阅读

- [AI Coding 常见问题与治理要点](../01-problems/01-ai-coding-common-problems.md)：本文对应的问题定义
- [AI Coding 工作流](../06-workflow/01-ai-coding-workflow.md)：平台层与业务层的边界、任务卡、风险定级
- [AI Coding 代码质量门禁](../07-quality-gate/01-ai-coding-quality-gate.md)：架构禁忌门禁、结构化报错与棘轮基线
- [AI Coding 上下文工程](../08-context/01-ai-coding-context.md)：C0 索引地图、按路径挂载的规则与保鲜检查
- [AI Coding 运行可靠性](../04-runtime/01-ai-coding-runtime-reliability.md)：幂等、锁与缓存的平台原语
- Michael Nygard, [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions.html)：架构决策记录（ADR）
- Martin Fowler, [Strangler Fig](https://martinfowler.com/bliki/StranglerFigApplication.html)：逐步替换旧系统的方式
- Neal Ford, Rebecca Parsons, Patrick Kua, [Building Evolutionary Architectures](http://evolutionaryarchitecture.com/)：适应度函数与演进式架构
- [ArchUnit](https://www.archunit.org/)、[import-linter](https://import-linter.readthedocs.io/)、[dependency-cruiser](https://github.com/sverweij/dependency-cruiser)：Java、Python、JavaScript / TypeScript 的依赖规则工具

> **最后一句：AI 让写代码变得几乎没有成本，也让“偷一步懒”几乎没有成本。架构约束的价值，不是让 AI 什么都不能做，而是让正确的路成为最省力的路，让错误的路走不通。**
