# RAG-Fusion / RRF 重排序流程案例

## 背景

用户原始问题：

> 什么是 Task Decomposition？

系统不会只用这一个问题去检索，而是先通过 LLM 生成多个不同角度的问题。

例如：

Query 1：What is Task Decomposition?  
Query 2：How do agents break complex tasks into smaller steps?  
Query 3：Why is task decomposition useful for planning?

然后每个 Query 都分别去向量数据库中检索相关文档。

---

## 1. 多个 Query 的检索结果

假设 3 个 Query 分别检索到了下面的文档：

Query 1 检索结果：

第 1 名：Doc A —— Task Decomposition 是把复杂任务拆成小步骤  
第 2 名：Doc B —— Chain of Thought 可以帮助模型逐步推理  
第 3 名：Doc C —— Agent 会通过规划决定下一步行动  

Query 2 检索结果：

第 1 名：Doc B —— Chain of Thought 可以帮助模型逐步推理  
第 2 名：Doc A —— Task Decomposition 是把复杂任务拆成小步骤  
第 3 名：Doc D —— Memory 可以帮助 Agent 保存历史信息  

Query 3 检索结果：

第 1 名：Doc A —— Task Decomposition 是把复杂任务拆成小步骤  
第 2 名：Doc C —— Agent 会通过规划决定下一步行动  
第 3 名：Doc B —— Chain of Thought 可以帮助模型逐步推理  

所以 `results` 可以理解为：

[
  [Doc A, Doc B, Doc C],
  [Doc B, Doc A, Doc D],
  [Doc A, Doc C, Doc B]
]

---

## 2. 代码中的第一层循环

代码：

for docs in results:

含义：

依次处理每一个 Query 的检索结果。

也就是：

先处理 Query 1 的结果：  
[Doc A, Doc B, Doc C]

再处理 Query 2 的结果：  
[Doc B, Doc A, Doc D]

最后处理 Query 3 的结果：  
[Doc A, Doc C, Doc B]

---

## 3. 代码中的第二层循环

代码：

for rank, doc in enumerate(docs):

含义：

遍历当前 Query 检索出来的每个文档，同时记录它的排名。

注意：rank 从 0 开始。

所以：

第 1 名 → rank = 0  
第 2 名 → rank = 1  
第 3 名 → rank = 2  

---

## 4. 为什么要 dumps(doc)

代码：

doc_str = dumps(doc)

含义：

把 Document 对象转换成字符串。

原因是：

Document 对象本身不太适合作为字典的 key，  
所以先把它序列化成字符串，方便存入 `fused_scores` 字典中进行统计和去重。

可以理解为：

Doc A → "Doc A 的字符串形式"  
Doc B → "Doc B 的字符串形式"  

---

## 5. fused_scores 的作用

代码：

if doc_str not in fused_scores:
    fused_scores[doc_str] = 0

含义：

如果这个文档之前没有出现过，就先给它一个初始分数 0。

`fused_scores` 可以理解为一个记分板：

{
  Doc A: 分数,
  Doc B: 分数,
  Doc C: 分数,
  Doc D: 分数
}

---

## 6. RRF 打分公式

代码：

fused_scores[doc_str] += 1 / (rank + k)

含义：

根据文档在当前 Query 结果中的排名，给它加分。

假设 k = 60：

第 1 名：1 / (0 + 60) = 0.01667  
第 2 名：1 / (1 + 60) = 0.01639  
第 3 名：1 / (2 + 60) = 0.01613  

排名越靠前，分数越高。

但是 RRF 的重点不是只奖励某一次第一名，而是奖励：

1. 在多个 Query 中反复出现的文档
2. 在多个 Query 中排名都比较靠前的文档

---

## 7. 文档分数如何累加

Doc A 出现了 3 次：

Query 1 中：Doc A 第 1 名 → 加 0.01667  
Query 2 中：Doc A 第 2 名 → 加 0.01639  
Query 3 中：Doc A 第 1 名 → 加 0.01667  

Doc A 总分：

0.01667 + 0.01639 + 0.01667 = 0.04973

---

Doc B 也出现了 3 次：

Query 1 中：Doc B 第 2 名 → 加 0.01639  
Query 2 中：Doc B 第 1 名 → 加 0.01667  
Query 3 中：Doc B 第 3 名 → 加 0.01613  

Doc B 总分：

0.01639 + 0.01667 + 0.01613 = 0.04919

---

Doc C 出现了 2 次：

Query 1 中：Doc C 第 3 名 → 加 0.01613  
Query 3 中：Doc C 第 2 名 → 加 0.01639  

Doc C 总分：

0.01613 + 0.01639 = 0.03252

---

Doc D 只出现了 1 次：

Query 2 中：Doc D 第 3 名 → 加 0.01613  

Doc D 总分：

0.01613

---

## 8. 最终重新排序

代码：

reranked_results = [
    (loads(doc), score)
    for doc, score in sorted(fused_scores.items(), key=lambda x: x[1], reverse=True)
]

含义：

把所有文档按照融合后的分数从高到低排序。

最终结果：

第 1 名：Doc A，分数 0.04973  
第 2 名：Doc B，分数 0.04919  
第 3 名：Doc C，分数 0.03252  
第 4 名：Doc D，分数 0.01613  

其中：

loads(doc) 的作用是把之前 dumps(doc) 得到的字符串重新还原成 Document 对象。

---

## 9. 这段代码的核心流程

这段代码做了 4 件事：

1. 遍历多个 Query 的检索结果
2. 记录每个文档在不同 Query 中的排名
3. 使用 RRF 公式给文档累积分数
4. 按融合分数从高到低重新排序

---

## 10. 一句话总结

RAG-Fusion / RRF 的核心思想是：

不是只相信某一次检索结果，  
而是综合多个 Query 的检索结果。

如果一个文档在多个 Query 中都出现，  
并且经常排在前面，  
它就会获得更高的融合分数，  
最终更有可能被送给 LLM 作为上下文。