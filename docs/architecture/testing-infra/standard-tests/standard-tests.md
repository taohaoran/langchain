# 标准化集成测试套件（standard-tests）

> 本文是 `testing-infra` 域下的叶子子系统文档。本独立包为各 partner 集成提供标准化测试接口，确保不同
> 厂商实现满足统一契约。
>
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，独立包目录 `libs/standard-tests/langchain_tests/`（21 个 py 文件，约 9820 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `BaseStandardTests` | 标准测试基类，含"不得无理由覆盖标准测试"守卫 | `langchain_tests/base.py:4` |
| `ChatModelIntegrationTests` | 聊天模型标准集成测试基类（invoke/stream/tool_calling/structured_output/多模态等） | `integration_tests/chat_models.py:194` |
| 嵌入标准测试 | `EmbeddingsIntegrationTests` | `integration_tests/embeddings.py` |
| 工具标准测试 | 工具集成测试 | `integration_tests/tools.py` |
| 向量库标准测试 | `VectorStoreIntegrationTests` | `integration_tests/vectorstores.py` |
| 检索器/缓存/存储/索引 | retrievers / cache / base_store / indexer | `integration_tests/` |
| 沙箱测试 | `sandboxes.py` | `integration_tests/sandboxes.py` |
| 单元测试套件 | unit_tests 下 chat_models/embeddings/tools 的无网单测 | `unit_tests/` |
| Pydantic 工具 | utils/pydantic 校验辅助 | `utils/pydantic.py` |
| 流生命周期工具 | utils/stream_lifecycle | `utils/stream_lifecycle.py` |
| VCR 配置 fixture | conftest 提供 `vcr_config` 录制回放 | `conftest.py:152` |
| LangSmith 追踪插件 | `_langsmith_plugin.py`，把 CI 运行 trace 到 LangSmith | `_langsmith_plugin.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `BaseStandardTests` | `base.py:4` | 防覆盖守卫基类 |
| `ChatModelIntegrationTests(ChatModelTests)` | `integration_tests/chat_models.py:194` | 聊天模型标准契约 |
| partner 必实现属性 | `chat_model_class` / `chat_model_params` | 指明被测模型类与初始化参数 |
| 特性开关属性 | `has_tool_calling` 等 | 声明被测模型支持的特性，按需跳过用例 |

## 3. 关键调用链

**调用链：partner 接入标准测试**

1. partner 包定义 `class TestMyChatModel(ChatModelIntegrationTests)`（`integration_tests/chat_models.py:208`）。
2. 子类实现 `chat_model_class` 与 `chat_model_params` 两个属性，指定被测模型与参数（`integration_tests/chat_models.py:210`）。
3. 按需覆写 `has_tool_calling` 等特性开关，框架据此选择运行/跳过用例（`integration_tests/chat_models.py:247`）。
4. pytest 自动收集 `test_invoke / test_stream / test_tool_calling / test_structured_output` 等标准用例。
5. `BaseStandardTests.test_no_overrides_DO_NOT_OVERRIDE` 自动运行：比对 partner 类与标准基类，禁止删除标准用例、禁止无 `xfail(reason=...)` 理由覆盖（`base.py:7`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `chat_model_class` | 子类必须实现，被测模型类 | `chat_models.py:224` |
| `chat_model_params` | 子类必须实现，初始化参数字典 | `chat_models.py:234` |
| `has_tool_calling` | 缺省按是否覆写 `bind_tools` 推断 | `chat_models.py:247` |
| `vcr_config` | session 级 VCR 录制回放配置 | `conftest.py:152` |

## 5. 错误与重试语义

- **违反契约即失败**：删除或无理由覆盖标准测试 → 守卫断言失败（`base.py:37,65`）。
- **预期失败**：partner 对暂不支持的用例须用 `@pytest.mark.xfail(reason="...")` 标注，reason 必填（`base.py:45`）。
- 集成测试真实调外部服务；VCR 录制后可离线回放。

## 6. 并发细节

- pytest 插件模型：`_langsmith_plugin` 经 `pytest11` entry point 自动注册，会话级 trace 上下文桥接（见项目 AGENTS.md 集成测试追踪说明）。
- 无自定义并发原语。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 全部标准测试基类、特性开关、守卫、工具与 fixture。

**Out-of-Scope（不在本仓库源码内）**
- 各 partner 具体模型实现（在 `libs/partners/`）。
- 外部 API 服务（测试目标）。

## 8. 与相邻子系统交互

- 上游 partner 包 → 本包：继承标准测试基类、实现两个属性。
- 本包 → 下游 pytest：作为 pytest 插件与测试基类被收集运行。
- 本包 → LangSmith：CI 集成测试 trace 上传（外部服务）。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **测试框架能力缝**：本包定义"标准测试接口"（抽象契约），partner 是实现方；`BaseStandardTests` 守卫强制契约不被悄悄破坏，是"契约测试"模式。
- **可选依赖**：VCR、LangSmith 为可选集成，缺省时优雅降级。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| 标准测试套件架构图 | `standard-tests-architecture.html` | architecture | standard |

JSON IR 源文件位于 `json/`。本叶子为测试框架，单张架构图表达契约关系即可。
