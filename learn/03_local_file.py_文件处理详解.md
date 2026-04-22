# 03_local_file.py - 文件处理详解

## 1. 这个文件是做什么的？

`local_file.py` 负责 QAnything 的**文件处理流程**，包括：
1. **文件保存** - 把上传的文件保存到磁盘
2. **文件解析** - 把 PDF、Word、图片等格式转换成文本
3. **文本切割** - 把长文本切成小块（chunks）

### 打个比方
就像一个**图书管理员**的工作：
1. 把书籍放进书架（保存文件）
2. 理解书中的内容（解析文本）
3. 给书做摘要索引（切割文本）

---

## 2. 核心类：LocalFile

```python
class LocalFile:
    def __init__(self, user_id, kb_id, file, file_id, file_name, embedding, is_url=False, in_milvus=False):
        # 初始化文件信息
        pass

    def split_file_to_docs(self, ocr_engine):
        # 解析文件并切割成文档块
        pass
```

---

## 3. 文本切割器配置（第28-39行）

```python
text_splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", "。", "!", "！", "?", "？", "；", ";", "……", "…", "、", "，", ",", " ", ""],
    chunk_size=400,      # 每个块400个token
    chunk_overlap=100,   # 相邻块重叠100个token
    length_function=num_tokens,
)

pdf_text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,      # PDF块更大，800个token
    chunk_overlap=0,     # PDF块不重叠
    length_function=num_tokens
)
```

**关键概念**：
- **separators** - 分隔符列表，按优先级尝试切割
- **chunk_size** - 每个块的最大长度
- **chunk_overlap** - 相邻块重叠内容，避免重要内容被切断

---

## 4. 文件保存逻辑（第51-83行）

```python
class LocalFile:
    def __init__(self, user_id, kb_id, file, file_id, file_name, ...):
        # 构建保存路径
        upload_path = os.path.join(UPLOAD_ROOT_PATH, user_id)   # QANY_DB/content/用户ID
        file_dir = os.path.join(upload_path, self.file_id)     # QANY_DB/content/用户ID/文件ID
        os.makedirs(file_dir, exist_ok=True)

        # 保存文件
        self.file_path = os.path.join(file_dir, self.file_name)
        with open(self.file_path, "wb+") as f:
            f.write(self.file_content)
```

**文件保存路径结构**：
```
QANY_DB/content/
└── 用户ID/
    └── 文件ID/
        └── 文件名.pdf
```

---

## 5. split_file_to_docs 函数（第148-212行）

根据文件类型选择不同的解析器：

```python
if self.file_path.lower().endswith(".pdf"):
    loader = UnstructuredPaddlePDFLoader(...)
elif self.file_path.lower().endswith(".txt"):
    loader = TextLoader(...)
elif self.file_path.lower().endswith(".jpg"):
    loader = UnstructuredPaddleImageLoader(...)
# ... 更多文件类型
```

**支持的文件类型**：
- .pdf - UnstructuredPaddlePDFLoader
- .txt - TextLoader
- .md - UnstructuredFileLoader
- .docx - UnstructuredWordDocumentLoader
- .jpg/.png/.jpeg - UnstructuredPaddleImageLoader
- .xlsx - pandas.read_excel + CSVLoader
- .pptx - UnstructuredPowerPointLoader
- .csv - CSVLoader
- .eml - UnstructuredEmailLoader
- .mp3/.wav - UnstructuredPaddleAudioLoader

---

## 6. 二次切割（第219-239行）

解析后可能还需要二次切割：

```python
# 合并过短的片段
new_docs = []
min_length = 200  # 最小长度200 token
for doc in docs:
    if num_tokens(last_doc.page_content) + num_tokens(doc.page_content) < min_length:
        last_doc.page_content += '\n' + doc.page_content  # 合并
    else:
        new_docs.append(doc)

# 按配置大小切割
docs = text_splitter.split_documents(new_docs)
```

---

## 7. 添加元数据（第241-262行）

每个文本块都需要带上"身份证"：

```python
for idx, doc in enumerate(docs):
    new_doc.metadata["user_id"] = self.user_id
    new_doc.metadata["kb_id"] = self.kb_id
    new_doc.metadata["file_id"] = self.file_id
    new_doc.metadata["chunk_id"] = idx
```

---

## 8. 整体流程图

```
用户上传文件
    ↓
LocalFile.__init__() - 保存文件到磁盘
    ↓
split_file_to_docs() - 解析并切割
    ├─ 选择解析器
    ├─ 解析文件 → 原始文本
    ├─ 二次切割
    └─ 添加元数据
    ↓
返回 Document 列表
    ↓
交给向量数据库（FAISS）存储
```

---

## 9. 练习题

1. PDF 的 chunk_size 是多少？普通文本的呢？
2. 如果想支持新的文件格式（如 .epub），需要在哪个函数里添加代码？
3. 为什么要设置 chunk_overlap？如果设置成 0 会怎样？
