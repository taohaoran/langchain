# testing-infra 域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`libs/standard-tests/langchain_tests/`，branch `master`，commit `89252a8f`。

## 1. 域职责

standard-tests 是供各 partner 集成复用的标准化测试套件。它定义"标准测试契约"（抽象基类 + 必实现属性 +
特性开关），partner 包继承并填入被测模型即可获得一套统一的集成测试，并用守卫强制契约不被悄悄破坏。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 流程 | 职责一句话 |
|------|------|--------|------|-----------|
| standard-tests | [standard-tests.md](standard-tests/standard-tests.md) | [架构图](standard-tests/standard-tests-architecture.html) | [接入流程](standard-tests/standard-tests-workflow.html) | 标准化集成测试套件 |

## 3. 域级机制细节

- **标准测试机制**：partner 继承 `ChatModelIntegrationTests` 等基类，实现 `chat_model_class` 与 `chat_model_params`；`has_tool_calling` 等特性开关决定用例取舍。
- **防覆盖守卫**：`BaseStandardTests` 自动比对 partner 类与标准基类，禁止删除标准用例，禁止无 `xfail(reason=...)` 理由覆盖。
- **追踪桥接**：`_langsmith_plugin` 经 pytest11 entry point 把 CI 集成测试 trace 到 LangSmith（外部服务）。
- **本轮增强**：基线叶子仅 1 张架构图，本轮补第 2 张 workflow 图（partner 接入流程含契约守卫决策分支），并新增本域级架构图表达契约测试层在生态中的位置。

## 4. 域级图（合计 3 张 = 域总览架构图 1 张 + 代表性叶子图 2 张）

![standard-tests 契约测试层位置](testing-infra-domain-architecture.html)

本域为单叶域，域级图配额计入以下 3 张：

| 图 | 文件 | 类型 | 层次 |
|----|------|------|------|
| 契约测试层位置 | `testing-infra-domain-architecture.html` | architecture | 域级 |
| 标准测试套件架构图 | [standard-tests/standard-tests-architecture.html](standard-tests/standard-tests-architecture.html) | architecture | 叶子级 |
| partner 接入标准测试流程 | [standard-tests/standard-tests-workflow.html](standard-tests/standard-tests-workflow.html) | workflow | 叶子级 |

域级架构图表达该包作为"契约测试层"连接 partner 包、pytest 运行时、外部 API 服务与 LangSmith 追踪的位置；叶子级两张分别表达静态契约关系与带守卫决策分支的接入流程，三者语义不重复。
