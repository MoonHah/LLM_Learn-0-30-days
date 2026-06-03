# Step Back 退一步提问式 RAG 流程案例

## 背景

用户原始问题：

> What is task decomposition for LLM agents?

这个问题比较具体。有时候向量库里并没有刚好命中这个问法的文档，但可能有更宏观的背景资料能帮助理解。

Step Back 的做法是：先让 LLM 把原始问题"退一步"，生成一个更抽象、更通用的版本，然后同时用两个问题去检索，把两组结果合并作为上下文。

---

## 1. 核心流程

```
原始问题
   ↓
LLM 生成一个更抽象的 Step Back 问题
   ↓
用原始问题检索 → 得到 normal_context
用 Step Back 问题检索 → 得到 step_back_context
   ↓
把两组上下文合并，交给 LLM 生成最终答案
```

---

## 2. 什么叫"退一步"

具体问题通常只命中极少数文档，而更抽象的问题能覆盖更广泛的背景知识。

示例：

原始问题（具体）：

> What is task decomposition for LLM agents?

Step Back 问题（抽象）：

> What is the process of breaking down tasks for LLM agents?

再举一个例子：

原始问题：

> Could the members of The Police perform lawful arrests?

Step Back 问题：

> What can the members of The Police do?

可以看到：Step Back 问题去掉了细节，保留了核心概念，变成了一个更通用的问法。

---

## 3. Few-shot 的作用

为了让 LLM 生成合适的 Step Back 问题，代码里会提供几个示例（Few-shot）。

示例格式：

```
原始问题：Jan Sindel's was born in what country?
Step Back 问题：What is Jan Sindel's personal history?

原始问题：Could the members of The Police perform lawful arrests?
Step Back 问题：What can the members of The Police do?
```

通过这几个示例，LLM 能理解"退一步"的含义，生成风格合适的抽象问题。

---

## 4. 双路检索

Step Back 会同时发出两次检索请求：

检索 1（原始问题）：

> What is task decomposition for LLM agents?

可能找到：Doc A（直接讲 Task Decomposition 的内容）

检索 2（Step Back 问题）：

> What is the process of breaking down tasks for LLM agents?

可能找到：Doc B（讲 Planning 的整体概念）、Doc C（讲 Chain of Thought 的背景）

---

## 5. 合并上下文生成答案

最终 Prompt 结构：

```
正常检索的结果（normal_context）：
  {原始问题检索到的文档}

Step Back 检索的结果（step_back_context）：
  {抽象问题检索到的文档}

原始问题：
  {question}
```

LLM 同时参考两组上下文，给出更完整的答案。

如果两组结果有矛盾，LLM 会选择与两者都不矛盾的方向作答。

如果某组结果不相关，LLM 会忽略它，只用相关的部分。

---

## 6. Step Back 和 Multi Query 的区别

Multi Query：

```
一个问题 → 生成多个相似问法 → 多次检索 → 合并结果
```

目标：用不同措辞提高召回率。

Step Back：

```
一个问题 → 生成一个更抽象的问题 → 两次检索 → 合并上下文
```

目标：用更宏观的视角补充背景知识，让 LLM 理解更深入。

---

## 7. Step Back 的适用场景

适合：

1. 问题很具体，但背景知识也很重要
2. 需要先理解宏观概念，再回答具体细节
3. 向量库里的文档按主题分布，具体问题可能找不到精准命中的文档

不适合：

1. 问题本身已经足够宏观
2. 只需要找到具体事实，不需要背景知识

---

## 8. 一句话总结

Step Back 的核心思想是：

在回答具体问题之前，先退一步问一个更抽象的问题，  
用宏观问题补充背景知识，  
再结合原始问题的检索结果，  
让 LLM 在更完整的信息基础上给出更深入的答案。
