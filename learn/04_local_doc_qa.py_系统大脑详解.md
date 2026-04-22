# 04_local_doc_qa.py - 系统大脑详解

## 1. 这个文件是做什么的？

`local_doc_qa.py` 是整个 QAnything 系统的**核心大脑**，负责所有问答逻辑。

### 打个比方：餐厅

| 组件 | 比喻 | 作用 |
|------|------|------|
| LocalDocQA | 大厨+服务员长 | 统筹所有问答流程 |
| embeddings | 点菜助手 | 把文字变成数字向量 |
| llm | 大厨 | 根据食材生成答案 |
| faiss_client | 食材仓库 | 存储和检索文档 |
| local_rerank_backend | 品菜师 | 给答案排序 |
| ocr_reader | 翻译官 | 识别图片中的文字 |

---

## 2. 核心方法

### 2.1 insert_files_to_faiss - 文件入库

文件上传后的处理流程：

```python
async def insert_files_to_faiss(self, user_id, kb_id, local_files):
    for local_file in local_files:
        # 1. 解析文件 → 文本块
        local_file.split_file_to_docs(self.get_ocr_result)
        
        # 2. 向量化并存入 FAISS
        await self.faiss_client.add_document(local_file.docs)
        
        # 3. 更新状态
        self.mysql_client.update_file_status(local_file.file_id, status='green')
```

**文件状态流转**：
- gray（上传中）→ green（成功）/ red（失败）

---

### 2.2 retrieve - 检索流程

```python
async def retrieve(self, query, kb_ids, need_web_search=False):
    # 1. 向量检索
    retrieval_documents = await self.local_doc_search(query, kb_ids)
    
    # 2. 联网搜索（可选）
    if need_web_search:
        retrieval_documents.extend(self.web_page_search(query, top_k=3))
    
    # 3. 重排序
    if len(retrieval_documents) > 1:
        retrieval_documents = self.rerank_documents(query, retrieval_documents)
    
    return retrieval_documents
```

**流程图**：
```
用户问题 → 向量检索 → 获取文档 → 重排序 → 过滤低分 → 返回结果
```

---

### 2.3 rerank_documents - 重排序

让最相关的文档排在最前面：

```python
def rerank_documents(self, query, source_documents):
    if num_tokens(query) > 300:  # 问题太长不使用重排序
        return source_documents
    
    # 计算每个文档的得分
    scores = self.local_rerank_backend.get_rerank(
        query, 
        [doc.page_content for doc in source_documents]
    )
    
    # 按得分排序
    source_documents = sorted(source_documents, key=lambda x: x.metadata['score'], reverse=True)
    return source_documents
```

**为什么要重排序？**
- 向量检索（FAISS）只是"近似匹配"
- 重排序（Rerank）可以更精确地判断相关性
- 就像淘宝搜索：先用关键词搜100件商品，再用算法排序出最相关的10件

---

### 2.4 get_knowledge_based_answer - 生成答案（核心）

整个问答流程的指挥家：

```python
async def get_knowledge_based_answer(self, custom_prompt, query, kb_ids, ...):
    # 步骤1：检索相关文档
    retrieval_documents = await self.retrieve(query, kb_ids)
    
    # 步骤2：根据token限制裁剪文档
    source_documents = self.reprocess_source_documents(...)
    
    # 步骤3：构造提示词
    prompt = self.generate_prompt(query=query, source_docs=source_documents, ...)
    
    # 步骤4：调用LLM生成答案（流式输出）
    async for answer_result in self.llm.generatorAnswer(prompt=prompt, ...):
        yield response, history
```

**完整RAG流程**：
```
Query → 向量检索 → 重排序 → 文档裁剪 → 构造Prompt → LLM生成 → Answer
```

---

## 3. 核心概念

### RAG（Retrieval-Augmented Generation）

检索增强生成 = 先检索，再生成

1. **检索（Retrieval）**：从知识库中找到相关文档
2. **增强（Augmented）**：把检索到的文档作为上下文
3. **生成（Generation）**：让LLM根据上下文生成答案

---

## 4. 练习题

1. 找出 LLM 生成答案的代码位置
2. 如果用户问题超过300个token，会不会使用重排序？
3. 检索时默认返回多少个文档？
