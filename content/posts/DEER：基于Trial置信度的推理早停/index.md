---
title: "DEER：基于Trial置信度的推理早停"
date: 2025-09-02T07:22:12+08:00
categories:
  - 算法
  - 论文
tags:
  - 论文
  - 算法
  - CoT压缩
---

{{< katex >}}

## DEER：基于 Trial 置信度的推理早停

DYNAMIC EARLY EXIT IN REASONING MODELS

## 摘要

近期大型推理语言模型（LRLMs）的进展依赖于测试时扩展，这将长链条思维（CoT）生成延伸以解决复杂任务。然而，过长的 CoT 不仅会降低问题解决的效率，还可能因过于详细或冗余的推理步骤而导致准确性损失。我们提出了一种简单而有效的方法，允许大型语言模型（LLMs）在生成过程中通过**早期退出机制自我截断 CoT 序列**。与依赖固定启发式规则不同，所提出的方法**在潜在的推理转换点（例如，“Wait” tokens）监测模型行为，并在模型对试探性答案表现出高置信度时动态终止下一推理链的生成**。我们的方法**无需额外训练**，可无缝集成到现有的类似 o1 推理 LLMs 中。在多个推理基准测试 MATH-500、AMC 2023、GPQA Diamond 和 AIME 2024 上的实验表明，所提出的方法在 deepseek 系列推理 LLMs 上始终有效，平均将 CoT 序列长度减少 31% 至 43%，同时将准确性提高 1.7% 至 5.7%。

## Motivation

1. 过长 CoT 显著增加计算开销、推理延迟
2. 过长 CoT 可能偏离正确道路导致 Acc 下降
3. 推理信息中具有信息刚刚好足够的关键点（Pearl Reasoning）

### 验证 Motivation 3

{{< lead >}}
文中似乎对于“Wait”等 token 有一个假设，就是当这个 token 出现时，代表着 llm 要换推理路径了
{{< /lead >}}

选择 AIME2024 数据集，让 DeepSeek-R1-Distill-Qwen-14B 进行完整推理和解答，然后从 think 过程中找出“Wait”这个 token，基于这个 token 将完整的推理划分为思考片段，然后仅保留 len(思考片段) > 5 的样本。对于这些样本保留了其不同比例（20%-90%）的思考片段，并在每个截断的推理序列末尾添加了一个结束思考的 token 分隔符，以强制终止慢思考过程。结果如下图

{{< lead >}}
结束思考的 token 分隔符应该是</think>这种
{{< /lead >}}

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/G2kzbCIt6oNgTcxAYcLcv3PanDd.png)

LLM 对于大多数问题（75%）的推理具备这样一个 Pearl Reasoning，甚至部分（36.7%）问题的 pearl reasoning 还不到原问题的一半。

{{< lead >}}
值得注意的是，有一些 Question 在推理较短的时候正确，但是长了反而错了
{{< /lead >}}

此外，还验证了一下原始推理是正确/错误的但是使用早停机制后正确的数量（Threshold 为 1.0 代表原始推理）

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/PORZbw9vioomfUxNJNmcZoTdnNn.png)

## Method

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/X6FSbpcB4oCbymxyrm3cwik3nAh.png)

识别出推理中信息刚刚好充足的关键点（Pearl Reasoning），并迫使模型在这一步停止思考转为开始回答，提出三步走方法来识别关键点：

- 推理转换监控

像“Wait”这样的词识别为推理转换的关键点，并对其出现进行监测。出现以后进入下一步

- 试验性答案诱导

将最后的“Wait”替换为“Final Answer”，来诱导 model 尝试给出答案

{{< lead >}}
替换为 final answer 后，模型在推理过程中尝试给出答案，并在最后加上</think>
{{< /lead >}}

- 置信度评估

计算试答的置信度，如果试答的置信度够高，就让 LLM 基于已生成的想法直接给出结论；否则就撤回替换那一步，让 LLM 继续推理

$$
A=LRLM(P,T,I)
$$

P 代表输入的 Prompt，T 是目前已经生成的 Thought，I 代表引导回答的 token，A 代表着 final answer token 后的东西，由很多 token 组成的 token 序列：\(A=[a_1,a_2,a_3,……a_n]\)

通过下面的公式计算 A 的置信度，下图中 M 为 LM Head 及其前置组件，以 logits 作为输出。

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/GPvMbFCwpovTZ1xbMmIcSfwZnFc.png)

如果置信度 c 大于阈值 \(\lambda\)（超参），则认为到了 Pearl Reasoning。

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/Y6Ztb8AmYoQI8mxFKaLc97L9nEf.png)

### 小优化

当推理出现 Wait 的时候，不是停止继续推理，而是通过 Attention Mask 进行**继续往下推理**和**试着回答**的并行，并通过基于置信度的动态 KV 缓存管理进行剪枝。

{{< lead >}}
**不知道这个“基于置信度的动态 KV 缓存管理进行剪枝”是指什么**
是说试着回答之后将尝试回答的部分剪掉，然后重新开始推理？
{{< /lead >}}

## 实验

数据集：

- 数学：MATH-500、AMC 2023、AIME 2024、GPQA Diamond
- 编程：HumanEval、BigCodeBench

模型：DeepSeek-R1-Distill-Qwen 系列 1.5B、7B、14B、32B，额外测了 QwQ-32B

超参：max_len=16384

{{< lead >}}
没提到 lambda 的设置，但是后面测试了一系列 lambda
{{< /lead >}}

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/T8U5bbMovocff9xoZbNci9Z2nIc.png)

{{< lead >}}
关于 LEN，原文提到测量的是 the number of generated words，应该是包括了推理和最终答案的所有内容
{{< /lead >}}

### 其他现象

1. DEER 纠正的错误比 DEER 造成的错误多（绿色部分多于红色部分）

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/MgY4b7KPloMBh4xbE3gc7EJUnne.png)

1. 如果 DEER 能在红色样本上避免过早退出，在在 AMC23 上 7B 模型能打败 14B 模型，因此可以研究如何修改早停策略
2. 一系列 \(\lambda\)和编程数据集

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/SaqAb4UybocXNExLRKRc51JLnye.png)

1. 监控其他的推理分界线

DEER（W）是 Wait，DEER（A）是 Alternatively

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/A8mhbYUYro1trNxXunockhNmnsd.png)

1. 在 QwQ-32B 上的实验

![](/post_imgs/DEER%EF%BC%9A%E5%9F%BA%E4%BA%8ETrial%E7%BD%AE%E4%BF%A1%E5%BA%A6%E7%9A%84%E6%8E%A8%E7%90%86%E6%97%A9%E5%81%9C/T71WbsS43o7WLfx9Gc4cfmFwnsc.png)
