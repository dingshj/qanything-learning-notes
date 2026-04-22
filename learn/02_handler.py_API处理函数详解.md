# 02_handler.py - API处理函数详解

## 1. 这个文件是做什么的？

`handler.py` 是 QAnything 的**业务逻辑中心**，定义了所有 API 接口的具体实现。

### 打个比方
如果把 QAnything 比作一家餐厅：
- `sanic_api.py` = 餐厅的门面和招牌（决定有哪些服务）
- `handler.py` = 餐厅的厨房和员工（具体执行每个服务）

---

## 2. 主要函数一览

| 函数名 | 路由 | 作用 |
|--------|------|------|
| `new_knowledge_base` | `/api/local_doc_qa/new_knowledge_base` | 创建知识库 |
| `upload_files` | `/api/local_doc_qa/upload_files` | 上传文件 |
| `upload_weblink` | `/api/local_doc_qa/upload_weblink` | 上传网页链接 |
| `list_kbs` | `/api/local_doc_qa/list_knowledge_base` | 列出知识库 |
| `list_docs` | `/api/local_doc_qa/list_files` | 列出文件 |
| `delete_docs` | `/api/local_doc_qa/delete_files` | 删除文件 |
| `delete_knowledge_base` | `/api/local_doc_qa/delete_knowledge_base` | 删除知识库 |
| `local_doc_chat` | `/api/local_doc_qa/local_doc_chat` | 问答接口 |
| `new_bot` | `/api/local_doc_qa/new_bot` | 创建Bot |
| `upload_faqs` | `/api/local_doc_qa/upload_faqs` | 上传FAQ |
| `get_qa_info` | `/api/local_doc_qa/get_qa_info` | 获取问答记录 |

---

## 3. 代码逐行解析

### 第1-26行：导入模块

```python
from qanything_kernel.core.local_file import LocalFile
from qanything_kernel.core.local_doc_qa import LocalDocQA
from qanything_kernel.utils.general_utils import *
from qanything_kernel.utils.custom_log import debug_logger, qa_logger
from sanic.response import json as sanic_json
from sanic import request, response
```

**解释**：
- 从各个模块导入需要的类和函数
- `sanic_json` = Sanic框架的JSON响应函数
- `request` = Sanic的请求对象
- `debug_logger` = 调试日志记录器

---

### 第29-51行：`new_knowledge_base` 函数

```python
async def new_knowledge_base(req: request):
    local_doc_qa: LocalDocQA = req.app.ctx.local_doc_qa  # 获取全局问答系统实例
    user_id = safe_get(req, 'user_id')  # 从请求中获取user_id

    # 参数校验
    if user_id is None:
        return sanic_json({"code": 2002, "msg": f'输入非法！...'})
    is_valid = validate_user_id(user_id)
    if not is_valid:
        return sanic_json({"code": 2005, "msg": get_invalid_user_id_msg(user_id=user_id)})

    # 生成知识库ID
    default_kb_id = 'KB' + uuid.uuid4().hex  # UUID是一种唯一ID生成方式
    kb_id = safe_get(req, 'kb_id', default_kb_id)

    # 检查知识库是否已存在
    not_exist_kb_ids = local_doc_qa.mysql_client.check_kb_exist(user_id, [kb_id])
    if not not_exist_kb_ids:
        return sanic_json({"code": 2001, "msg": "fail, knowledge Base {} already exist"})

    # 创建知识库
    local_doc_qa.mysql_client.new_knowledge_base(kb_id, user_id, kb_name)

    # 返回成功响应
    return sanic_json({"code": 200, "msg": "success create knowledge base {}"}...
```

**流程图**：
```
用户请求 → 参数校验 → 检查是否已存在 → 创建知识库 → 返回成功
```

**关键语法解释**：

1. **`async def`** - 异步函数
   - 类比：餐厅可以有多个服务员同时接待不同的客人，而不是一个服务员一次只服务一个人

2. **`uuid.uuid4().hex`** - 生成唯一ID
   - 例子：`KBa1b2c3d4e5f678901234567890abcd`
   - 保证每个知识库都有唯一的ID

3. **`safe_get(req, 'user_id')`** - 安全获取参数
   - 避免直接访问字典导致报错

---

### 第89-155行：`upload_files` 函数（核心！）

这是**最重要的函数之一**，负责文件上传。

```python
async def upload_files(req: request):
    local_doc_qa: LocalDocQA = req.app.ctx.local_doc_qa
    user_id = safe_get(req, 'user_id')
    kb_id = safe_get(req, 'kb_id')
    mode = safe_get(req, 'mode', default='soft')  # soft=不重复上传, strong=强制上传

    # 获取上传的文件列表
    files = req.files.getlist('files')

    # 遍历处理每个文件
    for file, file_name in zip(files, file_names):
        # 1. 生成文件ID
        file_id, msg = local_doc_qa.mysql_client.add_file(...)

        # 2. 创建LocalFile对象（保存文件到磁盘）
        local_file = LocalFile(user_id, kb_id, file, file_id, file_name, ...)

        # 3. 异步入库（解析文本 + 生成向量）
        asyncio.create_task(local_doc_qa.insert_files_to_faiss(user_id, kb_id, [local_file]))

    return sanic_json({"code": 200, "msg": "success..."})
```

**流程图**：
```
用户上传文件 → 接收文件 → 保存到磁盘 → 异步解析 → 入向量库 → 返回成功
```

**关键概念**：

1. **`asyncio.create_task()`** - 创建异步任务
   - 作用：文件上传后，后台异步处理，不阻塞用户
   - 类比：餐厅接单后，后厨开始做菜，服务员可以继续接下一单

2. **文件名清洗**
   ```python
   # 删除全角字符（如中文括号）
   file_name = re.sub(r'[\uFF01-\uFF5E\u3000-\u303F]', '', file_name)
   # 将斜杠替换为下划线
   file_name = file_name.replace("/", "_")
   ```

---

### 第334-480行：`local_doc_chat` 函数（问答接口）

这是**最核心的函数**，处理用户问答。

```python
async def local_doc_chat(req: request):
    # 获取参数
    question = safe_get(req, 'question')  # 用户问题
    kb_ids = safe_get(req, 'kb_ids')      # 知识库ID列表
    history = safe_get(req, 'history', [])  # 对话历史
    streaming = safe_get(req, 'streaming', False)  # 是否流式输出

    # 检查知识库是否有文件
    if len(valid_files) == 0:
        return sanic_json({"code": 200, "msg": "当前知识库为空..."})

    # 调用问答系统获取答案
    async for resp, history in local_doc_qa.get_knowledge_based_answer(
        query=question,
        kb_ids=kb_ids,
        chat_history=history,
        streaming=streaming,
        rerank=rerank,
        need_web_search=need_web_search
    ):
        pass

    return sanic_json({"code": 200, "response": resp["result"], ...})
```

**流程图**：
```
用户提问 → 检索相关文档 → 重排序 → 构造提示词 → 调用LLM → 返回答案
```

---

## 4. 响应码说明

| 响应码 | 含义 |
|--------|------|
| 200 | 成功 |
| 2001 | 知识库/文件已存在 |
| 2002 | 输入参数非法 |
| 2003 | 资源不存在 |
| 2004 | 文件不存在 |
| 2005 | user_id格式错误 |

---

## 5. 常用语法解释

### `sanic_json()` - 返回JSON响应
```python
return sanic_json({"code": 200, "msg": "success", "data": {...}})
```
相当于 Flask 的 `jsonify()`

### `safe_get()` - 安全获取参数
```python
user_id = safe_get(req, 'user_id')  # 获取参数，不存在返回None
mode = safe_get(req, 'mode', default='soft')  # 不存在则返回默认值
```

### 异步生成器 `async for`
```python
async for resp, history in local_doc_qa.get_knowledge_based_answer(...):
    # 每次迭代返回一个chunk的答案（流式输出）
    pass
```

---

## 6. 练习题

### 基础练习
1. 找到 `upload_files` 函数，尝试说出文件上传的完整流程
2. 找出 `local_doc_chat` 函数中的"检查知识库是否有文件"代码的位置

### 进阶练习
1. 如果想限制上传文件的大小，应该在哪里修改？（提示：看 `sanic_api.py`）
2. 如果想让上传的文件自动带上时间戳，代码应该怎么改？

---

## 7. 下一步学习

下一个知识点：**local_file.py - 文件处理详解**

在这个文件中，我们可以学到：
- 文件是如何保存到磁盘的？
- PDF 是如何被解析成文本的？
- 文本是如何被切割成小块的？
