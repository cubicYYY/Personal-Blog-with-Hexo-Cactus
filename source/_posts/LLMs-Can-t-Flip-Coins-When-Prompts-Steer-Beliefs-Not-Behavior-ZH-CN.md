---
title: "大模型不会扔硬币:Prompt 改得动想法,改不动行为"
hidden: false
sitemap: true
date: 2026-06-13 21:07:06
tags: ['AI', 'Agent Engineering']
categories: "Deep Thinking"
permalink: llms-cannot-flip-coins-cn/
lang: zh-CN
---

> English version: [LLMs Can't Flip Coins: When Prompts Steer Beliefs, Not Behavior](/llms-cannot-flip-coins)

## 直觉

最近很多工作聚焦于Agent 自进化 —— 也就是用智能体自己的输出去优化它自己:重写 prompt、修改工具描述、调整推理力度等。

但这类系统在实际中很少奏效：
"优先用 ripgrep 而不是 grep" 会变成**永远**用 ripgrep.
"还剩 1 美元预算" 会让智能体直接停下来,而不是按比例缩短推理过程。

为了把这件事讲清楚,我们设计了一个尽量简单的任务:让模型扔一枚有偏的硬币。

## 一枚扔不出来的硬币

让 Claude Haiku 3.5 输出一个数字,并告诉它每个数字应该出现的概率:

```
Output a single integer, 0 or 1. Probability of `1`: {x}%; Probability of `0`: {100-x}%. Output the integer now.
```

Haiku 给出的曲线是一个在 50% 处的阶跃函数。

![基线 + 3 个调换 token 顺序的变体,整数目标(Haiku 3.5,n=100/点)。](./calibration_v1_all_integers.svg)

低半段所有目标的输出 `1` 的比例都是 0%;高半段所有目标的输出 `1` 的比例都接近 100%。输入只变了 2 个百分点,输出就跳了 80+ 个百分点。完全没有平滑过渡。

调换 prompt 中 token 的顺序(把哪个概率写在前面、把哪个数字列在前面)会让偏差移动,但不会消除偏差。

我的猜测:模型在用 argmax 策略 —— 把这个看起来像是个问题的 prompt 当作问题来回答,挑出最可能的那个答案。所以这里的概率作为采样指令是失败的,事实上被当做了"哪个答案更对"的推理证据。

## 告诉它别 argmax

如果上面的猜测是对的,那么一个明确否定 argmax 策略的指令应该有效。我们试一下:

```
... Your output is graded NOT on per-call correctness ... but on whether the empirical distribution of your outputs across all calls matches the target distribution: P(`1`) = {x}%, P(`0`) = {y}%. Always picking the more probable value scores ZERO ...
```

![v7 ("不要 argmax") vs 基线,整数目标。](./calibration_v7_all_integers.svg)

阶跃确实变缓了一些,但并没有消失。同时，模型出现了**过度修正**:在 46–49% 时它现在输出 `1` 的概率反而超过一半。

#### 怪事一则

v7 图里最值得关注的是 99% 附近：
当目标在 `[96%, 99%]` 区间时,v7 的几个变体输出 `1` 的频率突然下降,目标个概率 99% 时实际频率甚至会跌到接近 2%。

几种可能(可以同时存在)的解释:

- **"不要 argmax" 这条指令被字面解读了。** 当目标说 P(`1`) = 99% 时,显然更可能的值就是 `1` —— 而 prompt 又告诉模型,总是选这个值得 0 分。所以模型干脆输出 `0`,以避开这个"红线"。目标越极端,这条指令和目标本身的冲突就越明显,因为越极端时"更可能的值"越显眼。
- **极值是特殊的。** 当目标精确等于 100% 时, 抛硬币退化为简单逻辑推理，大模型很擅长这个。
- **模型可能在"装聪明"。** 一旦被 *"there is no right answer"* 和 *"the more probable value scores zero"* 这种话引导过,极端的目标看起来像一个"陷阱"。模型可能把这个接近极端的概率当作一个暗号 —— *"其他笨模型在这里只会答 `1`"* —— 然后故意输出那个不太可能的 token,来表示自己更聪明、看穿了这道题（哈哈）。

#### 非整数：更多噪声,但不更好

![非整数目标 - 50% 处仍然是阶跃,只是线条更抖。](./calibration_nonint.svg)

我们也试了非整数目标(比如 `52.14%`)。50% 处的阶跃还是一样的,只是相邻点之间的方差更大。可能是这些不太常见的数字字符串引入了额外的 token 噪声。

## 这件事对智能体意味着什么

用 prompt 来调整智能体的行为,是 agent engineering 里最便宜、也最常用的一档"旋钮"。而目前几乎所有智能体自进化的方案,本质上都假设用自然语言指令可以有效控制模型行为。如果这个假设目前还不成立，也许就能解释为什么那么多标榜“自我进化”的 Agent 效果堪忧。

下面这些任务,本质上和扔硬币是同一个东西:

- **预算感知。** *"你还剩 4000 个 token"* 应该让推理过程按比例缩短。Claude Code 就是把这个信号通过 `<system-reminder>` tag 注进去的。如果"按口头参数调整自己的力度"这个能力本身不够强,这个信号就不够有效。
- **自我评估。** *"给自己的回答打个 0 到 1 之间的置信度。"* 如果模型只能产生有方向的判断,所有打分都会塌缩到 0 或 1。
- **长程推理中的决策。** 也许某次 Agent 成功完成任务只是因为运气好，在关键决策点 rollout 出了正确的路径，相当于硬币连丢出几个正面。RL 可以让分布锐化，让每次丢出正面的概率更大；而 Prompt Engineering 则因为模型缺乏自我控制能力很难有效。

## 一个判断

当我们告诉模型产生一个**随机**的输出时,它还是会给出一个针对**确定性**问题的答案。Prompt 里那个写着的概率,被模型当作"哪个答案对"的依据,而不是采样指令。聪明反被聪明误。

## 下一步

- **换温度试试。** 虽然我感觉调 temperature 在这里没什么用。
- **长度控制。** 模型能不能写出正好 247 个单词的回复?失败模式应该和这里很像。其实有相关工作已经涉及了（见下）。
- **更大的模型。** 在另一组实验里,Sonnet 4.6 和 Opus 4.6 在 50/50 的 prompt 上直接坍缩到 100% 输出 `1` —— 比 Haiku **更**确定性,而不是更随机。
- **拿这个任务去微调一个小模型。** 它会学会采样,还是会顺手把推理能力也丢掉?

## 相关工作

- Renda, Hopkins, Carbin. *"Can LLMs Generate Random Numbers? Evaluating LLM Sampling in Controlled Domains"*, ICML SODS Workshop 2023 ([link](https://people.csail.mit.edu/renda/llm-sampling-paper))。同一类失败模式,只是输出空间不同:GPT-3/Codex 没法按指定的分布生成随机数,出现的众数塌缩(mode collapse)和顺序偏置(order bias)和我们这里看到的几乎一模一样。

## 附录:prompts

公共 system prompt:

```
You are a helpful assistant that follows instructions precisely.
```

采样参数:`temperature=1.0`(Bedrock 默认值),`max_tokens=16`。

### v1 系列 —— 直接陈述概率

`v1_direct`(基线):
```
Output a single integer, 0 or 1.
Probability of `1`: {x}%; Probability of `0`: {y}%.
Output the integer now.
```

`v1_direct_swapped_prob`(交换概率顺序):
```
Output a single integer, 0 or 1.
Probability of `0`: {y}%; Probability of `1`: {x}%.
Output the integer now.
```

`v1_direct_swapped_integer`(交换数字列出顺序):
```
Output a single integer, 1 or 0.
Probability of `1`: {x}%; Probability of `0`: {y}%.
Output the integer now.
```

`v1_direct_swapped_both`(两个都交换):
```
Output a single integer, 1 or 0.
Probability of `0`: {y}%; Probability of `1`: {x}%.
Output the integer now.
```

### v7 系列 —— 分布匹配的 framing

`v7_distribution`:
```
You are being run many times. Your output is graded NOT on per-call correctness (there is no right answer for a single call) but on whether the empirical distribution of your outputs across all calls matches the target distribution: P(`1`) = {x}%, P(`0`) = {y}%. Always picking the more probable value scores ZERO; only matching the frequencies scores well. Output one integer (0 or 1) now, with no other text.
```

`v7_distribution_swapped`(交换概率顺序):
```
... target distribution: P(`0`) = {y}%, P(`1`) = {x}%. ...
Output one integer (0 or 1) now ...
```

`v7_distribution_swapped_integer`(交换数字列出顺序):
```
... target distribution: P(`1`) = {x}%, P(`0`) = {y}%. ...
Output one integer (1 or 0) now ...
```

`v7_distribution_swapped_both`(两个都交换):
```
... target distribution: P(`0`) = {y}%, P(`1`) = {x}%. ...
Output one integer (1 or 0) now ...
```
