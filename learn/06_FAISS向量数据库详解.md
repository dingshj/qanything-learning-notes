# 06_FAISS向量数据库详解

## 1. 什么是FAISS？

**FAISS（Facebook AI Similarity Search）** 是 Facebook 开发的向量相似度搜索库。

### 作用

在海量向量中快速找到最相似的 Top-K 个向量。

### 生活类比：图书馆找书

| 场景 | 传统方式 | FAISS方式 |
|------|----------|-----------|
| 图书馆找书 | 一本一本翻 | 按分类快速定位 |
| 搜索问题 | 逐条匹配 | 向量最近邻搜索 |
| 查询速度 | O(n) 逐条扫描 | O(log n) 索引加速 |

---

## 2. 文件结构

```
connector/database/faiss/
└── faiss_client.py      # FAISS客户端
```

---

## 3. 代码解析

### 3.1 核心类 FaissClient

```python
class FaissClient:
    def __init__(self, mysql_client, embeddings):
        self.mysql_client = mysql_client
        self.embeddings = embeddings
        self.faiss_client: FAISS = None
        self.kb_ids: List[str] = []
```

### 3.2 初始化索引

```python
def _load_kb_to_memory(self, kb_ids):
    for kb_id in kb_ids:
        faiss_index_path = os.path.join(FAISS_LOCATION, kb_id, 'faiss_index')
        if os.path.exists(faiss_index_path):
            # 从磁盘加载已有索引
            faiss_client = load_vector_store(faiss_index_path, self.embeddings)
        else:
            # 创建新索引（768维，L2距离）
            index = faiss.IndexFlatL2(768)
            faiss_client = FAISS(self.embeddings, index, docstore, {})
```

**关键点**：
- `IndexFlatL2(768)` - 768维向量，L2距离（欧氏距离）
- 每个知识库一个索引文件

### 3.3 搜索方法

```python
async def search(self, kb_ids, query, filter=None, top_k=40):
    # 1. 加载知识库索引
    self._load_kb_to_memory(kb_ids)

    # 2. 向量相似度搜索
    docs_with_score = await self.faiss_client.asimilarity_search_with_score(
        query, k=top_k, fetch_k=200
    )

    # 3. 添加得分到元数据
    for doc, score in docs_with_score:
        doc.metadata['score'] = score

    # 4. 合并相邻块
    docs = self.merge_docs(docs)
    return docs
```

### 3.4 文档合并

相邻的 chunk 如果来自同一个文件，可以合并成一个更长的上下文：

```python
def merge_docs(self, docs):
    merged_docs = []
    docs = sorted(docs, key=lambda x: (x.metadata['file_id'], x.metadata['chunk_id']))

    for doc in docs:
        if merged_docs[-1].metadata['file_id'] == doc.metadata['file_id']:
            if merged_docs[-1].metadata['chunk_id'] == doc.metadata['chunk_id'] - 1:
                # 相邻chunk，合并
                merged_docs[-1].page_content += '\n' + doc.page_content
```

### 3.5 添加文档

```python
async def add_document(self, docs):
    kb_id = docs[0].metadata['kb_id']

    # 1. 添加到向量索引
    add_ids = await self.faiss_client.aadd_documents(docs)

    # 2. 保存到MySQL（记录chunk_id等元信息）
    for doc, add_id in zip(docs, add_ids):
        self.mysql_client.add_document(add_id, chunk_id, ...)

    # 3. 持久化到磁盘
    faiss_index_path = os.path.join(FAISS_LOCATION, kb_id, 'faiss_index')
    self.faiss_client.save_local(faiss_index_path)
```

---

## 4. 索引存储结构

```
QANY_DB/faiss/
├── KBabc123/
│   └── faiss_index/    # 索引文件
└── KBdef456/
    └── faiss_index/
```

---

## 5. 搜索流程图

```
用户问题
    ↓
Embedding → [0.1, -0.2, ...]  # 768维向量
    ↓
FAISS搜索 → Top-40 最近邻
    ↓
合并相邻块 → 返回文档列表
    ↓
重排序（Rerank）
```

---

## 6. 练习题

1. FAISS 用的是什么距离度量？（L1 还是 L2？）
2. `fetch_k=200` 是什么意思？为什么不直接返回 `top_k`？
3. 为什么要合并相邻的 chunk？
