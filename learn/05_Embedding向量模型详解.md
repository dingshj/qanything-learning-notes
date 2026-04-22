# 05_Embedding向量模型详解

## 1. 什么是Embedding？

**Embedding（嵌入）** 是把文字变成一串数字的技术。

### 生活类比：给词找"地址"

就像图书馆给每本书一个位置编码：
- 主题相似的书，放在相邻的位置
- "狗"和"猫"（都是动物）位置接近
- "汽车"和"飞机"（都是交通工具）位置接近

### Embedding 就是一个转换器

```
文字 → [0.123, -0.456, 0.789, ..., 0.321]  # 一串数字（向量）
```

---

## 2. 文件结构

```
connector/embedding/
├── embedding_backend.py          # 基类
├── embedding_onnx_backend.py      # ONNX后端（Linux）
└── embedding_torch_backend.py    # PyTorch后端（Mac）
```

---

## 3. 代码解析

### 3.1 基类 embedding_backend.py

```python
class EmbeddingBackend(Embeddings):
    embed_version = "local_v0.0.1_20230525_6d4019f1559aef84abc2ab8257e1ad4c"

    def __init__(self, use_cpu):
        self.use_cpu = use_cpu
        self._tokenizer = AutoTokenizer.from_pretrained(LOCAL_EMBED_PATH)
        self.workers = LOCAL_EMBED_WORKERS
```

**关键方法**：

| 方法 | 作用 |
|------|------|
| `embed_documents()` | 把多个文本向量化 |
| `embed_query()` | 把单个问题向量化 |
| `get_len_safe_embeddings()` | 批量向量化（多线程） |

### 3.2 ONNX后端 embedding_onnx_backend.py

```python
class EmbeddingOnnxBackend(EmbeddingBackend):
    def __init__(self, use_cpu: bool = False):
        super().__init__(use_cpu)
        # 加载ONNX模型
        self._session = InferenceSession(LOCAL_EMBED_MODEL_PATH, ...)

    def get_embedding(self, sentences, max_length):
        # 1. 分词
        inputs_onnx = self._tokenizer(sentences, padding=True, truncation=True, ...)
        # 2. 模型推理
        outputs_onnx = self._session.run(output_names=['output'], input_feed=inputs_onnx)
        # 3. 取[CLS]向量（句子的表示）
        embedding = outputs_onnx[0][:,0]
        # 4. L2归一化（让向量长度都为1）
        embedding = embedding / np.linalg.norm(embedding, axis=1, keepdims=True)
        return embedding.tolist()
```

---

## 4. 核心流程图

```
文本输入
    ↓
分词器（Tokenizer）→ [token_id, token_id, ...]
    ↓
模型推理 → [0.1, -0.2, 0.3, ...]  # 向量
    ↓
归一化 → [0.05, -0.1, 0.15, ...]  # 单位向量
```

---

## 5. 为什么要归一化？

归一化后，向量长度变为1，余弦相似度 = 点积

```
相似度 = cos(θ) = A·B / (|A| × |B|) = A·B  # 归一化后更简单
```

### 相似度计算示例

| 文本对 | 向量点积 | 相似度 |
|--------|----------|--------|
| "狗" vs "猫" | 0.85 | 很高（都是动物） |
| "狗" vs "汽车" | 0.12 | 低（不相关） |

---

## 6. 批量处理

```python
def get_len_safe_embeddings(self, texts: List[str]) -> List[List[float]]:
    all_embeddings = []
    batch_size = LOCAL_EMBED_BATCH

    # 使用线程池并行处理
    with concurrent.futures.ThreadPoolExecutor(max_workers=self.workers) as executor:
        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]
            future = executor.submit(self.get_embedding, batch, LOCAL_EMBED_MAX_LENGTH)
            futures.append(future)
```

---

## 7. 在LocalDocQA中的作用

```python
# local_doc_qa.py
self.embeddings: EmbeddingBackend = EmbeddingOnnxBackend(self.use_cpu)

# 1. 文档入库时：把文本块转成向量
await self.faiss_client.add_document(local_file.docs)

# 2. 用户提问时：把问题转成向量
docs = await self.faiss_client.search(kb_ids, query, ...)
```

---

## 8. 练习题

1. Embedding 的作用是什么？
2. 为什么要做 L2 归一化？
3. 批量处理用了什么技术？
批量处理用了多线程并行技术（ThreadPoolExecutor），把文本分批后同时处理，充分利用多核 CPU 加速！
