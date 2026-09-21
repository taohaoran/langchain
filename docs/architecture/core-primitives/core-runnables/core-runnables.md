# 可组合执行单元（Runnable）（core-runnables）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「LangChain Expression Language (LCEL) 的可组合执行单元抽象」，不重复展开消息类型（见 `../core-messages/core-messages.md`）、回调追踪（见 `../core-callbacks-tracers/core-callbacks-tracers.md`）等相邻叶子。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/runnables/`（15 个 `.py` 文件，约 1.43 万行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `Runnable` 抽象基类 | 可被 invoke/batch/stream/transform 的工作单元契约，泛型 `Runnable[Input, Output]` | `runnables/base.py:133` |
| 六种执行原语 | `invoke`/`ainvoke`（单输入）、`batch`/`abatch`（多输入并行）、`stream`/`astream`（流式产出） | `runnables/base.py:886`、`908`、`931`、`1194` |
| 流式事件协议 | `astream_log`（中间步骤日志流）、`astream_events`（标准化事件流 v2） | `runnables/base.py:1236`、`1368` |
| 组合运算符 | `__or__`/`__ror__` 把任意 Runnable/可调用/字典串成 `RunnableSequence`；字典字面量隐式构造 `RunnableParallel` | `runnables/base.py:648`、`697`、`724`（`pipe`） |
| 顺序组合 | `RunnableSequence`：逐 step 调用，上一步输出为下一步输入 | `runnables/base.py:3075`、`invoke` `3430` |
| 并行组合 | `RunnableParallel`：同一输入并发分发给多个命名分支，汇总为 dict | `runnables/base.py:3864`、`invoke` `4139` |
| 可调用包装 | `RunnableLambda`（包装同步/异步函数/生成器）、`RunnableGenerator`（包装流式生成器） | `runnables/base.py:4703`、`4399` |
| 绑定与装饰 | `bind`（固化 kwargs）、`with_config`、`with_listeners`/`with_alisteners`、`with_types`、`with_retry`、`map`、`with_fallbacks` | `runnables/base.py:1851`、`1885`、`1910`、`2101`、`2165`、`2188` |
| 回调感知执行包装 | `_call_with_config`/`_batch_with_config`/`_transform_stream_with_config` 统一注入回调管理器与配置上下文 | `runnables/base.py:2268`、`2360`、`2502` |
| 配置对象 | `RunnableConfig`（TypedDict）：tags/metadata/callbacks/run_name/max_concurrency/recursion_limit/configurable/run_id | `runnables/config.py:57` |
| 配置工具 | `ensure_config`/`patch_config`/`get_config_list`/`merge_configs`/`set_config_context` | `runnables/config.py:255`、`357`、`311`、`431`、`224` |
| 线程池执行 | `ContextThreadPoolExecutor`、`get_executor_for_config`（batch 默认并行化） | `runnables/config.py:607`、`660` |
| 重试 | `RunnableRetry`：基于 tenacity 的同步/异步指数退避 + 抖动 | `runnables/retry.py:48` |
| 降级 | `RunnableWithFallbacks`：主 Runnable 失败后按序回退到备选 | `runnables/fallbacks.py:37` |
| 条件分支 | `RunnableBranch`：按谓词条件路由到不同 Runnable | `runnables/branch.py:43` |
| 路由 | `RouterRunnable`：按 `key` 字符串查表分发到命名 Runnable | `runnables/router.py:46` |
| 透传与字段操作 | `RunnablePassthrough`、`RunnableAssign`（在原 dict 上增补字段）、`RunnablePick`（挑出字段） | `runnables/passthrough.py:74`、`352`、`passthrough.py` |
| 会话历史 | `RunnableWithMessageHistory`：自动注入/保存聊天历史到 runnable 链 | `runnables/history.py:39` |
| 运行期可配置 | `DynamicRunnable`/`RunnableConfigurableFields`/`RunnableConfigurableAlternatives`：运行时按 config 切换实现 | `runnables/configurable.py:50`、`317`、`474` |
| 图可视化 | `get_graph` + `graph.py`/`graph_ascii.py`/`graph_mermaid.py`/`graph_png.py`：导出链结构图 | `runnables/base.py:593`、`runnables/graph.py` |
| 序列化基类 | `RunnableSerializable`：继承 `Serializable`，使 Runnable 可 lc 序列化 | `runnables/base.py:2827` |
| 惰性导出面 | `__init__.py` 用 `__getattr__` + `import_attr` 惰性导入，避免顶层导入即加载全部 | `runnables/__init__.py:128` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Runnable[Input, Output]`（ABC） | `runnables/base.py:133` | 全栈核心能力缝。定义 `invoke`（抽象）与 batch/stream/transform 的默认实现；所有模型、prompt、parser、tool、retriever 均实现它 |
| `RunnableSerializable` | `runnables/base.py:2827` | `Serializable` + `Runnable` 多重继承，提供 `to_json` 与 `configurable_fields`/`configurable_alternatives` |
| `RunnableSequence` | `runnables/base.py:3075` | `|` 运算符产物；持有 `steps` 列表，串行执行 |
| `RunnableParallel` | `runnables/base.py:3864` | 字典字面量产物；并发执行命名分支，返回 `AddableDict` |
| `RunnableLambda` | `runnables/base.py:4703` | 把任意 sync/async 可调用适配成 Runnable；`deps` 为缓存属性 |
| `RunnableGenerator` | `runnables/base.py:4399` | 把生成器函数适配成流式 Runnable（transform/stream） |
| `RunnableBindingBase`（`RunnableBinding`） | `runnables/base.py` | 持有 `bound` + 绑定 kwargs 的装饰基类，bind/with_config/with_retry 均经它 |
| `RunnableConfig` | `runnables/config.py:57` | 贯穿全栈的配置 TypedDict；驱动运行时行为（并发、递归上限、回调、可配置字段） |
| `RunnableRetry` | `runnables/retry.py:48` | 装饰器，用 tenacity `Retrying`/`AsyncRetrying` 包裹 invoke/batch |
| `RunnableWithFallbacks` | `runnables/fallbacks.py:37` | 持有 `runnables`（主+备选）列表，失败回退 |
| `RunnableBranch` | `runnables/branch.py:43` | 持有 `(condition, runnable)` 元组列表，命中即执行 |
| `RouterRunnable` | `runnables/router.py:46` | 持有 `runnables: dict[key, Runnable]`，按输入 `key` 分发 |
| `RunnablePassthrough`/`RunnableAssign`/`RunnablePick` | `runnables/passthrough.py:74`/`352` | 恒等透传、增补字段、挑选字段 |
| `RunnableWithMessageHistory` | `runnables/history.py:39` | 用 `get_history` 回调在 invoke 前后读写会话历史 |
| `DynamicRunnable` 族 | `runnables/configurable.py:50` | 运行期 `_prepare` 动态解析出实际 Runnable |
| `ConfigurableField*` | `runnables/utils.py` | 声明式可配置字段规范（注册表式运行期开关） |
| `ensure_config`/`patch_config` | `runnables/config.py:255`/`357` | 把 None/部分配置补全为完整 `RunnableConfig`；局部打补丁并合并 tags/metadata |
| `ContextThreadPoolExecutor` | `runnables/config.py:607` | 在 worker 线程中传播配置上下文（contextvars） |

> 关键架构事实：抽象基类 `Runnable` 定义在消费方 `langchain_core`，而绝大多数具体实现（OpenAI 模型、Anthropic 模型、向量库等）位于 `libs/partners/*`，是典型的依赖倒置——宿主定义契约，实现方注册进来。

## 3. 关键调用链

**调用链一：`chain.invoke(x, config)` 顺序执行（`RunnableSequence.invoke`，`base.py:3430`）**

1. `config = ensure_config(config)` 补全配置（`config.py:255`）。
2. `callback_manager = get_callback_manager_for_config(config)` 取同步回调管理器（`config.py:563`）。
3. `run_manager = callback_manager.on_chain_start(...)` 开启根 run，记录 name/run_id（`base.py:3437`）。
4. 遍历 `self.steps`：每步 `patch_config(config, callbacks=run_manager.get_child(f"seq:step:{i+1}"))` 派生子回调（`base.py:3449`）。
5. `with set_config_context(config)` 在 contextvars 中下发配置，`context.run(step.invoke, input_, config)` 执行该步，输出成为下一步输入（`base.py:3452-3456`）。
6. 正常结束 `run_manager.on_chain_end(input_)`；任何 `BaseException` 走 `run_manager.on_chain_error(e)` 后 re-raise（`base.py:3458-3462`）。

**调用链二：用户子类用 `_call_with_config` 实现 `invoke`（`base.py:2268`）**

1. `ensure_config` → `get_callback_manager_for_config` → `on_chain_start`。
2. `patch_config(config, callbacks=run_manager.get_child())` 派生子配置。
3. `with set_config_context(child_config)` 后 `context.run(call_func_with_variable_args, func, input_, config, run_manager)`，把 run_manager 以变量参数注入用户函数。
4. 异常 → `on_chain_error` 并 re-raise；正常 → `on_chain_end(output)`。

**调用链三：`batch` 默认并行化（`Runnable.batch`，`base.py:931`）**

1. `configs = get_config_list(config, len(inputs))` 把单配置广播成配置列表（`config.py:311`）。
2. 单输入直接同步调用，省去线程池（`base.py:975`）。
3. 多输入用 `with get_executor_for_config(...) as executor: executor.map(invoke, inputs, configs)` 并发；`return_exceptions=True` 时把异常作为结果返回而非抛出（`base.py:966-979`）。

**调用链四：`with_retry` 重试（`RunnableRetry`，`retry.py:48`）**

1. `with_retry(stop_after_attempt=..., wait_exponential_jitter=...)`（`base.py:2101`）生成 `RunnableRetry` 装饰器包裹原 Runnable。
2. `_invoke` 内构造 tenacity `Retrying`/`AsyncRetrying`（`retry.py:152-155`），把用户 `invoke` 包进退避重试循环。
3. `_patch_config`/`_patch_config_list` 在重试之间合并配置，保留 run 上下文。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `RunnableConfig.tags` | 字符串标签，用于追踪过滤 | `config.py:80` |
| `RunnableConfig.metadata` | 附加元数据 dict，随 run 传播 | `config.py:86` |
| `RunnableConfig.callbacks` | `BaseCallbackHandler`/`BaseCallbackManager` 列表或管理器 | `config.py:92` |
| `RunnableConfig.run_name` | 覆盖本次 run 显示名 | `config.py:98` |
| `RunnableConfig.max_concurrency` | 并行执行上限；`None` 不限 | `config.py:103` |
| `RunnableConfig.recursion_limit` | 链递归深度上限（防无限递归） | `config.py:109` |
| `RunnableConfig.configurable` | 运行期可配置字段值 dict（供 `configurable_fields` 消费） | `config.py:115` |
| `RunnableConfig.run_id` | 显式指定 run 的 UUID | `config.py:124` |
| `with_retry(stop_after_attempt)` | 最大重试次数 | `base.py:2101`、`retry.py` |
| `wait_exponential_jitter` | 是否指数退避 + 抖动 | `retry.py:35`（`ExponentialJitterParams`） |
| `set_debug` 全局开关 | `globals.set_debug(True)` 开启全链调试输出 | `globals.py`（见 `core-serialization-cache-infra`） |

## 5. 错误与重试语义

- **回调包裹层**：`_call_with_config`/`RunnableSequence.invoke` 用 `try/except BaseException` 包住用户逻辑，无论同步抛错还是 async 异常都先 `on_chain_error(e)` 再 re-raise（`base.py:2310`、`3458`）。回调失败不吞业务异常。
- **batch 异常**：默认 `return_exceptions=False` 时任一输入失败即向 executor.map 传播；`return_exceptions=True` 时把每个异常对象填入结果列表对应位置，不中断其余（`base.py:966-972`）。
- **重试**：`RunnableRetry` 委托 tenacity `Retrying`（同步）/`AsyncRetrying`（异步），支持 `stop_after_attempt`、指数退避 + 抖动；重试仅包裹 invoke/batch，不包裹 stream（`retry.py:48-291`）。
- **降级**：`RunnableWithFallbacks.invoke` 先试主 Runnable，捕获异常后按 `runnables` 列表顺序尝试备选，全部失败才抛出最后一个异常（`fallbacks.py:165`）。
- **递归保护**：`recursion_limit` 在配置上下文里递减，链过深时抛错防无限递归（`config.py:109`）。
- **错误传播**：异常带原始 traceback 上抛，回调层只记录不转换；LangSmith 等外部追踪后端（不在本仓库源码内）负责把错误可视化。

## 6. 并发细节

- **异步默认桥接**：`ainvoke` 默认实现 `await run_in_executor(config, self.invoke, ...)`，把同步 invoke 丢到 asyncio 线程池；子类应覆盖为原生 async 以获得真正并发（`base.py:929`）。
- **batch 默认线程池**：`Runnable.batch` 用 `get_executor_for_config` 返回的 `ContextThreadPoolExecutor`，`executor.map` 并行（`base.py:978`、`config.py:660`）。`max_concurrency` 限制并发度。
- **上下文传播**：`ContextThreadPoolExecutor.submit/map` 把当前 contextvars（含 `set_config_context` 下发的配置与 run 上下文）复制到 worker 线程，保证子线程回调/配置不丢（`config.py:607-658`）。`set_config_context` 是 contextvars 上下文管理器（`config.py:224`）。
- **async batch**：`abatch` 用 asyncio 任务并发，受 `max_concurrency` 信号量约束；`abatch_as_completed` 按完成先后产出 `(index, output)`。
- **流式管道**：`RunnableSequence.stream` 用 `_transform_stream_with_config` 在步骤间通过生成器逐 chunk 传递，背压由生成器自然承担（`base.py:2502`、`3827`）。
- **无共享可变临界区**：Runnable 实例本身在调用间无状态（幂等），并发安全来自「每 invoke 独立 run_manager + contextvars 隔离」，而非全局锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `runnables/` 全部：Runnable ABC、组合原语、配置系统、重试/降级/分支/路由/透传/历史/可配置装饰器、图导出。

**Out-of-Scope（不在本仓库源码内）**
- 具体模型实现（OpenAI/Anthropic/Ollama 等）在 `libs/partners/*`；本叶子只定义它们必须实现的 `Runnable` 契约。
- 外部追踪后端 LangSmith（云服务）不在本仓库源码内；本叶子只经回调把事件交出去。
- 第三方 HTTP/LLM API 调用本身不在本仓库源码内。
- 消息类型体系（见 `core-messages`）、输出解析（见 `core-output-parsers`）、工具（见 `core-tools`）是被 Runnable 包装的相邻叶子，本叶子不展开。

## 8. 与相邻子系统交互

- **上游（用户 / 上层包）**：`langchain_v1`、`langchain_classic`、`libs/partners/*` 中的所有模型/检索器/解析器都实现 `Runnable`；用户用 `|` 把它们串成 LCEL 链后调 `invoke/batch/stream`。
- **本叶子 → 回调（`core-callbacks-tracers`）**：每个执行入口经 `get_callback_manager_for_config` 创建 run_manager，`on_chain_start/on_chain_error/on_chain_end` 是横切注入点；子步经 `run_manager.get_child()` 派生。
- **本叶子 → 配置上下文**：`set_config_context` 用 contextvars 把 RunnableConfig 隐式传给下游（如 tool 内部取回调）。
- **本叶子 → 历史（`core-documents-loaders` 的 chat_history）**：`RunnableWithMessageHistory` 经 `BaseChatMessageHistory` 抽象读写会话。
- **本叶子 → 工具（`core-tools`）**：`Runnable.as_tool()`（`base.py:2708`）把任意 Runnable 包装成 `BaseTool`，供模型调用。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：本叶子是 langchain-core 最典型的「抽象基类定义契约 + 组合运算符扩展点」能力缝——`Runnable` ABC 在消费方定义 `invoke`，全部实现方在 partners 侧；`__or__`/字典字面量是组合扩展点；`with_retry`/`with_fallbacks`/`with_listeners` 是可插拔装饰钩子链。
- **注册表与工厂**：`configurable_fields`/`configurable_alternatives`（`configurable.py`）是声明式运行期注册表，按 `RunnableConfig.configurable` 在 `_prepare` 时解析出实际实现；`RouterRunnable` 是字符串 key→Runnable 的显式路由表。
- **可选依赖与懒加载**：`runnables/__init__.py` 用模块级 `__getattr__` + `import_attr`（来自 `_import_utils`）实现符号惰性导入，`import langchain_core.runnables` 不立即加载 base.py 全部重实现（`__init__.py:128`）。
- **配置驱动静态分派**：`RunnableConfig` 是运行时行为总开关——`max_concurrency`、`recursion_limit`、`configurable`、`callbacks` 都驱动同一套代码在不同配置下走不同路径；`DynamicRunnable._prepare` 按 config 动态选实现，是框架级静态多态。
- **双轨/异步桥接**：每个原语都有 sync/async 双实现（`invoke`/`ainvoke`），默认 async 经 `run_in_executor` 桥接 sync，子类可覆盖为原生 async——这是纯 Python 框架常见的「性能路径 + 兜底路径」双轨。
- **图类型侧重**：用 architecture 表达 Runnable 协议与组合原语分层，用 sequence 表达 `RunnableSequence.invoke` 的调用链（含回调包裹）。本叶子无「隐藏在循环里的状态机」（流式是生成器管道而非显式状态机），故不补 lifecycle 图。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Runnable 协议与组合原语架构图 | `core-runnables-architecture.html` | architecture | standard（showcase 布局校验对「中心扇出」边的端点方向约束较严，降 standard；主路径清晰） |
| `RunnableSequence.invoke` 时序图（含回调包裹） | `core-runnables-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录（`core-runnables-architecture.json`、`core-runnables-sequence.json`）。架构图因中心节点向多个组合原语扇出、showcase 对端点侧方向校验严格而降 standard；时序图主路径清晰、通过 showcase。
