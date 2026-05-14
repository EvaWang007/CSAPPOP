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
***1.问答SOP***

> 文本被 loader 载入成 Document，Document 包含正文 pageContent 和描述来源的 metadata。
> 随后 splitter 把长文档切成带 metadata 的 chunk，再对 chunk 文本做 embedding 并存入向量库。
> 用户提问时，问题也被转成向量，系统用向量相似度检索相关 chunk，再把这些 chunk 的正文（必要时附带 metadata，如页码和来源）拼进 prompt，交给 LLM 生成答案。
>metadata 主要负责保留来源、页码、chunk 编号等信息，方便检索过滤和答案溯源。
> 🔥metadata 让系统在检索出 chunk 后，知道这个 chunk 来自原文本的哪个部分；
>如果把它写入 prompt，LLM 也能基于这些来源信息做引用


***2.文本切块和Metadata提取策略

一、文档与 metadata 从哪来

1. **Loader 读入**（如 `PDFLoader.load()`）把 PDF 变成若干 `Document`：每个有 `pageContent`（正文）和 `metadata`（如 `source`、`pdf`、`page`、`loc.pageNumber` 等）。  
2. **或手写** `new Document(正文, { id, topic, … })`，metadata 由你定义。  
3. **`Document` 类**只负责把 `pageContent` 和 `metadata` 两个字段存在一起，不做语义分析。

---

二、切块入口与数据流

4. 对 `Document[]` 调用 **`splitDocuments`**：对每个文档取 `pageContent` 与 `metadata`，交给 **`createDocuments([text], [metadata])`**。  
5. **`createDocuments`** 用子类实现的 **`splitText(text)`** 把字符串切成多段字符串。  
6. 对每一段字符串 **`new Document(块正文, { ...原metadata, chunk: j, totalChunks: N })`**：  
   - **`...原metadata`**：整块继承父文档的 metadata（来源、页码等不变）。  
   - **`chunk` / `totalChunks`**：只表示「**当前父文档**被切成第几块、共几块」（换父文档后 `chunk` 会重新从 0 计数）。

---

三、切块策略（与 metadata 无关的「怎么切」）

7. **`CharacterTextSplitter`**：按**单一**分隔符（默认 `\n\n`）先 `split`，再用 **`mergeSplits`** 按 `chunkSize`、`chunkOverlap`、`lengthFunction` 合并小段，控制单块长度与重叠。  
8. **`RecursiveCharacterTextSplitter`**：按 **`separators` 优先级**（默认 `\n\n` → `\n` → `. ` → ` ` → `''`）先选「能在文中找到的最靠前分隔符」切开；**仍超长**的子串用**更细**的分隔符列表**递归** `splitText`；合适长度的片段再交给 **`mergeSplits`**。  
9. **`TokenTextSplitter`**：用「**字符数/4 上取整**」近似 token 作为 `lengthFunction`，内部仍用 **`RecursiveCharacterTextSplitter`** 做实际切分。

---

四、metadata「策略」总结（不叫「提取」更准确）

10. **生成时机**：主要在 **载入/创建 `Document` 时**写入；**切块器不解析正文去「提取」**新 metadata。  
11. **切块阶段**：**只复制并追加**——复制父级 metadata，**追加** `chunk`、`totalChunks`。  
12. **入库阶段**（示例代码常见写法）：在 metadata 里**再并入** `content`（正文副本）和向量一起存，便于检索后直接取文本；**向量本身仍由 `pageContent` 嵌入得到**。  
13. **给 LLM**：默认只把拼进 prompt 的**字符串**（如正文）给模型；**metadata 对象不自动进模型**；若要让模型知道出处，需在 prompt 里**手写**来源/页码等（由 metadata 提供值）。

---

五、一句话串联

**加载（或创建）时挂上 metadata → 切块时继承父 metadata 并加上块序号 → 嵌入用正文 → 入库常把正文副本与 metadata 和向量绑在一起 → 检索与回溯靠 metadata，进 LLM 的主要是你拼好的正文与指令。**



















