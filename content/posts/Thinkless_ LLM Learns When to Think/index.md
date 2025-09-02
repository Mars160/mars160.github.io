---
title: "Thinkless: LLM Learns When to Think"
date: 2025-09-02T07:22:19+08:00
categories:
  - 算法
  - 论文
tags:
  - 论文
  - 算法
  - CoT压缩
---

## Thinkless: LLM Learns When to Think

Thinkless: LLM Learns When to Think

## 摘要

推理语言模型（Reasoning Language Models），具备扩展的链式思维推理能力，在需要复杂逻辑推理的任务上展现了卓越的性能。然而，对所有查询应用详尽推理通常会导致显著的计算效率低下，尤其是在许多问题有简单解决方案的情况下。这引发了一个开放性问题：大型语言模型（LLMs）能否学会何时进行深度思考？为了解答这一问题，我们提出了 Thinkless，这是一种可学习的框架，能够使 LLM 根据任务复杂性和模型自身的能力，自适应地选择短形式或长形式的推理方式。Thinkless 在强化学习范式下进行训练，并使用两种控制 token：<short> 用于简洁的回答，<think> 用于详细的推理过程。我们方法的核心是一种解耦组相对策略优化（Decoupled Group Relative Policy Optimization, DeGRPO）算法，该算法将混合推理的学习目标分解为两个部分：(1) 控制 token 损失，用于指导推理模式的选择；(2) 回答损失，用于提高生成答案的准确性。这种解耦的公式化设计使得可以精细地控制每个目标的贡献，从而稳定训练过程，并有效避免了在标准 GRPO 中观察到的崩溃现象。实验结果表明，在 Minerva Algebra、MATH-500 和 GSM8K 等多个基准数据集上，Thinkless 能够将长链思维的使用减少 50% 至 90%，大幅提升了推理语言模型的效率。代码可在 [https://github.com/VainF/Thinkless](https://github.com/VainF/Thinkless) 获取。

## Motivation

- Long cot 效率低
- 已有的研究的模型推理与否受制于人类的先验知识，可能陷入次优，让模型自己决定才是正解

## 方法

### 影响推理的因素

- 用户问题的难度：形如 1+1=？的问题不需要推理
- 模型的能力：即使有些问题较难，但是如果模型能力上去了也不用过多推理
- 用户对于效率和准确性的容忍度

### 具体方法：

#### 用于预热的蒸馏

目的：构建一个能生成 short 形式和 long 形式的模型

步骤：取一个 reasoning 模型和一个 instruct 模型，分别对于同一个问题进行回复，并在其回复前加上<think>或<short>标签，构建以下数据集

![](/post_imgs/Thinkless__LLM_Learns_When_to_Think/NrzUbstZKouzUPx9nGxcNshvn0e.png)

并使用该数据集 SFT，

{{< lead >}}
为什么不把两个回复拆开呢？
或者是说训练的时候其实是分开训练的

让模型遇到<think>就开始 reasoning，遇到<short>就开始简单回答
{{< /lead >}}

#### 通过解耦的 GRPO 学习何时思考

##### 优化策略

![](/post_imgs/Thinkless__LLM_Learns_When_to_Think/VhePbHtguoh6hrxQ8v4cb0O6nQc.png)

其中 c 要么是<think>要么是<short>，代表进入推理模式或直接回答

##### 奖励设计

![](/post_imgs/Thinkless__LLM_Learns_When_to_Think/TRsTbzcDMovGXRxkJmMcwCHBnic.png)

简言之，鼓励<short>且 ✅，稍微鼓励<think>且 ✅，❌ 了扣大分

##### 解耦策略优化

原 GRPO

![](/post_imgs/Thinkless__LLM_Learns_When_to_Think/RVtjboFGqorCXJx8504clBN8nge.png)

但是（1）模式-准确性不平衡 - 每条轨迹仅包含一个控制 token，而有 Ti 个响应 token，这不成比例地削弱了模式选择相对于响应准确性优化的影响。（2）长思-短思不平衡 - 更长的序列由于归一化因子 1/(Ti + 1)的存在进一步抑制了控制 token 的梯度贡献，导致<think> token 相比<short>被优化得不足。

所以设计了新的策略

![](/post_imgs/Thinkless__LLM_Learns_When_to_Think/QIdUbGkAWoxPvTxGQIKciOYhnEf.png)

## 实验

模型：DeepSeek-R1-Distill-Qwen-1.5B

数据集：AIME、Minerva Algebra、MATH-500 和 GSM-8K

使用 DeepSeek-R1-671B 和 Qwen2.5-Math-1.5B-Instruct 在 DeepScaleR 上分别生成 think 和 short

硬件：4 张 H100

超参：

生成数据集 max_len: 16k（长了截断），SFT 时调整到 24k

SFT 训了一轮，RL 训 600 步

![](/post_imgs/Thinkless__LLM_Learns_When_to_Think/Bq1UbJNiGoQEzAxO958cUYq2nVc.png)

在 AIME 上大胜！
