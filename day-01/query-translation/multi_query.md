# Multi Query 检索流程案例

## 背景

用户原始问题：

> 什么是 Task Decomposition？

普通 RAG 会直接用这个问题去向量数据库中检索。

但是 Multi Query 不会只相信用户的原始问法，而是先让 LLM 把这个问题改写成多个不同角度的问题。

例如：

Query 1：What is Task Decomposition?
Query 2：How do agents break complex tasks into smaller steps?
Query 3：Why is task decomposition useful for planning?
Query 4：What methods help agents decompose tasks?

然后每个 Query 都分别去向量数据库中检索相关文档。

---

## 1. 为什么需要 Multi Query

用户的问题和文档中的表达方式可能不一样。

用户可能问：

> What is Task Decomposition?

但是文档里可能写的是：

> Complex tasks can be broken down into smaller and simpler subtasks.

这两句话意思接近，但表达方式不同。

如果只用原始问题检索，可能会漏掉相关文档。

Multi Query 的作用就是：

让 LLM 帮我们生成多个不同问法，从多个角度去检索，提高找到相关文档的概率。

---

## 2. Multi Query 的核心流程

Multi Query 的流程可以理解为：

用户原始问题
↓
LLM 生成多个不同 Query
↓
每个 Query 分别去向量数据库检索
↓
得到多组 Document
↓
合并结果
↓
去重
↓
交给后续 RAG Chain 生成答案

---

## 3. 多个 Query 的生成结果

假设用户问题是：

> What is Task Decomposition?

LLM 可能生成：

Query 1：What is Task Decomposition?
Query 2：How can a complex task be broken into smaller steps?
Query 3：Why do LLM agents use task decomposition?
Query 4：What is the relationship between planning and task decomposition?

这些 Query 都围绕同一个问题，但表达角度不同。

---

## 4. 多个 Query 的检索结果

假设 4 个 Query 分别检索到了下面的文档：

Query 1 检索结果：

第 1 名：Doc A —— Task Decomposition 是把复杂任务拆成小步骤
第 2 名：Doc B —— Chain of Thought 可以帮助模型逐步推理
第 3 名：Doc C —— Agent 会通过规划决定下一步行动

Query 2 检索结果：

第 1 名：Doc A —— Task Decomposition 是把复杂任务拆成小步骤
第 2 名：Doc D —— Subtask 是复杂任务中的子任务
第 3 名：Doc E —— Planning 可以帮助 Agent 安排执行步骤

Query 3 检索结果：

第 1 名：Doc B —— Chain of Thought 可以帮助模型逐步推理
第 2 名：Doc A —— Task Decomposition 是把复杂任务拆成小步骤
第 3 名：Doc F —— ReAct 结合推理和行动

Query 4 检索结果：

第 1 名：Doc C —— Agent 会通过规划决定下一步行动
第 2 名：Doc E —— Planning 可以帮助 Agent 安排执行步骤
第 3 名：Doc A —— Task Decomposition 是把复杂任务拆成小步骤

所以 `documents` 可以理解为：

[
[Doc A, Doc B, Doc C],
[Doc A, Doc D, Doc E],
[Doc B, Doc A, Doc F],
[Doc C, Doc E, Doc A]
]

---

## 5. 代码中的第一步：生成多个 Query

代码通常类似：

generate_queries = (
prompt_perspectives
| llm
| StrOutputParser()
| lambda x: x.split("\n")
)

含义：

先把用户问题交给 prompt，让 LLM 生成多个不同角度的问题。

然后通过：

lambda x: x.split("\n")

把 LLM 输出的一整段文本按换行切开，变成一个 Query 列表。

例如：

原始输出：

What is Task Decomposition?
How can a complex task be broken into smaller steps?
Why do LLM agents use task decomposition?

切分后变成：

[
"What is Task Decomposition?",
"How can a complex task be broken into smaller steps?",
"Why do LLM agents use task decomposition?"
]

---

## 6. 代码中的第二步：多个 Query 分别检索

代码通常类似：

retriever.map()

含义：

把 Query 列表中的每一个 Query 都交给 retriever 检索。

也就是：

Query 1 → retriever 检索 → 得到 [Doc A, Doc B, Doc C]
Query 2 → retriever 检索 → 得到 [Doc A, Doc D, Doc E]
Query 3 → retriever 检索 → 得到 [Doc B, Doc A, Doc F]
Query 4 → retriever 检索 → 得到 [Doc C, Doc E, Doc A]

最终得到一个二维列表：

[
[Doc A, Doc B, Doc C],
[Doc A, Doc D, Doc E],
[Doc B, Doc A, Doc F],
[Doc C, Doc E, Doc A]
]

---

## 7. 为什么要合并结果

因为多个 Query 可能会检索到重复文档。

例如：

Doc A 在多个 Query 中都出现了：

Query 1 中出现 Doc A
Query 2 中出现 Doc A
Query 3 中出现 Doc A
Query 4 中出现 Doc A

这说明 Doc A 很可能是重要文档。

但是在最终传给 LLM 之前，不应该重复塞入同一个文档。

所以需要：

合并多个 Query 的结果，并去重。

---

## 8. 为什么要 dumps(doc)

代码通常类似：

flattened_docs = [dumps(doc) for sublist in documents for doc in sublist]

含义：

把多个 Query 返回的二维 Document 列表拉平成一维列表，并把每个 Document 转成字符串。

原始结构：

[
[Doc A, Doc B, Doc C],
[Doc A, Doc D, Doc E],
[Doc B, Doc A, Doc F]
]

拉平后：

[
Doc A,
Doc B,
Doc C,
Doc A,
Doc D,
Doc E,
Doc B,
Doc A,
Doc F
]

再通过 dumps(doc) 转成字符串，方便后续使用 set() 去重。

---

## 9. 为什么要去重

如果不去重，最终 context 里可能会出现重复内容。

例如：

Doc A 出现了 4 次。

如果全部交给 LLM，会造成：

1. 上下文重复
2. token 浪费
3. 可能挤掉其他有价值的文档
4. LLM 看到的信息不够多样

所以需要去重。

去重后的结果可能是：

[
Doc A,
Doc B,
Doc C,
Doc D,
Doc E,
Doc F
]

---

## 10. 为什么要 loads(doc)

代码通常类似：

unique_docs = [loads(doc) for doc in set(flattened_docs)]

含义：

前面 dumps(doc) 把 Document 对象转成了字符串。

但是后续 RAG Chain 通常需要使用：

doc.page_content

所以需要用 loads(doc) 把字符串重新还原成 Document 对象。

可以理解为：

dumps(doc)：Document → 字符串
loads(doc)：字符串 → Document

---

## 11. Multi Query 和普通 Retriever 的区别

普通 Retriever：

用户问题
↓
检索一次
↓
得到一组文档

Multi Query Retriever：

用户问题
↓
生成多个不同 Query
↓
每个 Query 检索一次
↓
得到多组文档
↓
合并去重
↓
得到更丰富的文档结果

---

## 12. Multi Query 的优势

Multi Query 的优势是：

1. 提高召回率
2. 减少因为用户问法不同导致的漏检
3. 能从多个角度找到相关文档
4. 适合复杂问题、模糊问题、概念解释类问题

比如用户问：

> 什么是 Task Decomposition？

Multi Query 不只搜索 “Task Decomposition”，还可能搜索：

复杂任务如何拆解
Agent 如何规划任务
Subtask 是什么
Planning 和 Task Decomposition 的关系

这样更容易找到完整答案。

---

## 13. Multi Query 的缺点

Multi Query 也有成本：

1. 会多调用一次 LLM 来生成 Query
2. 会多次检索，速度比普通检索慢
3. 可能引入一些不太相关的文档
4. 如果 Query 生成质量不好，检索结果也会受影响

所以它适合需要提高召回率的场景，不一定适合所有简单问题。

---

## 14. Multi Query 和 RAG-Fusion 的区别

Multi Query 的重点是：

生成多个 Query，然后多次检索，最后合并去重。

RAG-Fusion 的重点是：

生成多个 Query，多次检索，然后根据文档在不同检索结果中的排名进行融合打分和重排序。

简单来说：

Multi Query：多问法 + 合并去重
RAG-Fusion：多问法 + 融合打分 + 重新排序

---

## 15. 一句话总结

Multi Query 的核心思想是：

不要只用用户原始问题检索，
而是让 LLM 先生成多个不同角度的问题，
再分别检索，
最后把多个检索结果合并去重，
从而提高 RAG 找到相关文档的概率。
