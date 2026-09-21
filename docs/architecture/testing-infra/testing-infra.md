# testing-infra 域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，独立包 `libs/standard-tests/langchain_tests/`（21 个 py / 约 9820 行）。

## 1. 域职责

standard-tests 是供各 partner 集成复用的标准化测试套件。它定义"标准测试契约"（抽象基类 + 必实现属性 +
特性开关），partner 包继承并填入被测模型即可获得一套统一的集成测试，并用守卫强制契约不被悄悄破坏。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|-----------|
| standard-tests | [standard-tests.md](standard-tests/standard-tests.md) | [架构图](standard-tests/standard-tests-architecture.html) | 标准化集成测试套件 |

## 3. 域级机制细节

- **标准测试机制**：partner 继承 `ChatModelIntegrationTests` 等基类，实现 `chat_model_class` 与 `chat_model_params`；`has_tool_calling` 等特性开关决定用例取舍。
- **防覆盖守卫**：`BaseStandardTests` 自动比对 partner 类与标准基类，禁止删除标准用例，禁止无 `xfail(reason=...)` 理由覆盖。
- **追踪桥接**：`_langsmith_plugin` 经 pytest11 entry point 把 CI 集成测试 trace 到 LangSmith（外部服务）。
