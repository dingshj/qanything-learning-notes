# QAnything 源码学习笔记

> 网易开源 RAG 框架 · 源码深度分析

本仓库是我学习 [QAnything](https://github.com/netease-youdao/QAnything) 过程中整理的源码笔记，涵盖文档入库、向量化、检索、重排序、LLM调用等完整链路。

---

## 📚 笔记目录

| 编号 | 文件名 | 内容说明 |
|------|--------|----------|
| 01 | 项目介绍与环境准备.md | 项目概述、环境配置、启动流程 |
| 02 | handler.py API处理函数详解.md | 路由注册、请求处理、upload_files / local_doc_chat |
| 03 | local_file.py 文件处理详解.md | LocalFile 类、文件解析、split_file_to_docs |
| 04 | local_doc_qa.py 系统大脑详解.md | LocalDocQA 核心类、retrieve、get_knowledge_based_answer |
| 05 | Embedding向量模型详解.md | EmbeddingBackend、ONNX 推理、CLS token、L2归一化 |
| 06 | FAISS向量数据库详解.md | FaissClient、索引加载、增删查、merge_docs |
| 07 | Rerank重排序模型详解.md | Cross-Encoder、Sigmoid、max合并、阈值过滤 |
| 08 | MySQL数据库详解.md | 元数据存储、文件映射、状态管理 |
| 09 | LLM模型调用详解.md | 大模型调用、流式输出、prompt构造 |

---

## 🛠 技术栈

- **语言**：Python
- **框架**：Sanic、LangChain
- **向量库**：FAISS
- **推理加速**：ONNX Runtime
- **模型**：Embedding（BGE）、Rerank（Cross-Encoder）
- **数据库**：MySQL

---

## 🎯 学习收获

- 深入理解 RAG 系统的完整工程实现
- 掌握文档解析 → 向量化 → 检索 → 重排序 → LLM生成的全链路
- 熟悉 ONNX 模型推理加速和 FAISS 向量检索优化
- 具备独立分析和调试开源项目的能力

---

## 📎 相关链接

- [QAnything 官方仓库](https://github.com/netease-youdao/QAnything)

---

*持续更新中*
