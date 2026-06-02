# 从零开始的大模型复习生活

这是一个按天记录的大模型学习项目，包含 RAG、Query Translation、LangChain、向量数据库等主题的笔记、示例代码和 notebook。

## 项目结构

```text
LLMLearn/
├── pyproject.toml
├── uv.lock
├── day-01/
│   ├── rag_example.ipynb
│   ├── rag_from_scratch_1_to_4.ipynb
│   ├── rag_from_scratch_5_to_9.ipynb
│   └── query-translation/
├── day-02/
│   └── main.py
└── README.md
```

## 环境

项目使用 Python 3.12 和 uv 在根目录统一管理依赖：

```bash
cd /Users/moonhuang/LLMLearn
uv sync
uv run --with jupyter jupyter lab
```

运行某一天的 Python 文件时，也从根目录执行：

```bash
uv run python day-02/main.py
```

## 说明

- `.env` 用于存放本地 API Key，不应提交到 GitHub。
- `.venv/`、`.ipynb_checkpoints/`、`.DS_Store` 等本地文件已通过根目录 `.gitignore` 忽略。
- 每个 `day-*` 目录代表一个独立学习主题或阶段。
