---
title: "LLaDA：大语言扩散模型"
date: 2025-09-02T07:22:50+08:00
categories:
  - 算法
  - 论文
tags:
  - 论文
  - 算法
  - CoT压缩
---

{{< katex >}}

## LLaDA：大语言扩散模型

Large Language Diffusion Models

## 摘要

自回归模型（ARMs）被广泛认为是大型语言模型（LLMs）的基石。我们通过引入 LLaDA，一种在预训练和监督微调（SFT）范式下从头开始训练的扩散模型，对这一观点提出了挑战。LLaDA 通过前向数据掩码过程和反向过程来建模分布，并通过一个原始的 Transformer 架构来预测被掩码的标记。通过优化似然界，它为概率推理提供了一种合理的生成方法。在广泛的基准测试中，LLaDA 表现出强大的可扩展性，优于我们自建的自回归模型基线。值得注意的是，LLaDA
8B 在上下文学习方面与 LLaMA3
8B 等强大的 LLM 具有竞争力，并且在经过 SFT 后，在多轮对话等案例研究中展现出出色的指令遵循能力。此外，LLaDA 解决了反转诅咒，在反转诗歌补全任务中超过了 GPT-4o。我们的研究结果确立了扩散模型作为 ARMs 的一种可行且有前景的替代方案，挑战了上述讨论的关键 LLM 能力本质上与 ARMs 相关的假设。项目页面和代码：[https://ml-gsai.github.io/LLaDA-demo/](https://ml-gsai.github.io/LLaDA-demo/)。

## 动机

- 当前的 ARM 声称具有很强的扩展性，但是这种扩展性实际上是由于数据规模和生成原理所引起的 Fisher 一致性相互作用的结果，不是 ARM 架构所独有的
- 当前 ARM 生成具有强的指令遵循和上下文学习能力，但是这实际上是在结构一致的语言任务中所有合理条件生成模型的固有特性
- 自回归特性带来了较高的计算成本，且从左到右的建模限制了在逆向推理中的有效性，限制了其表现

## 方法

### 概率公式化

LLaDA 的核心是一个 mask
predictor，记这个模型为 \(p\_\theta(·|x_t)\)，以 \(x_t\)为输入，并同时预测所有被 mask 的 token（记为 M），并仅在 M 上计算交叉熵

![](/post_imgs/LLaDA%EF%BC%9A%E5%A4%A7%E8%AF%AD%E8%A8%80%E6%89%A9%E6%95%A3%E6%A8%A1%E5%9E%8B/UePkbEhePob2jexOEs8cATfRnGh.png)

## 实验
