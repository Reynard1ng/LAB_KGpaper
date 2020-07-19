## 在知识图谱上的多跳复杂问题询问图生成

[TOC]

文章提到，过去的工作对于具有约束的问题和多跳关系的问题处理，通常是分开完成的。本篇文章将同时处理这两类问题。

本文提出了一种分阶段的查询图生成方法，旨在修改阶段性问题图的生成方法，用于允许更长的关系路径（多跳）。

同样地，不同于以往的在生成关系路径之后再添加约束的方法（这看上去比较僵硬），本文提供的方法将添加方案与扩展关系路径同步进行以降低搜索空间。因为随着路径的延长，搜索空间的增长是指数级的。

本文的方法在三个数据集 ComplexWebQuestions，WebQuestionsSP，ComplexQuestions 着不错的表现。



### 关于两种问题

- 单关系问题，但有约束，比如 “谁是第一个美国总统”，“第一个”就是约束。
- 多跳关系问题，比如 “谁是脸书的创立者的妻子”。



### 又到了讲概念的时间

- KB 三元组（不多说了）
- Query Graph 问题图，四种节点，一种边
  - grounded entity 阴影矩形，是已知的 KB 图中实体
  - existential variable 无阴影矩形，ungrounded entity
  - lambda variable 圆，ungrounded entity 但是是答案
  - aggregation function 菱形，一个函数，自变量为一个集合的实体，类似于 argmin 取最小值，count 计数等
  - path 路径，代表关系
  - 问题图要求有且仅有一个答案节点，至少一个已知实体节点，和任意的 existential variable 和 aggregation function 节点。
  - 可以参考 figure1 进行理解
- 提到一个核心关系路径，大概可以理解为一个多跳的主线任务
  - 在第二页左下角 1）中提到已有的工作认为该主线任务包含一个单个的关系。
- 前人的工作
  - 从问题中找一个实体，探索一个单条或者两跳（CVT）关系作为主线任务
  - 向主线任务添加约束
  - 通常会用一个 CNN 判断这个任务和问题的相似度
  - 把最好的输出来。



### 核心科技

#### 整体思想

简单多跳就相当于在图中找到从这个实体，走K段路能到达的所有点，这无法接受

有个办法 top-K t-hop (Chen et al,. 2019; Lan et al., 2019b)，大概可以理解为我们会在前 t 跳进行评价，保留最好的K个就行了。但是这忽略了约束。

文章提到，在多跳问题的扩展中，加入约束是非常有效的，就比如说文章中提到的问题：

- Who is the first wife of TV producer that was nominated for The Jeff Probst Show

主线任务完成后加约束大概是这样的：

- Who is the first wife of TV producer.
- was nominated for The Jeff Probst Show 加给 TV producer

边拓展边加约束是这样的：

- TV producer
- was nominated for The Jeff Probst Show 加给 TV producer (搜索空间显著降低)
- the first wife of TV producer (再做主线任务)

那在加上之前的 top-K t-hop，岂不美哉？



#### 问题图生成

它在处理的每一步，有三种备选项：

- extend:
  - 增加一跳，就是增长主线任务链
  - 当然可以是经过 CVT node 的两跳
  - 当前图中只有一个 entity，那么就是它拓展所有关系
  - 如果图中有 lambda variable (答案)，那么从它拓展所有关系，并变成extential variable
  - 连出去的点是新的答案
- connect:
  - 除了主话题实体以外，有其他的 grounded entities 将会用到这个操作
  - 这可以将这些实体连接到当前的 lambda variable 或者 连接到 lambda variable 的 CVT node
  - 执行当前的问题图，找到当前答案的所有实体与你当前在处理的那个实体之间的所有关系
- aggregate:
  - 检测到聚合函数（aggregate function）（Luo et al. 2018）的时候，将聚合函数附着至当前答案或者它相邻的CVT node 上。

这么搞下来每次仍然是指数级的数据量增长，那么我们通过接下来提到的评分系统，留下 top-K 即可。

那最后肯定是留下最好的作为答案。



#### 问题图评分

一个 7 维度的评分系统，以评判 $p(g|Q)$，g是当前的问题图，Q是自然语言问题。

第一维为 Bert-Based 匹配模型。将 g 中除了 existential variables 和 lambda variables 的其他节点与边按顺序将其中文本转化为有序序列，可参考 2.4 例子。做文本匹配。

其他六维一句话说过去了，没有做非常详细的解释。

（说实话2.4第一段我没有太看懂，如果有误请不吝赐教）

不过我觉得这应该是最麻烦且非常核心的一块地方，不知道它为什么没有细说。我们可能需要花很大一部分时间来研究如何对图进行评分。



### I have a question

也许是我没看清楚，看到 3.1 的第一段末尾，aggregate functions 只有 argmax 和 argmin 嘛，那似乎并没有处理顺次某位等情况。

不过本文提到了可以处理 ComplexQuestions 数据集，就很奇怪，可能会是我看漏了，或者 ComplexQuestions 数据集中此类问题并不多。



### 评估与总结

文章提到，相当多的问题在于对问题图的错误评估。更直观地说，这些问题主要出在某些询问中的关系对于人来说都很难辨识。

还有一些问题在于实体和表达式的链接错误，也就是没有正确给到约束。

少量错误在于难以找到合适的问题图，比如问题：

- What jobs did John Adams have before he was president.

总的来说本文提出了一个较为新颖的方法来同时解决多跳问题与多约束问题。打破常规在完成主线任务添加约束的思路，将二者一同进行以减小搜索规模，并在三个复杂数据集上取得了不错的效果。



