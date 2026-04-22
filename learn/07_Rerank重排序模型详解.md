# 07_Rerank重排序模型详解

## 1. 什么是Rerank？

**Rerank（重排序）** 是在向量检索之后，对结果进行更精确的排序。

### 为什么需要Rerank？

| 阶段 | 作用 | 精度 | 速度 |
|------|------|------|------|
| 向量检索（Embedding） | 快速找到候选集 | 较高 | 快 |
| Rerank | 精细排序 | 最高 | 慢 |

### 生活类比：相亲节目

1. **初筛**：海选，找100个候选人（向量检索）
2. **复筛**：面试精选，找10个最合适的（rerank）

---

## 2. 文件结构

```
connector/rerank/
├── rerank_backend.py          # 基类
├── rerank_onnx_backend.py    # ONNX后端（Linux）
└── rerank_torch_backend.py   # PyTorch后端（Mac）
```

---

## 3. 代码解析

### 3.1 基类 rerank_backend.py

```python
class RerankBackend(ABC):
    def __init__(self, use_cpu):
        self._tokenizer = AutoTokenizer.from_pretrained(LOCAL_RERANK_PATH)
        self.overlap_tokens = 80
        self.batch_size = LOCAL_RERANK_BATCH
        self.max_length = LOCAL_RERANK_MAX_LENGTH
```

### 3.2 核心方法：get_rerank

```python
def get_rerank(self, query: str, passages: List[str]):
    # 1. 构造 [query, passage] 输入对
    tot_batches, merge_inputs_idxs_sort = self.tokenize_preproc(query, passages)
    
    # 2. 批量推理
    tot_scores = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=self.workers) as executor:
        for k in range(0, len(tot_batches), self.batch_size):
            batch = self._tokenizer.pad(tot_batches[k:k + self.batch_size], ...)
            future = executor.submit(self.inference, batch)
            tot_scores.extend(future.result())
    
    # 3. 合并分数（同一文档可能有多个chunk）
    merge_tot_scores = [0 for _ in range(len(passages))]
    for pid, score in zip(merge_inputs_idxs_sort, tot_scores):
        merge_tot_scores[pid] = max(merge_tot_scores[pid], score)
    
    return merge_tot_scores
```

### 3.3 ONNX后端 rerank_onnx_backend.py

```python
class RerankOnnxBackend(RerankBackend):
    def inference(self, batch):
        # 1. 准备输入
        inputs = {
            self.session.get_inputs()[0].name: batch['input_ids'],
            self.session.get_inputs()[1].name: batch['attention_mask']
        }
        
        # 2. 模型推理（输出logits）
        result = self.session.run(None, inputs)
        
        # 3. sigmoid归一化到[0,1]
        sigmoid_scores = 1 / (1 + np.exp(-np.array(result[0])))
        
        return sigmoid_scores.reshape(-1).tolist()
```

---

## 4. 工作流程图

```
问题: "如何安装Python？"
    ↓
向量检索 → Top-40 候选文档
    ↓
Rerank → 精细计算相关性分数
    ↓
排序 → Top-10 最相关文档
    ↓
过滤 → score > 0.35
    ↓
返回结果
```

---

## 5. 关键机制：分数合并

一个长文档被切成多个chunk，每个chunk都和query计算相关性分数：

```python
# 合并策略：取最大值
merge_tot_scores[pid] = max(merge_tot_scores[pid], score)
```

### 为什么取最大值？

- 任一chunk相关 → 文档相关
- 最相关的chunk决定文档的最终分数

---

## 6. 在LocalDocQA中的应用

```python
# local_doc_qa.py
async def retrieve(self, query, kb_ids, need_web_search=False, score_threshold=0.35):
    # 1. 向量检索
    retrieval_documents = await self.local_doc_search(query, kb_ids)
    
    # 2. Rerank（如果文档数>1）
    if len(retrieval_documents) > 1:
        retrieval_documents = self.rerank_documents(query, retrieval_documents)
        # 3. 过滤低分
        tmp_documents = [item for item in retrieval_documents 
                        if float(item.metadata['score']) > score_threshold]
        if tmp_documents:
            retrieval_documents = tmp_documents
    
    return retrieval_documents
```

---

## 7. 练习题

1. Rerank 和向量检索的区别是什么？
2. 为什么要用 sigmoid 函数？
3. 分数合并为什么要取 max 而不是 sum？
