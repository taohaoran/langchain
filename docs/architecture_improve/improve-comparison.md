# 相对基线的改进对比说明

> 基线（第一轮）：`../architecture/`（只读未改动，41 MD + 46 HTML + 46 JSON，7 域 32 叶）。
> 本轮（第二轮改进版）：本目录 `docs/architecture_improve/`，按改造后 skill 的「五种图核心语义选型 + 分级配额」重出图与重写文档。
> 源码基准 commit 相同（`4492ad7a`），叶子清单不变（7 域 32 叶）。

## 一、配额与图数变化

| 层级 | 基线 | 本轮 | 变化 |
|------|------|------|------|
| 系统级 | 3 图（architecture/sequence/dataflow） | **4 图**（+ lifecycle：代理运行状态机） | +1 |
| 域级 | 多数域仅 0–1 张域图 | **每域 ≥3 张**（7 域共 16 张域级图） | 系统性补齐 |
| 叶子级 | 11 叶 2 图、21 叶仅 1 图 | **32 叶全部 ≥2 图** | 21 个单图叶各补第 2 张 |
| HTML 总数 | 46 | **83** | +37 |
| JSON IR 总数 | 46 | **83** | +37 |
| MD 总数 | 41 | **42** | +1（本对比说明） |

## 二、按域的改进明细

### core-primitives（10 叶）
- 9 个单图叶各补第 2 张：按核心语义选 dataflow（消息归一化/提示渲染/输出解析/检索管道/文档准备/序列化往返）或 sequence（工具调用链/模型调用链）；core-callbacks-tracers 反向补 architecture。
- 域级从 0 → 3 张（architecture + dataflow + sequence）。
- 多张架构图删冗余边标签后由 standard 升 showcase。
- 省略：域级 lifecycle——无单实体状态机，已写入域总览。

### classic（8 叶）
- 6 个单图叶各补第 2 张：llms-chat 委托时序、memory 对话读写时序、loaders 架构图、retrievers 检索数据流、utils-eval 重试时序、callbacks 事件分发数据流。
- 域级 0 → 3 张（architecture + dataflow + sequence：Chain 内 LLM 调用跨 core→partner→远端 API）。
- classic-agents 与 classic-retrievers-stores 架构图由 standard 升 showcase。
- 披露：chains-sequence、loaders-dataflow 在当前 archify 版本校验趋严下回退 standard（非内容退化）。

### agents（6 叶）+ text-splitters / testing-infra / model-profiles（3 单叶域）
- 3 个单图叶补第 2 张：agent-execution-tools 子代理数据流、chat-models-embeddings init_chat_model 时序、messages-rate-limiters 限流数据流；standard-tests 补接入流程 workflow。
- agents 域级 0 → 3 张（architecture + 构建执行 workflow + 洋葱链时序）。
- 3 个单叶域各补 1 张域级架构图，域总览明示"域级图 = 域架构 1 + 叶子 2 = 3"。
- 新增 10 张图全部一次通过 showcase。

### partners（5 叶）
- 2 个单图叶补第 2 张：model-providers、search-tools 各补时序图（均 showcase）。
- 域级 0 → 3 张（architecture + 请求时序 + 消息变换数据流）。
- 基线 8 张全 standard → 本轮 13 张中 **7 张达 showcase**（anthropic 双图、vector-stores 双图、openai 时序、model-providers/search-tools 时序、域级时序）。
- 事实修正：`ChatOpenRouter` 实际继承 `BaseChatModel`（自实现转换），非基线所写的 `BaseChatOpenAI`，已在 MD 与域总览更正。
- 省略：lifecycle/workflow——partners 为请求-响应适配层，无单实体状态机与带分支审批流程，已在各叶第 10 节说明。

## 三、系统级变化
- 新增 **system-lifecycle.html**：单实体「一次 agent 运行」状态机（已创建→模型推理→工具执行→完成/失败/取消，含 ToolMessage 回传循环与重试回迁边）。
- system-sequence 移除与当前版本校验冲突的 `segments` 阶段带，三图重渲染通过。
- 省略系统级 workflow：库级框架无多角色审批/发布流，相关语义已由 sequence + lifecycle 表达。

## 四、质量档位
- 系统级 4 图 standard（跨层连接复杂，符合预期）。
- 叶子级新增图以 showcase 为主；落 standard 者均在各叶 MD 第 10 节披露失败检查名与修复动作（去 segments、labelDy 错标、压缩 y、拆图等）。

## 五、未改动
- 基线 `docs/architecture/` 内任何文件未改动；git 仅新增 `docs/architecture_improve/`。
- 叶子清单 7 域 32 叶与基线一致，未增删。
