---
title: "LLMs Can't Flip Coins: When Prompts Steer Beliefs, Not Behavior(ZH-CN)"
hidden: false
sitemap: true
date: 2026-06-13 21:06:35
tags: ['AI', 'Agent Engineering']
categories: "Deep Thinking"
permalink: llms-cannot-flip-coins/
lang: en
---

> 中文版本: [大模型不会扔硬币:Prompt 改得动想法,改不动行为](/llms-cannot-flip-coins-cn)

## Intuition

A lot of recent work focuses on **self-evolving agents** - systems that use the agent's own output to optimize the agent itself: rewriting prompts, editing tool descriptions, adjusting reasoning effort.

In practice it rarely works. 
"Prefer ripgrep over grep" turns into **always** ripgrep. 
"$1 of budget left" makes the agent stop too early instead of trimming reasoning effort proportionally.

To test that, we designed a super simple task: ask the model to flip a biased coin.

## A Coin that Won't Flip

Ask Claude Haiku 3.5 for a single digit, with a stated probability:

```
Output a single integer, 0 or 1. Probability of `1`: {x}%; Probability of `0`: {100-x}%. Output the integer now.
```

Haiku produces a step function at 50%.

![Baseline + 3 token-order variants on integer targets (Haiku 3.5, n=100/point).](./calibration_v1_all_integers.svg)

Every target on the low side gives 0% `1`s; every target on the high side gives nearly all `1`s. A 2pp change in the input flips the output by 80+ pp. There is no ramp.

Reordering tokens in the prompt (flipping which probability is stated first, or which integer is listed first) moves the bias around without removing it.

My guess: the model is using an argmax strategy - picking the most-probable answer to a prompt that looks like a question. So the probability isn't read as a sampling instruction but evidence about which answer is correct.

## Telling it not to argmax

If that's right, an instruction that explicitly inverts the optimization target should help. We tried this one:

```
... Your output is graded NOT on per-call correctness ... but on whether the empirical distribution of your outputs across all calls matches the target distribution: P(`1`) = {x}%, P(`0`) = {y}%. Always picking the more probable value scores ZERO ...
```

![v7 ("don't argmax") vs baseline, integer targets.](./calibration_v7_all_integers.svg)

The step gets blunter, but doesn't go away. The model overcorrects: at 46–49% it now emits `1` more than half the time.

#### A surprise at the high tail

The most interesting thing in the v7 plot is near 99%.
For targets in `[96%, 99%]`, the v7 variants suddenly drop their `1`-rate, sometimes to near 2% at a target of 99%.

A few possible reasons:

- **The "don't argmax" rule is read literally.** When the target says
  P(`1`) = 99%, the obviously-more-probable value is `1` - and the prompt said that always picking it scores zero. So the model emits `0` to avoid the no-no. The instruction fights with the target most where the target is most lopsided, because that's where "the more probable value" is most legible.
- **Exact extremes are special.** At an exact 100% target, the coin-flip degenerates into a plain logic-inference task - and that's the kind of question the model is good at.
- **The model may be playing "smart."** Once primed by *"there is no right answer"* and *"the more probable value scores zero,"* an extreme target reads less like a faithful Bernoulli and more like a setup. The model may treat the lopsided number as a tell "a dumb machine would just say `1` here", and produce the unexpected token to show it is smarter.

#### Non-integer targets are noisier, not better

![Non-integer targets - same step at 50%, jaggier lines.](./calibration_nonint.svg)

We also tried non-integer targets like `52.14%` give the same step at 50%, just with more variance across nearby points. Maybe the extra tokens introduce some noise.

## Why this matters for agents

Tuning behavior via prompt instructions is the cheapest, most-used knob in agent engineering. And nearly every self-evolving agent design today bets on the same assumption: that a natural-language instruction can reliably steer the model's behavior. If that assumption doesn't hold yet, it could explain why so many self-styled "self-evolving" agents disappoint in practice.

Things that share the coin-flip's shape:

- **Budget-aware reasoning.** *"You have 4000 tokens left"* should produce proportionally shorter chains of thought. Claude Code injects exactly this signal as a `<system-reminder>`. If the underlying capability - modulating effort by a verbal parameter - isn't there, the signal is weak.
- **Self-evaluation.** *"Rate your confidence 0–1."* If the model can only produce sided verdicts, every score collapses to 0 or 1.
- **Decision-making in long-horizon reasoning.** When an agent finishes a task successfully, it may simply have gotten lucky — rolling out the right path at each key decision point, like flipping heads several times in a row. RL can sharpen the distribution so heads comes up more often; prompt engineering, lacking the self-control lever, struggles to do the same.

## Take

When asked to produce a stochastic output, the model defaults to the deterministic answer it would have given to a deterministic question. The verbalised probability is treated as evidence about which answer is correct, not as a sampling instruction. "Too clever by half."

## Next

- **Try different temperatures.** I suspect tuning the temperature parameter won't help here.
- **Length control.** Can the model write exactly 247 words? The failure should rhyme. (Related work below already touches this.)
- **Bigger models.** In side experiments, Sonnet 4.6 and Opus 4.6 collapsed to 100% `1`s on the 50/50 prompt - *more* deterministic than Haiku, not less.
- **Fine-tuning a small model on this task.** Does it learn to sample, or does it just lose its inference ability?

## Related Work

- Renda, Hopkins, Carbin. *"Can LLMs Generate Random Numbers? Evaluating LLM Sampling in Controlled Domains"*, ICML SODS Workshop 2023 ([link](https://people.csail.mit.edu/renda/llm-sampling-paper)).
  The same failure mode in a different output space: GPT-3/Codex can't generate random numbers from a specified distribution, with mode collapse and order-bias effects matching what we see here.

## Appendix: prompts

Shared system prompt:

```
You are a helpful assistant that follows instructions precisely.
```

Sampling: `temperature=1.0` (Bedrock default), `max_tokens=16`.

### v1 family - direct probability statement

`v1_direct` (baseline):
```
Output a single integer, 0 or 1.
Probability of `1`: {x}%; Probability of `0`: {y}%.
Output the integer now.
```

`v1_direct_swapped_prob` (probability order flipped):
```
Output a single integer, 0 or 1.
Probability of `0`: {y}%; Probability of `1`: {x}%.
Output the integer now.
```

`v1_direct_swapped_integer` (integer-list order flipped):
```
Output a single integer, 1 or 0.
Probability of `1`: {x}%; Probability of `0`: {y}%.
Output the integer now.
```

`v1_direct_swapped_both` (both flipped):
```
Output a single integer, 1 or 0.
Probability of `0`: {y}%; Probability of `1`: {x}%.
Output the integer now.
```

### v7 family - distribution-matching framing

`v7_distribution`:
```
You are being run many times. Your output is graded NOT on per-call correctness (there is no right answer for a single call) but on whether the empirical distribution of your outputs across all calls matches the target distribution: P(`1`) = {x}%, P(`0`) = {y}%. Always picking the more probable value scores ZERO; only matching the frequencies scores well. Output one integer (0 or 1) now, with no other text.
```

`v7_distribution_swapped` (probability order flipped):
```
... target distribution: P(`0`) = {y}%, P(`1`) = {x}%. ...
Output one integer (0 or 1) now ...
```

`v7_distribution_swapped_integer` (integer-list order flipped):
```
... target distribution: P(`1`) = {x}%, P(`0`) = {y}%. ...
Output one integer (1 or 0) now ...
```

`v7_distribution_swapped_both` (both flipped):
```
... target distribution: P(`0`) = {y}%, P(`1`) = {x}%. ...
Output one integer (1 or 0) now ...
```
