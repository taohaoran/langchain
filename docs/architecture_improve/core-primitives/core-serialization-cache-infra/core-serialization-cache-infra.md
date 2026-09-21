# 序列化、缓存与基础设施（core-serialization-cache-infra）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「LC 序列化、LLM 缓存、键值存储、限流、全局配置、异常体系、懒加载与安全策略」等横切基础设施，不重复展开各业务叶子。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/load/` + `caches.py` + `rate_limiters.py` + `stores.py` + `exceptions.py` + `globals.py` + `env.py` + `_api/` + `_import_utils.py` + `utils/` + `_security/` 等。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `Serializable` | 所有 langchain 对象的序列化基类（pydantic） | `load/serializable.py:106` |
| 序列化构造 | `to_json`/`lc_id`/`lc_secrets`/`lc_attributes`：导出为 `SerializedConstructor` | `serializable.py:227`/`196`/`178`/`186` |
| 安全反序列化 | `load`：带类路径白名单的 `load`，防模板注入与任意实例化 | `load/load.py:289`、`_block_jinja2_templates` |
| dump | `dumpd`/`dump`：把对象序列化为 dict/JSON | `load/dump.py` |
| 类路径映射 | `mapping.py`：旧类路径→新路径 | `load/mapping.py` |
| `BaseCache` | LLM 响应缓存 ABC：`lookup`/`update`/`clear` | `caches.py:32` |
| 内存缓存 | `InMemoryCache`（可选 LRU maxsize） | `caches.py:155` |
| `BaseStore` | 通用键值存储 ABC：`mget`/`mset`/`mdelete`/`yield_keys` | `stores.py:26` |
| 内存存储 | `InMemoryStore`/`InMemoryByteStore` | `stores.py:244`/`267` |
| `BaseRateLimiter` | 限流 ABC：`acquire(blocking)` | `rate_limiters.py:11` |
| 内存限流 | `InMemoryRateLimiter`（令牌桶） | `rate_limiters.py:67` |
| 全局配置 | `set_verbose`/`set_debug`/`set_llm_cache`/`get_*` | `globals.py:18-66` |
| 运行环境 | `get_runtime_environment()` | `env.py:10` |
| 异常体系 | `LangChainException` 根 + `OutputParserException`/`Model*Error`/`ContextOverflowError` 等 | `exceptions.py:7` |
| 错误码 | `ErrorCode` 枚举 | `exceptions.py:131` |
| 懒加载导入 | `import_attr`：模块级 `__getattr__` 惰性导入符号 | `_import_utils.py:4` |
| 版本 API 装饰器 | `@deprecated`/`@beta`/`rename_parameter` | `_api/deprecation.py:120`、`beta_decorator.py:32` |
| SSRF 防护 | `validate_safe_url`/`SSRFPolicy`/`validate_hostname` | `_security/_ssrf_protection.py:41`、`_policy.py:103` |
| 通用工具 | `utils/`：json/pydantic/iter/mustache/json_schema 等 | `utils/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Serializable` | `load/serializable.py:106` | 序列化契约；Runnable/Message/Prompt 均继承它 |
| `BaseCache` | `caches.py:32` | LLM 缓存契约；按 (prompt, llm_string) 键存响应 |
| `BaseStore[K,V]` | `stores.py:26` | 通用 KV 契约，批量 mget/mset/mdelete |
| `BaseRateLimiter` | `rate_limiters.py:11` | 限流契约 |
| `import_attr` | `_import_utils.py:4` | 懒加载工具：按 `_dynamic_imports` 映射延迟 import |
| `LangChainException` | `exceptions.py:7` | 异常根类 |
| `SSRFPolicy` | `_security/_policy.py:103` | URL 安全策略（allow_private/allow_http） |
| `validate_safe_url` | `_security/_ssrf_protection.py:41` | 校验 URL 是否可安全访问 |

## 3. 关键调用链

**调用链一：对象序列化（`Serializable.to_json`，`serializable.py:227`）**

1. `lc_id` 返回模块路径标识，`lc_secrets` 标记需脱敏字段。
2. `to_json` 产出 `SerializedConstructor`（含 `id`/`kwargs`），secrets 替换成 `SerializedSecret`。
3. `dump`/`dumpd` 递归遍历对象图产出 JSON 可序列化 dict。

**调用链二：安全反序列化（`load`，`load.py:289`）**

1. 从 `SerializedConstructor` 的 `id` 解析类路径。
2. `_compute_allowed_class_paths` 校验类路径在白名单内；`_block_jinja2_templates` 阻止模板注入。
3. `default_init_validator` 校验 kwargs 后实例化；不在白名单则拒绝，防任意代码执行。

**调用链三：LLM 缓存命中（`BaseCache`）**

1. 模型调用前 `lookup(prompt, llm_string)`，命中则直接返回缓存响应，跳过 API 调用。
2. 未命中则调用模型，`update(prompt, llm_string, response)` 写入缓存。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `set_debug`/`set_verbose` | 全局调试/详细日志开关 | `globals.py:37`/`18` |
| `set_llm_cache` | 全局 LLM 缓存实例 | `globals.py:56` |
| `InMemoryCache(maxsize)` | LRU 上限，None 不限 | `caches.py:182` |
| `InMemoryRateLimiter` 速率 | 每秒请求数/时间窗口 | `rate_limiters.py:120` |
| SSRF `allow_private`/`allow_http` | 是否允许内网/HTTP URL | `_security/_policy.py:103` |
| 反序列化类路径白名单 | `allowed_imports` 参数 | `load/load.py` |

## 5. 错误与重试语义

- **异常层级**：`LangChainException` 根；`OutputParserException` 继承 ValueError；`Model*Error` 族按 401/403/404/429/5xx 细分（`exceptions.py:68-123`），`ContextOverflowError` 表示上下文超长。
- **反序列化安全**：不在白名单的类路径直接拒绝；jinja2 模板注入被 `_block_jinja2_templates` 拦截。
- **缓存 miss**：未命中是正常路径，不报错；`lookup` 返回 None。
- **限流**：`acquire(blocking=False)` 时超限返回 False，不抛错。

## 6. 并发细节

- **全局状态**：`globals.py` 的 debug/verbose/cache 是模块级全局，经普通变量读取；非线程安全的写期不并发。
- **内存缓存**：`InMemoryCache` 用 dict，并发读写需外部同步。
- **限流**：`InMemoryRateLimiter` 令牌桶，`acquire` 可阻塞。
- **无 async 原语**：本叶子同步为主。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `load/` 序列化、`caches.py`、`stores.py`、`rate_limiters.py`、异常/全局/环境、`_api/` 版本装饰器、`_security/` SSRF、`_import_utils.py`、`utils/`。

**Out-of-Scope（不在本仓库源码内）**
- 外部缓存后端（Redis/Redis 缓存）在 partners；本叶子只定义 `BaseCache` 契约。
- 持久化 KV 后端（Redis/DB）在 partners。
- 反序列化白名单外的第三方库由用户显式允许。

## 8. 与相邻子系统交互

- **上游（全栈）**：`Serializable` 是 Runnable/Message/Prompt/Model 的共同基类，序列化贯穿所有叶子。
- **本叶子 → 模型（`core-language-models`）**：`BaseCache` 在模型层拦截重复请求。
- **本叶子 → 加载器（`core-documents-loaders`）**：`BaseStore` 供增量索引用。
- **本叶子 → 全栈**：`_import_utils.import_attr` 是各包 `__getattr__` 懒加载的底层；`exceptions.py` 全栈共用；`_api` 装饰器标记弃用/beta API。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`Serializable`/`BaseCache`/`BaseStore`/`BaseRateLimiter` 是抽象基类契约；`import_attr` 是懒加载机制。
- **可选依赖与懒加载**：`_import_utils.import_attr` 被各包 `__getattr__` 使用，实现符号惰性导入，避免顶层导入即加载全部重实现。
- **配置驱动**：`globals.py` 全局开关（debug/cache）驱动运行时行为。
- **安全**：`_security/` 的 SSRF 防护与 `load/` 的反序列化白名单是 LangChain 安全模型核心，防 RCE。
- **图类型侧重**：architecture 表达序列化往返与基础设施分层组件拓扑，dataflow 表达「对象→to_json→SerializedConstructor→load→白名单校验→新实例」的序列化往返数据管道；无单实体状态机，不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 序列化往返与基础设施架构图 | `core-serialization-cache-infra-architecture.html` | architecture | **showcase**（沿用基线） |
| 对象序列化往返数据流图 | `core-serialization-cache-infra-dataflow.html` | dataflow | standard（本轮新增；表达对象→to_json→SerializedConstructor→load→白名单校验→新实例数据管道，符合 dataflow 核心语义；缩短长标签、两行布局后通过） |

JSON IR 源文件位于 `json/` 目录。本轮相对基线的改进：基线仅有 1 张架构图，本轮新增 1 张 dataflow 图表达序列化往返数据管道。两图均实际渲染成功（退出码 0、HTML 非空）。
