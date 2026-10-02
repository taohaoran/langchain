# model-profiles 域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`libs/model-profiles/langchain_model_profiles/`，branch `master`，commit `89252a8f`。

## 1. 域职责

model-profiles 是一个 CLI 工具（`langchain-profiles`），从 models.dev 拉取模型能力数据，合并本地
`profile_augmentations.toml` 增强，生成各 partner 包使用的 `_profiles.py`（能力标志、上下文窗口等），
并能对比 git ref 生成 PR 变更摘要。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流 | 职责一句话 |
|------|------|--------|--------|-----------|
| model-profiles | [model-profiles.md](model-profiles/model-profiles.md) | [架构图](model-profiles/model-profiles-architecture.html) | [生成数据流](model-profiles/model-profiles-dataflow.html) | 模型配置文件生成 CLI |

## 3. 域级机制细节

- **refresh 流程**：下载 models.dev/api.json → 按 provider 提取 → 加载本地 TOML → `_model_data_to_profile` 转换 → `_apply_overrides` 合并 → 写出 `_profiles.py`（自动生成勿手改）。
- **summarize 流程**：经 `git show` 取 base-ref 旧版 profile，与工作树 `diff_profiles`，渲染 Markdown 供 PR body。
- **代码生成边界**：产物 `_profiles.py` 是生成文件，禁止手改；refresh 命令即生成命令。

## 4. 域级图（合计 3 张 = 域总览架构图 1 张 + 代表性叶子图 2 张）

![model-profiles 生成器位置](model-profiles-domain-architecture.html)

本域为单叶域，域级图配额计入以下 3 张：

| 图 | 文件 | 类型 | 层次 |
|----|------|------|------|
| 生成器位置 | `model-profiles-domain-architecture.html` | architecture | 域级 |
| profile 生成 CLI 架构图 | [model-profiles/model-profiles-architecture.html](model-profiles/model-profiles-architecture.html) | architecture | 叶子级 |
| refresh 生成数据流 | [model-profiles/model-profiles-dataflow.html](model-profiles/model-profiles-dataflow.html) | dataflow | 叶子级 |

域级架构图表达该 CLI 在维护者、models.dev 数据源、本地 TOML 增强与 partner 包之间的生成链路位置；叶子级两张分别表达 CLI 静态组件与 refresh 数据管道，三者语义不重复。
