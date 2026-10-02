# 相对基线的改进对比说明（第三轮）

> 基线（第一轮）：`../architecture/`（只读未改动，41 MD + 46 HTML + 46 JSON，7 域 32 叶）。
> 本轮（第三轮改进版）：本目录 `docs/architecture_improve/`，按改造后 skill 的「五种图核心语义选型 + 分级配额」出图，并相对第一轮基线逐项对比。
> 源码基准 commit：`89252a8f`（2026-09-21；第二轮为 `4492ad7a`，两者间仅 1 个 docs 部署 commit，无源码改动）。

## 一、配额与图数变化

| 层级 | 基线（第一轮） | 本轮（第三轮） | 变化 |
|------|------|------|------|
| 系统级 | 3 图（architecture/sequence/dataflow） | **4 图**（+ lifecycle） | +1 |
| 域级 | 多数域仅 0–1 张域图 | **每域 ≥3 张**（7 域共 15 张域级图） | 系统性补齐 |
| 叶子级 | 11 叶 2 图、21 叶仅 1 图（32 叶） | **35 叶全部 ≥2 图**（partners 深化 +3 叶） | +3 叶、21 个单图叶补第 2 张 |
| HTML 总数 | 46 | **89** | +43 |
| JSON IR 总数 | 46 | **89** | +43 |
| MD 总数 | 41 | **45** | +4（README/总览/对比说明 + 新叶 MD 净增） |

## 二、按域的改进明细

### core-primitives（10 叶，保持）
- 4 张图由 standard 升 showcase：documents-architecture、documents-dataflow、prompts-dataflow、serialization-dataflow（新版 archify CLI 布局引擎下重渲染自然达标）。
- 3 张扇形 dataflow（callbacks / messages / output-parsers）与 2 张域级图保持 standard：一对三扇出/扇入出边标签间距 0–3.3px，经 labelDy/路由/居中布局多轮修复仍差 showcase 阈值，第 10 节披露失败检查名与修复动作。
- 全部叶子关键调用链行号按当前 HEAD 实读核验（Runnable ABC、`_call_with_config`、VectorStore、Document、BaseTool 等 6 处抽查精确命中）。

### classic（8 叶，保持）
- 第二轮两个已知薄弱图 **chains-sequence、loaders-dataflow 由 standard 提升至 showcase**（移除显式 viewBox 让渲染器自动适配画布 + 缩短过长子标签）。
- 另有 5 张图同步提升 showcase：memory-conversation-sequence、retrievers-retrieval-dataflow、utils-eval-retry-sequence、域级 classic-dataflow、域级 classic-sequence。
- 6 张复杂图保持 standard（域级 architecture、chains-architecture、agents-lifecycle、callbacks-dataflow），跨层连接复杂属预期，第 10 节披露。
- 行号核验：`Chain`(chains/base.py:52)、`AgentExecutor`(agents/agent.py:1012)、`ConversationBufferMemory`、`RetryWithErrorOutputParser`、`load_chain` 等与 HEAD 精确一致。

### agents（6 叶）+ text-splitters（1 叶，保持）
- 9 张图由 standard 升 showcase：agent-factory-architecture、agent-middleware 双图、agent-execution-tools-architecture、mcp-integration 双图、chat-models-embeddings-architecture、messages-rate-limiters-architecture、text-splitters 双图与域架构图。
- agent-factory-lifecycle 保持 standard（tools→model 回边触发 `orthogonal-arrows`，显式 via 走廊残留斜向肘段；M4 兜底保留并在第 10 节披露）。**事实更正：第二轮误标该图为 showcase，本轮按实渲染档位如实记录为 standard。**
- agents 域级 3 图全部达 showcase（修复 `desktop-readability`：收紧过宽 viewBox）。

### partners（5 叶 → 8 叶，本轮最大深化）
- **叶子拆分**：openai / anthropic / search-tools 沿用刷新；qdrant、chroma 由 vector-stores 拆为独立叶；llm-api-providers（perplexity/fireworks/groq/mistralai/openrouter/xai/deepseek 七家云端 API 提供商）、local-inference（ollama + huggingface 本地推理）、nomic（嵌入/重排）由 model-providers 拆出。旧叶 `partner-vector-stores/`、`partner-model-providers/` 已删除。
- 17 个包逐包归组：深读核验 openai/anthropic/exa/qdrant/chroma/deepseek/xai/groq/openrouter/ollama 入口类与关键方法行号；mistralai/perplexity/fireworks/huggingface 以共性列表覆盖并如实披露未逐行深读。
- 新增 3 叶 6 图全量渲染成功；partner-qdrant/chroma/nomic 六图全部 showcase；llm-api-providers、local-inference 的架构图保持 standard（10+ 组件跨两层继承，披露）。
- 沿用叶（openai/anthropic/search-tools）架构图均提升 showcase；第二轮已核实事实（`ChatOpenRouter` 继承 `BaseChatModel`、deepseek/xai 继承 `BaseChatOpenAI`）本轮保持。

### testing-infra / model-profiles（单叶域，保持）
- 两个叶子架构图均由 standard 升 showcase；域级架构图保持 standard（单叶域"域级 1 + 叶子 2 = 3 图"口径不变）。

## 三、系统级变化
- 源码基准 commit 由 `4492ad7a` 更新为 `89252a8f`；4 张系统图在新版 archify CLI（schema 要求 `meta.output`）下统一补齐字段并重渲染成功（全部 standard，跨层连接复杂为预期，档位披露）。
- system-architecture 修复「实现契约」连线标签与 partners 组件重叠的布局约束问题（`labelAt` 定位）。
- 省略系统级 workflow 的原因维持：库级框架无多角色审批/发布流，语义已由 sequence + lifecycle 表达。

## 四、质量档位与回归对比
- **回归口径说明**：上游 archify CLI 在第二轮后升级（HEAD `69cf672`），五种图 schema 强制要求 `meta.output` 字段，缺失即校验失败。为保证同一 CLI 口径可比，第二轮基线摘要已先行机械补齐该字段后重采（83 图 / 61.4% showcase 通过率）。
- **回归结果（`regression-check.py compare`）**：通过率 **61.4% → 77.5%（+16.1 个百分点）**；13 项提升（原不过现过），**0 项退步（原过现不过）**；2 张新增叶图落 standard（partner-llm-api-providers-architecture、partner-local-inference-architecture，均为本轮新叶，属"新增图"，已在对应 MD 第 10 节披露失败检查名与修复动作）。
- 叶子级目标 showcase（合格线）；落 standard 者全部在叶子 MD 第 10 节写失败检查名 + 采取的修复动作，无降档不披露的情况。

## 五、未改动
- 基线 `docs/architecture/` 内任何文件未改动；git 改动严格限定于 `docs/architecture_improve/`（211 个文件：修改 + 新增；旧叶目录删除后净计数与 89 HTML / 89 JSON / 45 MD 一致）。
- 系统级 `json/` 下 4 份 IR 仅补 `meta.output` 与一处 labelAt，语义未变。
