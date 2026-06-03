# HyDE 假设文档嵌入式 RAG 流程案例

## 背景

用户原始问题：

> What is task decomposition for LLM agents?

普通 RAG 会直接把这个问题转成向量，然后去向量库里找语义相近的文档。

但问题有个天然的缺陷：

> 问题的向量 ≠ 答案文档的向量

问题通常很短，而文档通常较长，语义空间上两者可能距离较远，导致检索效果不理想。

HyDE（Hypothetical Document Embeddings）的做法是：先让 LLM 生成一段假设答案，再用假设答案的向量去检索，而不是用原始问题的向量。

---

## 1. 核心流程

```
用户问题
   ↓
LLM 生成一段假设答案（Hypothetical Document）
   ↓
对假设答案做 Embedding（向量化）
   ↓
用假设答案的向量去向量数据库检索
   ↓
找到真实的相关文档
   ↓
LLM 根据真实文档生成最终答案
```

---

## 2. 什么是假设答案

假设答案是 LLM 根据问题"凭空"生成的一段文字，不一定是正确答案，但语义上接近真实答案的风格和措辞。

示例：

用户问题：

> What is task decomposition for LLM agents?

LLM 生成的假设答案：

> Task decomposition is a fundamental concept in reinforcement learning and AI. It refers to the process of breaking down a complex task into smaller, more manageable sub-tasks. For LLM agents, this allows them to effectively tackle complex problems by focusing on individual components sequentially...

这段假设答案不一定完全正确，但它的表达风格和真实文档相近，向量更接近。

---

## 3. 为什么用假设答案而不是问题本身

向量检索的核心是：

> 找到向量空间中语义最相近的内容

问题的向量 vs 答案文档的向量：

```
用户问题（短、疑问句、关键词稀少）
  → 向量 A

真实文档（长、叙述句、内容详细）
  → 向量 B

假设答案（中等长度、叙述句、语义接近文档）
  → 向量 C
```

向量 C 和向量 B 在语义空间里更接近，所以用假设答案检索，能找到更相关的文档。

---

## 4. HyDE 的 Prompt 设计

为了让 LLM 生成接近真实文档风格的内容，Prompt 通常这样写：

```
Please write a scientific paper passage to answer the question

Question: {question}
Passage:
```

注意这里用的是 "scientific paper passage"（科学论文段落），目的是让生成内容更接近文档库中的写作风格。

---

## 5. 假设答案只用于检索，不用于回答

这是 HyDE 的重要特点：

假设答案只用来做向量检索，找到真实文档。

LLM 最终回答问题时，用的是从向量库中检索到的真实文档，而不是假设答案。

流程对比：

```
普通 RAG：
  问题向量 → 检索 → 真实文档 → 回答

HyDE：
  问题 → LLM 生成假设答案 → 假设答案向量 → 检索 → 真实文档 → 回答
```

---

## 6. 具体例子

假设向量库里有以下文档：

Doc A：Task Decomposition is the process of breaking down complex tasks...  
Doc B：Chain of Thought transforms big tasks into multiple manageable tasks...  
Doc C：Tree of Thoughts extends CoT by exploring multiple reasoning possibilities...

用户问题直接检索，向量相似度较低，可能只找到 Doc A。

LLM 生成的假设答案很长，包含了"breaking down tasks"、"sub-tasks"、"manageable components"等词，向量检索时相似度更高，可能同时找到 Doc A、Doc B、Doc C。

---

## 7. HyDE 和其他技术的比较

| 技术 | 做法 | 目标 |
|------|------|------|
| 普通 RAG | 直接用问题向量检索 | 简单直接 |
| Multi Query | 生成多个不同问法 | 提高召回率 |
| Step Back | 生成更抽象的问题 | 补充背景知识 |
| HyDE | 生成假设答案再检索 | 缩短问题向量与文档向量的距离 |

---

## 8. HyDE 的优缺点

优点：

1. 解决了问题和文档语义空间不一致的问题
2. 不需要修改向量库或检索器
3. 适用于专业领域（法律、医学、学术论文等）

缺点：

1. 额外调用一次 LLM 生成假设答案，增加延迟和 token 消耗
2. 如果 LLM 生成的假设答案方向偏差较大，可能反而检索到不相关文档
3. 对于简单问题，效果提升不明显

---

## 9. 一句话总结

HyDE 的核心思想是：

不用问题的向量去检索，  
而是先让 LLM 生成一段假设答案，  
用假设答案的向量去找语义相近的真实文档，  
再用真实文档来回答原始问题，  
从而弥补问题和文档在语义空间中的距离差距。
