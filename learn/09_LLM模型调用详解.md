# 09_LLM模型调用详解

## 1. 概述

LLM（Large Language Model）模块负责调用大语言模型生成答案。

---

## 2. 文件结构

```
connector/llm/
├── base/
│   └── base.py              # 基类
├── llm_for_openai_api.py    # OpenAI API 后端
├── llm_for_fastchat.py      # FastChat (vLLM) 后端
└── llm_for_llamacpp.py      # Llama.cpp 后端
```

---

## 3. 基类定义 (base.py)

### 消息实体

```python
class AnswerResult:
    """消息实体"""
    history: List[List[str]] = []      # 对话历史 [[问, 答], [问, 答]]
    llm_output: Optional[dict] = None  # LLM输出
    prompt: str = ""                    # 提示词
```

### 抽象基类

```python
class BaseAnswer(ABC):
    """上层业务包装器"""
    
    @property
    @abstractmethod
    def _history_len(self) -> int:
        """返回历史长度"""
    
    @abstractmethod
    def set_history_len(self, history_len: int) -> None:
        """设置历史长度"""
    
    def generatorAnswer(self, prompt, history, streaming):
        """生成答案"""
        pass
```

---

## 4. OpenAI API 实现

### 核心配置

```python
class OpenAILLM(BaseAnswer):
    model: str = None
    token_window: int = None       # 上下文窗口
    max_token: int = 512            # 最大生成长度
    temperature: float = 0          # 温度（创造性）
    top_p: float = 0.95             # top_p采样
    history_len: int = 10            # 保留历史轮数
    
    def __init__(self, args):
        self.client = OpenAI(
            base_url=args.openai_api_base,
            api_key=args.openai_api_key
        )
        self.model = args.openai_api_model_name
```

### 核心方法：生成答案

```python
async def generatorAnswer(self, prompt, history=[], streaming=False):
    # 1. 构建消息列表
    messages = []
    for pair in history:
        question, answer = pair
        messages.append({"role": "user", "content": question})
        messages.append({"role": "assistant", "content": answer})
    messages.append({"role": "user", "content": prompt})
    
    # 2. 调用API
    response = self.client.chat.completions.create(
        model=self.model,
        messages=messages,
        stream=streaming,
        max_tokens=self.max_token,
        temperature=self.temperature,
    )
    
    # 3. 返回结果
    return response
```

---

## 5. FastChat (vLLM) 实现

使用 vLLM 作为推理后端，支持高并发推理。

```python
class OpenAICustomLLM(BaseAnswer):
    def __init__(self, args):
        # 使用 AsyncLLMEngine
        engine_args = AsyncEngineArgs.from_cli_args(args)
        self.engine = AsyncLLMEngine.from_engine_args(engine_args)
        
        # 采样参数
        self.sampling_params = SamplingParams(
            temperature=0.6,
            top_p=0.8,
            max_tokens=512
        )
```

### Conversation Template

```python
# 使用对话模板构建 prompt
conv = get_conv_template(self.conv_template)
for pair in history:
    question, answer = pair
    conv.append_message(conv.roles[0], question)  # user
    conv.append_message(conv.roles[1], answer)    # assistant
conv.append_message(conv.roles[0], prompt)
prompt = conv.get_prompt()
```

---

## 6. Llama.cpp 实现

使用 llama-cpp-python，支持本地部署。

```python
class LlamaCPPCustomLLM(BaseAnswer):
    def __init__(self, args):
        self.llm = Llama(
            model_path=args.model,
            n_gpu_layers=-1,  # 使用GPU
            n_ctx=4096        # 上下文长度
        )
```

---

## 7. 流式输出 (Streaming)

### 什么是流式输出？

不是等模型完全生成后再返回，而是一边生成一边返回。

```python
async def _call(self, prompt, history, streaming=True):
    if streaming:
        response = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            stream=True  # 开启流式
        )
        for event in response:
            delta = event["choices"][0]["delta"]["content"]
            yield f"data: {json.dumps({'answer': delta})}\n\n"
    yield "data: [DONE]\n\n"
```

### SSE 格式

```json
data: {"answer": "今天"}
data: {"answer": "天气"}
data: {"answer": "很好"}
data: [DONE]
```

---

## 8. Token 计算

使用 tiktoken 计算 token 数量：

```python
def num_tokens_from_messages(self, messages):
    encoding = tiktoken.encoding_for_model("gpt-3.5-turbo")
    num_tokens = 0
    for message in messages:
        num_tokens += len(encoding.encode(message["content"]))
    num_tokens += 3  # 特殊token
    return num_tokens
```

---

## 9. 在LocalDocQA中的作用

```python
# local_doc_qa.py
async def local_doc_qa_remaining_operations(self, query, prompt, retrieval_documents):
    # 1. 构建 prompt
    prompt = self.prompt_template.format(
        context=context,
        query=query
    )
    
    # 2. 调用 LLM
    async for answer_result in self.llm.generatorAnswer(
        prompt=prompt,
        history=chat_history,
        streaming=True
    ):
        yield answer_result
    
    # 3. 解析结果
    answer = answer_result.llm_output["answer"]
```

---

## 10. 参数对比

| 参数 | OpenAI | vLLM | Llama.cpp |
|------|--------|------|-----------|
| temperature | 0 | 0.6 | 0.7 |
| top_p | 0.95 | 0.8 | 0.8 |
| max_tokens | 512 | 512 | 512 |

---

## 11. 练习题

1. 为什么需要 `history_len` 参数？它有什么作用？
2. 什么是流式输出？相比非流式有什么优势？
3. Token 计算为什么用 tiktoken 而不是直接数字符数？
