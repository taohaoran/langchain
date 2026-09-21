# 对话记忆（classic-memory）

> 本文是 `classic` 域下的叶子子系统文档（第二轮改进版）。域级总览见 `../classic.md`，本文只展开
> 经典包的 **对话记忆抽象与实现**，不展开 Chain 如何调用记忆（见 `../classic-chains/`）、消息
> schema（见 `../classic-callbacks-infra/` 的 `schema/`）。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `4492ad7a8`。
>
> 本轮改进：在基线架构图之外，新增"一次对话轮次的记忆读写"时序图，把 `prep_inputs` 注入、
> `save_context` 落盘、`BaseChatMessageHistory` 持久化三段交互画清楚。

## 1. 功能清单

记忆抽象让无状态的 Chain/Agent 在多次调用间保持对话历史。源码位于
`memory/`（39 文件）+ `base_memory.py`。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 记忆抽象基类 | `BaseMemory`：三方法契约 | `base_memory.py:30` |
| 聊天记忆基类 | `BaseChatMemory`：组合 `BaseChatMessageHistory` | `memory/chat_memory.py:28` |
| 缓冲记忆 | 全量保留历史消息 | `memory/buffer.py:24`（`ConversationBufferMemory`） |
| 字符串缓冲 | 以字符串而非消息对象保留 | `memory/buffer.py:105` |
| 窗口记忆 | 只保留最近 N 轮 | `memory/buffer_window.py` |
| 摘要记忆 | 用 LLM 滚动总结历史 | `memory/summary.py:96`（`SummarizerMixin`） |
| 摘要缓冲 | 摘要 + 近期原文混合 | `memory/summary_buffer.py` |
| token 缓冲 | 按 token 数裁剪 | `memory/token_buffer.py` |
| 组合记忆 | 多个记忆拼接 | `memory/combined.py:10` |
| 实体记忆 | 维护实体状态表 | `memory/entity.py:488` |
| 向量检索记忆 | 用检索器按相关性取历史 | `memory/vectorstore.py:26` |
| 只读共享记忆 | 只读视图，供多链共享 | `memory/readonly.py` |
| 简单记忆 | 直接透传 dict | `memory/simple.py` |
| 消息历史后端 | 21 种持久化后端 | `memory/chat_message_histories/` |

消息历史后端：`in_memory`、`file`、`redis`、`sql`、`mongodb`、`postgres`、`elasticsearch`、
`dynamodb`、`cosmos_db`、`neo4j`、`zep`、`streamlit`、`upstash_redis` 等。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseMemory(Serializable, ABC)` | `base_memory.py:30` | 能力缝契约：`memory_variables`/`load_memory_variables`/`save_context` |
| `BaseChatMemory` | `memory/chat_memory.py:28` | 持有 `chat_memory: BaseChatMessageHistory`，统一消息增删接口 |
| `BaseChatMessageHistory` | 来自 `langchain_core` | 消息记录存储契约（`add_user_message`/`add_ai_message`/`clear`） |
| `ConversationBufferMemory` | `memory/buffer.py:24` | 最简单实现：全量历史拼成字符串变量 |
| `SummarizerMixin` | `memory/summary.py:29` | 摘要逻辑混入：用 LLM 压缩旧对话 |
| `CombinedMemory` | `memory/combined.py:10` | 组合多个子记忆，合并变量 |
| `VectorStoreRetrieverMemory` | `memory/vectorstore.py:26` | 按向量相似度取相关历史 |

## 3. 关键调用链

**Chain 调用记忆（对应新增对话轮次时序图）**：

1. `Chain.prep_inputs`（`chains/base.py:521`）调 `memory.load_memory_variables(inputs)`，
   把历史变量注入输入 dict。
2. Chain 带历史 prompt 调 LLM 生成本轮响应。
3. `Chain.prep_outputs`（`chains/base.py:471`）调 `memory.save_context(inputs, outputs)`，
   落盘本轮问答。

**ConversationBufferMemory.load_memory_variables**（`memory/buffer.py:83`）：
从 `chat_memory.messages` 取全部消息 → 经 `get_buffer_string` 拼成 `"history"` 变量 →
返回 `{"history": ...}`。

**save_context**：把 user 输入 `add_user_message`、AI 输出 `add_ai_message` 追加到
`chat_memory`（持久化后端，`memory/chat_memory.py`）。

**异步**：`aload_memory_variables`/`asave_context` 默认经 `run_in_executor` 包装同步实现
（`base_memory.py:82`、`:102`）。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `memory_variables` | 抽象属性，返回该记忆对外暴露的变量名列表 | `base_memory.py:68` |
| `chat_memory` | `BaseChatMessageHistory`，默认 `InMemoryHistory` | `memory/chat_memory.py:39` |
| `return_messages` | 是否返回消息对象而非字符串 | `memory/chat_memory.py` |
| `k`（窗口记忆） | 保留最近轮数 | `memory/buffer_window.py` |
| `max_token_limit`（token 缓冲/摘要缓冲） | token 上限触发裁剪 | `memory/token_buffer.py`、`summary_buffer.py` |

## 5. 错误与重试语义

- **键校验**：`get_prompt_input_key` 在校验输入只含单 key 时定位变量；多键歧义时抛错
  （`memory/buffer.py:151` 附近）。
- **持久化失败**：本层不做重试；后端（Redis/SQL 等）连接失败直接向上抛。
- **摘要失败**：`SummarizerMixin` 调 LLM 失败时，摘要链异常向上传播。
- 异步缺省实现不吞错，`run_in_executor` 透传异常。

## 6. 并发细节

- 记忆层无自建线程；同步 `load_memory_variables`/`save_context` 在 Chain 执行循环内单线程调用。
- 异步默认 `run_in_executor` 包装，不引入新并发模型。
- **只读共享**：`ReadOnlySharedMemory`（`memory/readonly.py`）让多链共享同一记忆实例但
  只允许读取，避免多写竞争——这是唯一的并发/共享语义点。
- 持久化后端（Redis/SQL）自身的并发安全由后端实现保证。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `base_memory.py`、`memory/` 的记忆变体与消息历史适配层。

**Out-of-Scope（不在本仓库源码内）**
- `BaseChatMessageHistory` 基类在 `langchain-core`。
- Redis/SQL/MongoDB/Postgres 等**外部存储系统**本身不在本仓库源码内（仅客户端封装）。
- Zep、Upstash、Momento 等**外部托管服务**不在本仓库源码内。
- 摘要所用 LLM 调用走模型层（见 `classic-llms-chat/`）。

## 8. 与相邻子系统交互

- 上游：`Chain`（`classic-chains/`）与 `AgentExecutor`（`classic-agents/`）注入并读写记忆。
- 本叶子 → 下游：
  - `BaseChatMessageHistory` 持久化后端 → 外部存储。
  - `ConversationSummaryMemory` → LLM Runnable 做摘要。
  - `VectorStoreRetrieverMemory` → 检索器/向量库（`classic-retrievers-stores/`）。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseMemory` 与 `BaseChatMessageHistory` 是两级能力缝——前者定义
  "变量注入/落盘"契约，后者定义"消息增删/持久化"契约；`BaseChatMemory` 把两者粘合。
- **组合模式**：`CombinedMemory` 把子记忆组合成一个；`BaseChatMemory` 组合
  `BaseChatMessageHistory`，是典型对象组合能力缝。
- **混入（Mixin）**：`SummarizerMixin` 把摘要行为混入具体记忆类，而非继承体系分支。
- **目录即实例库**：`chat_message_histories/` 21 个文件是"存储后端目录即实例库"，深读
  `in_memory.py` + `redis.py` 代表即可推断，其余为列表。
- **可选依赖与懒加载**：Redis/SQL/Mongo 等后端在对应文件内延迟 import 第三方客户端。
- **与 core/v1 的关系**：`BaseChatMessageHistory` 已在 core；本包记忆类为遗留实现，
  新代码建议用 LCEL 的 `RunnableWithMessageHistory`。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 记忆体系与持久化后端架构图 | `classic-memory-architecture.html` | architecture | showcase |
| 一次对话轮次的记忆读写时序图 | `classic-memory-conversation-sequence.html` | sequence | standard |

档位披露：架构图组件分层清晰（基类 → 变体 → 历史后端 → 外部存储），保持 `showcase`。
新增时序图刻画一轮对话中 `Chain ↔ BaseMemory ↔ BaseChatMessageHistory` 的注入/生成/落盘
三方交互；因参与者激活条与返回消息较多，showcase 间距校验未全过、回退 `standard`。
JSON IR 位于 `json/`。
