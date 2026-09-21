# model-profiles 域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，独立包 `libs/model-profiles/langchain_model_profiles/`（3 个 py / 约 945 行）。

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
