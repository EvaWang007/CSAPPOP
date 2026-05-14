# RAG
## 工作流程
我读了一圈这个项目，先说一个关键判断：它不是 LangChain 那种“带工具调用的 Agent 框架”，而是一个 **RAG from scratch 教学型项目**。它把 RAG 拆成了可理解的模块：`loaders -> splitters -> embeddings -> vector store -> retriever -> prompt/LLM -> answer`。所以这里的“agent 工作流程”更准确说是 **RAG Agent/问答代理流程**：先检索，再把检索到的上下文交给 LLM 生成答案。

**项目结构**
核心库在 `/home/evawang/Downloads/rag-from-scratch1-main/src`：

1. `loaders/`：把外部资料载入成统一的 `Document` 对象。  
代表文件：[PDFLoader.js](/home/evawang/Downloads/rag-from-scratch1-main/src/loaders/PDFLoader.js)、[BaseLoader.js](/home/evawang/Downloads/rag-from-scratch1-main/src/loaders/BaseLoader.js)  
注意：[TextLoader.js](/home/evawang/Downloads/rag-from-scratch1-main/src/loaders/TextLoader.js) 目前还是空类，真正完整实现的是 `PDFLoader`。

2. `utils/Document.js`：统一文档结构。  
每个文档都有 `pageContent` 和 `metadata`。见 [Document.js](/home/evawang/Downloads/rag-from-scratch1-main/src/utils/Document.js)。

3. `text-splitters/`：把长文档切成 chunk。  
核心是 [BaseTextSplitter.js](/home/evawang/Downloads/rag-from-scratch1-main/src/text-splitters/BaseTextSplitter.js)、[CharacterTextSplitter.js](/home/evawang/Downloads/rag-from-scratch1-main/src/text-splitters/CharacterTextSplitter.js)、[RecursiveCharacterTextSplitter.js](/home/evawang/Downloads/rag-from-scratch1-main/src/text-splitters/RecursiveCharacterTextSplitter.js)。

4. `embeddings/`：把文本 chunk 转成向量。  
[EmbeddingModel.js](/home/evawang/Downloads/rag-from-scratch1-main/src/embeddings/EmbeddingModel.js) 用 `node-llama-cpp` 加载本地 GGUF embedding 模型，并支持 cache。

5. `vector-stores/`、`retrievers/`、`chains/`：目录存在，但当前 `src` 里很多是占位空类，例如 `InMemoryVectorStore`、`VectorStoreRetriever`、`RAGChain` 目前还没真正实现。完整 RAG 流程主要写在 examples 里。

6. `llms/`：LLM 抽象层。  
[BaseLLM.js](/home/evawang/Downloads/rag-from-scratch1-main/src/llms/BaseLLM.js) 定义统一接口：`invoke()`、`stream()`、`batch()`、`getModelType()`。  
[LlamaCpp.js](/home/evawang/Downloads/rag-from-scratch1-main/src/llms/LlamaCpp.js) 是具体实现，用本地 llama.cpp 模型生成文本。

**一个具体例子：PDF 文本被 loader 载入后发生什么**
以 showcase 里的 Einstein PDF 问答流程为例，入口在 [showcase.js](/home/evawang/Downloads/rag-from-scratch1-main/examples/06_retrieval_strategies/01_basic_retrieval/showcase.js)。

用户准备一个 PDF，比如 `./docs/einstein.pdf`。

流程如下：

```text
PDF 文件
  -> PDFLoader.load()
  -> Document[]
  -> CharacterTextSplitter.splitText()
  -> chunk[]
  -> embeddingContext.getEmbeddingFor(chunk)
  -> VectorDB.insert(...)
  -> 用户 query
  -> query embedding
  -> VectorDB.search(...)
  -> retrieved chunks
  -> 拼成 context
  -> LLM prompt
  -> 最终回答
```

更具体一点：

1. `PDFLoader` 读取 PDF  
在 [PDFLoader.js](/home/evawang/Downloads/rag-from-scratch1-main/src/loaders/PDFLoader.js) 中，`load()` 用 `pdf-parse` 解析 PDF。如果 `splitPages: true`，每一页会变成一个 `Document`。

每个 `Document` 大概长这样：

```js
{
  pageContent: "这一页提取出来的文本...",
  metadata: {
    source: "./docs/einstein.pdf",
    pdf: {
      totalPages: 22,
      info: ...
    },
    page: 1,
    loc: { pageNumber: 1 }
  }
}
```

2. 文本切块  
在 [showcase.js](/home/evawang/Downloads/rag-from-scratch1-main/examples/06_retrieval_strategies/01_basic_retrieval/showcase.js) 里，`splitDocuments()` 用 `CharacterTextSplitter`，配置是：

```js
chunkSize: 500
chunkOverlap: 40
separator: ' '
```

也就是说它按空格切，再合并成不超过约 500 字符的 chunk，并保留 40 字符左右的重叠，避免答案信息刚好被切断。

3. 生成 embedding  
每个 chunk 会调用：

```js
embeddingContext.getEmbeddingFor(document.pageContent)
```

文本会被转成一个 384 维向量，然后保存到 `embedded-vector-db` 里。

4. 存入向量库  
每个 chunk 存储时包含三部分：

```js
id
embedding vector
metadata: {
  content: chunk文本,
  原始页码等信息
}
```

这样后面检索出来的不只是向量分数，还能拿回原文片段。

**Agent 工作流程**
这个项目里的“agent”可以理解成一个 RAG 问答代理，它不是自己乱猜，而是按固定决策链路工作：

1. 用户提问  
例如：

```text
What was Einstein's school performance like?
```

2. 对 query 做 embedding  
系统把问题也转成同一维度的向量。

3. 向量检索  
用 `VectorDB.search(namespace, queryVector, topK)` 找最相似的 chunk。

4. 构造上下文  
把检索到的 chunk 拼成：

```text
Context:
chunk1...

chunk2...

chunk3...

Question:
What was Einstein's school performance like?

Answer:
```

5. 调用 LLM  
LLM 不是凭空回答，而是在 prompt 里拿到检索上下文后生成回答。

6. 输出答案  
最终回答来自“检索到的证据 + LLM 语言组织”。

**这个项目的核心亮点**
它把 RAG 每一步都拆开了，所以非常适合面试讲“我理解 RAG 的底层流程”：

1. Loader 解决“外部资料如何进入系统”。  
2. Document 解决“资料统一表示”。  
3. Splitter 解决“长文本无法直接进模型/向量效果差”。  
4. Embedding 解决“文本语义如何变成可搜索的数字”。  
5. Vector store 解决“如何快速找相似内容”。  
6. LLM 解决“如何基于检索内容生成自然语言回答”。  



***In Conclusion🥑:***
> 文本被 loader 载入成 Document，Document 包含正文 pageContent 和描述来源的 metadata。
> 随后 splitter 把长文档切成带 metadata 的 chunk，再对 chunk 文本做 embedding 并存入向量库。
> 用户提问时，问题也被转成向量，系统用向量相似度检索相关 chunk，再把这些 chunk 的正文（必要时附带 metadata，如页码和来源）拼进 prompt，交给 LLM 生成答案。
>metadata 主要负责保留来源、页码、chunk 编号等信息，方便检索过滤和答案溯源。
> 🔥metadata 让系统在检索出 chunk 后，知道这个 chunk 来自原文本的哪个部分；
>如果把它写入 prompt，LLM 也能基于这些来源信息做引用



















