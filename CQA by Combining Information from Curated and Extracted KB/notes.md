##结合 Curated / Extracted KB 回答复杂问题的方法

[TOC]

### 主要工作

- 提出了一个全新的 KB-QA 系统，MULTIQUE。
  结合了 curated / extracted knowledge bases 以回答复杂问题。这是从多信息源优化答案的**第一次尝试**。
- 为了利用多个 KB 的信息，创建了使用简单查询完成困难查询的问题模式。每个简单查询对应一个特定的 KB。
- 提出了基于神经网络的模型。用于将多个 KB 中的关系对齐以进行集体推理。
- 评测

同样的，本文提到多数现有的系统着眼于单关系的简单问题，而对于多实体关系的复杂问题没有很深入的研究。于是这篇文章也是比较新的工作，不是非常成熟。



### 一些概念

- KB 很常见的概念
  - curated KB 指利用 ontological relations ，覆盖面较小。类似于将关系的宾语设置为主语的一个域。实体和关系唯一标号。
  - extracted KB 着眼于多样的描述形式（自然语言），没有归一化。
- 复杂问题。
  一个问题 Q 相当于一个短问题集合 G。它有多于一个的关系和关注点（答案或已知实体等）。
  短问题可以通过不同的条件聚合在一起。
  - 短问题有四要素：
    种子实体 s，理解为问题的主语
    变量节点 o，理解为问题的宾语（答案）
    一个关系 (s, p, o)，这个关系可以是任意一种 KB 中的
    和一些约束，一个实体通过另一个关系连接到主关系上的
  - composition tree 组成树
    描述了 query G 是如何根据部分询问产生并评价的
    包含两个函数
    - simQA 寻找简单询问。对于每个**候选者**，寻找简单询问，和问题的自然语言表达作比较。寻找最合适的候选者。
    - join 合并两个 partial query. 
      也就是多个 partial query 是否共用相同的 query focus 或variable node
    - 可以参考一下 example1 和 figure 2



### Overview

给一个 complex question：

- 计算一棵 composition tree ，描述如何将复杂问题拆分成简单问题
- 对于所有的简单问题，在 所有 KB 上找到答案，我们称之为候选者。
- 对于每个候选者，我们评估它与原自然语言问题的相似程度。我们使用一个基于神经网络的模型来做这件事，但是一定要适配多种（2种）不同形式的关系。也就是 curated 和 extracted
- 接下来我们要整合所有的部分询问来找到整个 complex question 的答案。于是就可能有很多种方法来回答 complex question，因为关系形式多种多样。于是我们就依据 partial query，query structure，entity linking scores 找到最合适的回答方案。



### 具体工作流程

#### partial query candidate generation

相比于其他的已有的方法，尤其指默认关系为单跳的那些，该方案可以解决多跳问题。

- 识别种子实体：a linked entity 或 一个之前已经被评估过的 partial query 的答案
- 识别主线关系：给定一个种子实体，考虑所有的1跳或2条路径p，两种知识图谱上的都要（听起来很暴力）。
- 识别限制（constraints）：
  - 找到实体和类型限制：
  - 所有的答案节点的 is_a 属性（关系）
  - 所有路径上的单跳关系连接的实体
- 过渡到下一个部分询问：
  - 到 composition tree 上决定 $G_{i+1}$ 的起始节点。如果下一个操作是 simQA 那么用接下来会提到的 semanic matching 来匹配 $G_i$ 和原自然语言问题的相似度，找到最好的 K 个。答案为 $G_{i+1}$ 提供种子实体。
  - 不然的话（就是 join 操作？）继续从 $G_i$ 中不重叠的实体关系继续推导。
  - 那这里是说的通的，join 操作要求有实体重叠，也就是关系间有共享的实体，这里就自然地实体重叠了。同样，它没有明说下一步是 simQA 还是 join，应该是一起做的搜索过程。而程序并不关心到底是 simQA 或是 join

####Semanic Matching

接下来本文将描述使用到神经网络模型，用于匹配不同种知识图谱上关系到自然语言问题的相似度。

- 自然语言编码：token 序列和 dependency structure (tree) 
  将词语嵌入表示，依存树同样嵌入表示，拼接
- 主路径编码：
  主路径有两种不同的形式，我们首先将 textual (extracted) relation 对齐到 ontological (curated) relation. 
  - 此部分在 4.2 提及，这里直接先说
  - 学习 textual 的 embedding，并将他们做一个clustering 以规范化
  - 用规范化的 textual 来和 ontological 的关系进行匹配。匹配的标准即为关系重叠了多少个相同的 头尾 节点。
    - 比如 is author of - book.author 关系一定比 is author of - education.institution 重叠的关系多。
  - 避免巧合的匹配，要求设一个阈值为共享关系数量的最小值才能对齐它们为同一个关系。

- 主路径编码：
  对于每种对齐关系，学习每个关系的潜在向量表示（textual）:

  - 考虑所有的 tokens 和 ids （参考例子）。
  - 嵌入，拼接。

  并对所有的向量做 max pooling，作为补充 curated KB (ontological relations)

- 限制编码：

  和上边那个差不多，看图就懂了

- 关注机制：
  因为我们是拿部分问题和整个自然语言相匹配，那么自然语言的所有部分如果权重相同就会对匹配造成干扰，这是我们不希望的。
  这部分我就没太看懂了，但整个过程可以反映对每次将partial query匹配对自然语言问题某一部分的关注程度。
- 目标函数：
  使用文本向量，依存向量 和 询问向量 进入 MLP，评分语义相似度。

####Query Composition

用一系列的小问题来构建整个复杂问题，每个小问题有且仅有一个主 relation path

构造合成树可以通过估计问题中的动词短语数量和他们之间的关系完成 (subordinating / coordinating).

这个有前人的工作完成。

然后对于这棵树，每个level (partial query)，使用 beam search (也就是每次暴力搜索，保存最好的k个的一种贪心算法，但拥有更大的搜索空间) 找到最好的候选者。但如果仅仅使用 semanic matching model 又很单调，于是引入更多的部分来判断当前状态的合理性：

语法相似度，entity linking scores，约束数量，变量节点数量，关系数量和答案实体数量。

