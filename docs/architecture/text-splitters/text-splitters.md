# text-splitters 域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，独立包 `libs/text-splitters/langchain_text_splitters/`（13 个 py / 约 3686 行）。

## 1. 域职责

text-splitters 是独立发布的文档分块工具集，把长文本按语义与 token 限制切分为适合 LLM 上下文的块，
是 RAG 流程的标准前置。抽象基类 `TextSplitter` 定义契约，派生字符/递归/token/各标记语言实现。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流 | 职责一句话 |
|------|------|--------|--------|-----------|
| text-splitters | [text-splitters.md](text-splitters/text-splitters.md) | [架构图](text-splitters/text-splitters-architecture.html) | [分块数据流](text-splitters/text-splitters-dataflow.html) | 文档分块工具集 |

## 3. 域级机制细节

- **分块机制**：`RecursiveCharacterTextSplitter` 按 `["\n\n","\n"," ",""]` 递归选最佳分隔符；超块向更细粒度递归；`_merge_splits` 把小片合并到 chunk_size 内并维护 chunk_overlap 重叠。
- **配置驱动**：chunk_size/chunk_overlap/length_function/分隔符表决定行为；`length_function` 可换成 tokenizer 计长。
- **可选依赖懒加载**：spaCy/NLTK/KoNLPy 等重依赖不强制导入，`__getattr__` 惰性加载。
