# 模型配置文件生成（model-profiles）

> 本文是 `model-profiles` 域下的叶子子系统文档。本独立包是一个 CLI，从 models.dev 拉取模型能力数据，
> 合并本地 TOML 增强，生成供 partner 包使用的 `_profiles.py`。
>
> 源码基准：`libs/model-profiles/langchain_model_profiles/`，branch `master`，commit `89252a8f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `langchain-profiles` CLI | argparse 命令行入口，子命令 `refresh` / `summarize` | `cli.py:378`；pyproject entry `cli:main` |
| `refresh` 子命令 | 下载并合并某 provider 的模型 profile | `cli.py:261` |
| models.dev 拉取 | 从 `https://models.dev/api.json` 下载全量模型数据 | `cli.py:272` |
| 本地增强合并 | 加载 `profile_augmentations.toml` 的 provider/model 覆盖 | `cli.py:63,325` |
| profile 转换 | `_model_data_to_profile` 把 models.dev 数据转为内部 profile 结构 | `cli.py:107` |
| 增强-only 模型 | 纯 TOML 定义、models.dev 没有的模型也纳入 | `cli.py:336` |
| 生成 `_profiles.py` | 写出带"自动生成勿手改"横幅的 Python 模块 | `cli.py:357` |
| `summarize` 子命令 | 对比工作树与 git ref，输出 PR 用 Markdown 变更摘要 | `cli.py:403`；`_summary.py:416` |
| profile diff | `extract_profiles / diff_profiles` 计算新增/变更/删除 | `_summary.py:122,171` |
| git 读取 | `_git_show / _verify_ref` 从 git 历史取旧版 profile | `_summary.py:398,360` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `main()` | `cli.py:378` | CLI 入口，分派子命令 |
| `refresh(provider, data_dir)` | `cli.py:261` | 刷新主流程 |
| `summarize(providers, base_ref, repo_root)` | `_summary.py:416` | 变更摘要主流程 |
| `ProfileDiff / FieldChange / ProviderEntry` | `_summary.py:101,74,90` | diff 数据结构 |
| `_ModelProfileRegistry / ModelProfile` | 来自 langchain-core | profile 数据结构 |

## 3. 关键调用链

**调用链一：`langchain-profiles refresh --provider openai --data-dir ...`**

1. `main()` 解析子命令 `refresh`（`cli.py:429`）。
2. `_validate_data_dir` 校验目录（`cli.py:270`）。
3. `httpx.get("https://models.dev/api.json")` 下载全量数据，超时/HTTP/连接错误分别 `sys.exit(1)`（`cli.py:281`）。
4. 按 provider 提取 `models`，`_load_augmentations` 读本地 TOML（`cli.py:325`）。
5. 逐模型 `_model_data_to_profile` 转换后 `_apply_overrides` 应用 TOML 覆盖；增强-only 模型单独加入（`cli.py:329,336`）。
6. `_warn_undeclared_profile_keys` 警告未声明字段（`cli.py:342`）。
7. 把 dict 序列化为 Python（true→True、false→False、null→None），写 `_profiles.py`（`cli.py:357`）。

**调用链二：`langchain-profiles summarize ...`**

1. 解析 `--providers` JSON 数组（`cli.py:435`）。
2. `summarize` 用 `_git_show` 取 base-ref 的旧 profile，与工作树新 profile `diff_profiles`（`_summary.py:171`）。
3. `build_summary` 渲染 Markdown（`_summary.py:314`）打印，供 PR body 使用。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `--provider` | 必填，models.dev 的 provider ID | `cli.py:390` |
| `--data-dir` | 必填，含 `profile_augmentations.toml` 的目录 | `cli.py:395` |
| `--base-ref` | summarize 对比的 git ref，默认 `HEAD` | `cli.py:417` |
| models.dev URL | 硬编码 `https://models.dev/api.json` | `cli.py:272` |
| 输出文件 | `<data-dir>/_profiles.py`（自动生成） | `cli.py:357` |

## 5. 错误与重试语义

- **网络错误**：超时/HTTP 错误/连接失败分别打印 ❌ 并 `sys.exit(1)`；无重试（`cli.py:283`）。
- **provider 缺失**：models.dev 数据里没有该 provider → 报错退出（`cli.py:314`）。
- **JSON 解析失败**：API 返回非 JSON → 退出（`cli.py:298`）。
- **目录权限**：无法创建目录 → 退出（`cli.py:346`）。
- **未声明字段**：仅警告，不中断（`cli.py:342`）。

## 6. 并发细节

- 纯串行 CLI，无并发；同步 `httpx.get`。
- 文件写入一次性完成，`_ensure_safe_output_path` 防路径越界。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- CLI 解析、models.dev 拉取、TOML 合并、`_profiles.py` 生成、diff 摘要。

**Out-of-Scope（不在本仓库源码内）**
- models.dev 模型能力数据源（外部 API，不在本仓库源码内）。
- `ModelProfile / ModelProfileRegistry` 数据结构本体（langchain-core）。
- 生成的 `_profiles.py` 是产物，不在本包源码内（在各 partner `data/` 目录）。

## 8. 与相邻子系统交互

- 上游维护者 → 本包：在 partner `data/` 目录跑 `langchain-profiles refresh`。
- 本包 → 下游 partner：生成的 `_profiles.py` 被 partner 包 import 作为模型能力配置。
- 本包 → 外部 models.dev：HTTP 拉取数据源。
- 本包 → git：summarize 经 `git show` 读历史版本。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **配置驱动代码生成**：本包是"TOML 增强 + 远端数据 → 生成 Python 模块"的代码生成器，产物带"勿手改"横幅；refresh 命令即生成命令。
- **CLI 工具**：argparse 子命令模式，entry point 注册为 `langchain-profiles`。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| profile 生成 CLI 架构图 | `model-profiles-architecture.html` | architecture | **showcase** |
| refresh 生成数据流 | `model-profiles-dataflow.html` | dataflow | standard |

**第三轮刷新**：补全 `meta.output` 字段后重渲染，架构图由第二轮的 standard 提升为 showcase；数据流图保持 standard（涉及外部 API 与生成产物管道，跨层连线较多）。JSON IR 源文件位于 `json/`。
