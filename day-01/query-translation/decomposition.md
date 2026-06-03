# Query Decomposition 问题分解式 RAG 流程案例

## 背景

用户原始问题：

> What are the main components of an LLM-powered autonomous agent system?

这个问题比较宏观，一次检索很难覆盖所有方面。

Decomposition 的做法是：先把原始问题拆分成几个小问题，再分别检索回答，最后综合所有答案。

---

## 1. 核心流程

```
原始问题
   ↓
LLM 生成多个子问题（sub-questions）
   ↓
每个子问题单独检索 + 单独生成答案
   ↓
把所有子问题的 Q+A 汇总
   ↓
LLM 综合所有 Q+A 生成最终答案
```

---

## 2. 拆分出的子问题示例

假设原始问题是：

> What are the main components of an LLM-powered autonomous agent system?

LLM 可能生成：

子问题 1：What is LLM technology and how does it work in autonomous agent systems?  
子问题 2：What are the specific components that make up an autonomous agent system?  
子问题 3：How do the main components interact with each other to enable autonomous behavior?

每个子问题都更聚焦、更容易检索到精准结果。

---

## 3. 两种回答方式

Decomposition 有两种变体：

### 方式一：逐步递推（Answer Recursively）

按顺序依次回答每个子问题。

回答子问题 2 时，会把子问题 1 的答案作为背景知识一起传入 LLM。

流程：

```
回答子问题 1（无背景）
   ↓
得到 Q+A 1
   ↓
回答子问题 2（背景 = Q+A 1）
   ↓
得到 Q+A 2
   ↓
回答子问题 3（背景 = Q+A 1 + Q+A 2）
   ↓
得到最终答案
```

特点：后面的子问题可以利用前面的答案作为上下文，适合有依赖关系的问题。

---

### 方式二：各自独立（Answer Individually）

每个子问题完全独立回答，互不依赖。

流程：

```
子问题 1 → 检索 → 回答 → 得到答案 1
子问题 2 → 检索 → 回答 → 得到答案 2
子问题 3 → 检索 → 回答 → 得到答案 3
   ↓
把所有 Q+A 汇总成 context
   ↓
LLM 综合回答原始问题
```

特点：每个子问题平行处理，速度更快，适合子问题之间相互独立的情况。

---

## 4. 逐步递推方式的 Prompt 结构

方式一的每轮 Prompt 包含三部分：

```
当前子问题：{question}

已有的子问题 + 答案背景（前几轮的 Q+A）：{q_a_pairs}

当前子问题检索到的相关文档：{context}
```

示例（回答子问题 2 时）：

```
当前子问题：
  What are the specific components of an LLM-powered autonomous agent system?

已有背景：
  Question: What is LLM technology?
  Answer: LLM is a large language model that can generate text...

当前检索到的文档：
  Doc A: The main components include planning, memory, and tool use...
```

LLM 同时参考已有背景和当前检索结果来生成答案。

---

## 5. 各自独立方式的最终 Prompt 结构

方式二的最终汇总 Prompt 包含：

```
一组 Q+A pairs：{context}

原始问题：{question}
```

示例：

```
Q+A pairs:
  Question 1: What is LLM technology?
  Answer 1: LLM is a large language model...

  Question 2: What are the specific components?
  Answer 2: The components include planning, memory, and tool use...

  Question 3: How do components interact?
  Answer 3: They work together to enable autonomous behavior...

Original Question:
  What are the main components of an LLM-powered autonomous agent system?
```

LLM 把这些 Q+A 当作背景知识，综合生成最终答案。

---

## 6. Decomposition 和 Multi Query 的区别

Multi Query：

```
一个问题 → 多个不同问法 → 多次检索 → 合并去重 → 生成答案
```

目标是：用不同问法找到更多相关文档，提高召回率。

Decomposition：

```
一个复杂问题 → 拆成多个子问题 → 分别检索回答 → 综合答案
```

目标是：把一个复杂问题拆成多个简单问题，分步骤弄清楚，提高回答的深度和准确性。

---

## 7. 适用场景

Decomposition 适合：

1. 问题本身比较复杂，涉及多个方面
2. 问题可以自然地分解成相互独立或有先后关系的子问题
3. 需要深度回答，而不只是找到一段相关文本

不适合：

1. 简单问题（一次检索就能回答）
2. 子问题之间边界不清晰，容易生成重叠的子问题

---

## 8. 一句话总结

Decomposition 的核心思想是：

把一个复杂的大问题拆成几个小问题，  
分别检索并回答每个小问题，  
再把所有答案综合起来，  
从而给出更全面、更深入的最终答案。
