# 08_MySQL数据库详解

## 1. 概述

**注意**：虽然文件名是 `mysql_client.py`，但实际使用的是 **SQLite** 数据库！

SQLite 是一个轻量级的关系型数据库，无需安装服务器，适合本地存储。

---

## 2. 数据表结构

### 2.1 表关系图

```
User (用户)
    ↓ 1:N
KnowledgeBase (知识库)
    ↓ 1:N
File (文件)
    ↓ 1:N
Document (文档块)
```

### 2.2 各表详解

| 表名 | 作用 | 关键字段 |
|------|------|----------|
| User | 用户 | user_id, user_name |
| KnowledgeBase | 知识库 | kb_id, user_id, kb_name |
| File | 上传文件 | file_id, kb_id, status, file_name |
| Document | 文档块(chunks) | docstore_id, chunk_id, file_id |
| QanythingBot | 机器人配置 | bot_id, prompt_setting, kb_ids_str |
| Faqs | FAQ问答对 | faq_id, question, answer |
| QaLogs | 问答日志 | qa_id, query, result, prompt |

---

## 3. 代码解析

### 3.1 数据库连接

```python
class KnowledgeBaseManager:
    def __init__(self):
        self.database = SQLITE_DATABASE
        self.create_tables_()

    def execute_query_(self, query, params, commit=False, fetch=False):
        conn = sqlite3.connect(self.database)
        conn.execute('PRAGMA foreign_keys = ON')
        cursor = conn.cursor()
        cursor.execute(query, params)
        
        if commit:
            conn.commit()
        if fetch:
            result = cursor.fetchall()
        
        cursor.close()
        conn.close()
        return result
```

### 3.2 创建表结构

```python
def create_tables_(self):
    # 用户表
    query = """CREATE TABLE IF NOT EXISTS User (
        user_id VARCHAR(255) PRIMARY KEY,
        user_name VARCHAR(255)
    );"""
    
    # 知识库表
    query = """CREATE TABLE IF NOT EXISTS KnowledgeBase (
        kb_id VARCHAR(255) PRIMARY KEY,
        user_id VARCHAR(255),
        kb_name VARCHAR(255),
        deleted INTEGER DEFAULT 0,
        FOREIGN KEY (user_id) REFERENCES User(user_id)
    );"""
```

---

## 4. 核心操作

### 4.1 创建知识库

```python
def new_knowledge_base(self, kb_id, user_id, kb_name):
    if not self.check_user_exist_(user_id):
        self.add_user_(user_id, user_name)
    query = "INSERT INTO KnowledgeBase VALUES (?, ?, ?)"
    self.execute_query_(query, (kb_id, user_id, kb_name), commit=True)
    return kb_id, "success"
```

### 4.2 添加文件

```python
def add_file(self, user_id, kb_id, file_name, timestamp, status="gray"):
    file_id = uuid.uuid4().hex
    query = "INSERT INTO File VALUES (?, ?, ?, ?, ?)"
    self.execute_query_(query, (file_id, kb_id, file_name, status, timestamp), commit=True)
    return file_id, "success"
```

---

## 5. 文件状态流转

```
gray (上传中)
    ↓ 成功
green (已完成)
    ↓ 失败
red (失败)
```

---

## 6. 练习题

1. 为什么要用 SQLite 而不是 MySQL？
2. File 表中的 status 有哪些值？分别代表什么？
3. QaLogs 表的作用是什么？
