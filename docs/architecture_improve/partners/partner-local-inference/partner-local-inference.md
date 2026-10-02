# partner-local-inference 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开本地进程内推理集成（ollama + huggingface）的职责边界，不重复展开云端 API 提供商、
> OpenAI/Anthropic 主适配层、向量库等相邻叶子。
>
> 源码基准：`libs/partners/ollama/` 与 `libs/partners/huggingface/`；branch `master`，commit `89252a8f`。

## 1. 功能清单

### Ollama 包（`langchain-ollama`）

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Chat 模型 | `ChatOllama` 封装本地 Ollama 守护进程的 Chat API | `ollama/langchain_ollama/chat_models.py:262` |
| 文本补全 LLM | `OllamaLLM` 封装 Ollama 纯文本补全 | `ollama/langchain_ollama/llms.py:26` |
| 嵌入 | `OllamaEmbeddings` 调 Ollama 本地嵌入端点 | `ollama/langchain_ollama/embeddings.py:25` |
| 生成 | `_generate`（:1204）/ `_stream`（:1295）支持同步/异步流式 | chat_models.py |

### HuggingFace 包（`langchain-huggingface`）

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 本地 Pipeline LLM | `HuggingFacePipeline` 包装 transformers 本地 pipeline | `huggingface/langchain_huggingface/llms/huggingface_pipeline.py:39` |
| 远端 Endpoint LLM | `HuggingFaceEndpoint` 调 HF Inference API 远端端点 | `llms/huggingface_endpoint.py:45` |
| Chat 模型 | `ChatHuggingFace` 包装 TGI（Text Generation Inference）端点 | `chat_models/huggingface.py:325` |
| 本地嵌入 | `HuggingFaceEmbeddings` 包装 sentence-transformers | `embeddings/huggingface.py:10` |
| 远端嵌入 | `HuggingFaceEndpointEmbeddings` 调 HF 远端嵌入端点 | `embeddings/huggingface_endpoint.py:15` |
| 可选依赖检测 | `import_utils.py` 检测 transformers/torch 是否安装 | `utils/import_utils.py` |

**与云端 API 提供商的关键区别**：Ollama 连本地守护进程（`localhost:11434`），HF Pipeline 是进程内直接加载模型权重——不调云端 API。

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `ChatOllama(BaseChatModel)` | ollama chat_models.py:262 | 本地 Ollama Chat 适配；`base_url`（:694）默认 `http://localhost:11434` |
| `OllamaLLM(BaseLLM)` | ollama llms.py:26 | 本地文本补全 |
| `OllamaEmbeddings(BaseModel, Embeddings)` | ollama embeddings.py:25 | 本地嵌入 |
| `HuggingFacePipeline(BaseLLM)` | hf pipeline.py:39 | 进程内 transformers pipeline 包装 |
| `HuggingFaceEndpoint(LLM)` | hf endpoint.py:45 | 远端 HF Inference API |
| `ChatHuggingFace(BaseChatModel)` | hf chat_models/huggingface.py:325 | TGI Chat 端点包装 |
| `HuggingFaceEmbeddings(BaseModel, Embeddings)` | hf embeddings/huggingface.py:10 | sentence-transformers 本地嵌入 |

## 3. 关键调用链

### 链 1：Ollama Chat 请求

1. `ChatOllama._generate`（ollama chat_models.py:1204）构建请求 payload。
2. `:953` 解析 `base_url` 与认证头，调 `self.client.chat(...)`（ollama Python SDK）。
3. SDK 发 HTTP 到本地 Ollama 守护进程（默认 `localhost:11434`）。
4. Ollama 守护进程加载本地模型权重（GGUF 格式）做推理。
5. 响应转换为 langchain `ChatResult`。`_stream`（:1295）支持流式 token。

### 链 2：HF Pipeline 本地推理

1. `HuggingFacePipeline`（pipeline.py:39）在初始化时接收一个 transformers `pipeline` 对象。
2. `_call` 调 `pipeline(prompt, ...)` 进程内推理（加载模型权重到 GPU/CPU）。
3. 返回生成文本。

### 链 3：HF Endpoint 远端推理

1. `HuggingFaceEndpoint`（endpoint.py:45）配置 `endpoint_url` 与 `HUGGINGFACEHUB_API_TOKEN`。
2. `_call` 发 HTTP POST 到 HF Inference API 远端端点。
3. 与云端 API 提供商模式类似，但端点是 HuggingFace 托管的推理服务。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `base_url`（Ollama） | `http://localhost:11434` | ollama chat_models.py:694 |
| `model`（Ollama） | 本地模型名（如 `llama3`） | chat_models.py |
| `HUGGINGFACEHUB_API_TOKEN` | HF 远端 API token 环境变量 | hf endpoint |
| `endpoint_url`（HF Endpoint） | HF 推理端点 URL | hf endpoint.py:45 |
| `pipeline`（HF Pipeline） | 用户注入的 transformers pipeline 实例 | hf pipeline.py:39 |

## 5. 错误与重试语义

- **Ollama 连接失败**：守护进程未启动时 HTTP 连接失败，异常透传给调用方。
- **HF Pipeline 依赖缺失**：`import_utils.py` 检测 transformers/torch 未安装时抛带安装指引的 ImportError。
- **重试**：本层不实现重试；Ollama 由 SDK 负责，HF Endpoint 由 `requests`/httpx 负责。

## 6. 并发细节

- **Ollama**：同步/异步双实现；`ChatOllama` 通过 ollama SDK 连本地守护进程。
- **HF Pipeline**：进程内推理，GIL 限制下同步执行；GPU 推理可并行化但本层不管理。
- **无共享可变状态**：pydantic 实例字段即配置。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_ollama/`：ChatOllama/OllamaLLM/OllamaEmbeddings 适配
- `langchain_huggingface/`：Pipeline/Endpoint/Chat/Embeddings 适配
- `langchain_huggingface/utils/import_utils.py`：可选依赖检测

**Out-of-Scope（不在本仓库源码内）**

- Ollama 本地守护进程（`ollama` 二进制）——外部运行时
- 本地模型权重文件（GGUF/safetensors）——外部文件
- `transformers`/`torch`/`sentence-transformers` 库——依赖项
- `ollama` Python SDK——依赖项
- HuggingFace Inference API 远端服务——外部服务
- 相邻叶子：云端 API 提供商、OpenAI/Anthropic 主适配层

## 8. 与相邻子系统交互

- **上游 → 本叶子**：用户应用通过 `BaseChatModel`/`LLM`/`Embeddings` 抽象调用。
- **本叶子 → 下游**：
  - `ChatOllama` → ollama SDK → Ollama 本地守护进程（外部）；
  - `HuggingFacePipeline` → transformers 库（进程内推理，外部依赖）；
  - `HuggingFaceEndpoint` → HF Inference API（外部服务）。
- **同域交互**：依赖 `langchain_core` 抽象；与其他 partner 包无直接代码依赖。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：核心能力缝是 `BaseChatModel`/`LLM`/`Embeddings` 抽象的本地实现——与云端提供商并列，但计算在本地进程。
- **可选依赖与懒加载**：`import_utils.py` 集中检测 transformers/torch 是否安装；`HuggingFacePipeline` 接收外部 pipeline 对象，不在 import 时加载重框架。
- **双轨实现**：HF 包同时存在进程内（Pipeline）与远端（Endpoint）两条轨——同一 LLM 接口下，本地 transformers 推理 vs 远端 API 调用。
- **外部边界**：Ollama 守护进程、transformers/torch、HF Inference API 均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 本地推理架构图 | `partner-local-inference-architecture.html` | architecture | standard |
| Ollama 请求数据流图 | `partner-local-inference-dataflow.html` | dataflow | **showcase** |

**档位说明**：架构图跨"抽象层 → Ollama/HF 两分支 → 守护进程/transformers/远端 API/模型权重"四路末端、连线较多，降为 standard（修复动作：合并同类组件、缩短标签）。数据流图聚焦 Ollama 主路径一次通过 showcase。不补 sequence 图——请求链路是单向管道（适配→守护进程→模型→响应），与 dataflow 信息重复。不补 lifecycle 图——无明确多状态机语义。JSON IR 源文件位于 `json/` 目录。
