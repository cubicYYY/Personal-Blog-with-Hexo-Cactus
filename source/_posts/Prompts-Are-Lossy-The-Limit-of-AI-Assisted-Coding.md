---
title: 'Prompts Are Lossy: The Limit of AI-Assisted Coding'
tags:
  - LLM
  - AI
  - Software Engineering
  - Thoughts
  - Opinion
permalink: prompts-are-lossy/
hidden: false
sitemap: true
lang: en
date: 2026-06-06 23:27:56
categories:
---


> 中文版：[Prompt 是一种有损压缩：AI 辅助编程的边界](/prompts-are-lossy-zh/)

## Intro

A few years ago, many software engineers believed they were irreplaceable - LLMs, the argument went, could never *think* like a real human. Today, many of those same people believe software engineering is dead, because LLMs have grown strong enough to handle almost any coding task.

Both camps share a hidden assumption: that software engineering is coding. It isn't. Coding is the visible output; the actual work is deciding what to build, for whom, under which constraints, and how the answer should change as those constraints shift.

That distinction matters because the role does not disappear when the typing gets cheap — only when the *deciding* gets cheap. And the deciding is hard for a reason that has nothing to do with how clever the model is: every prompt is a compression of something larger that lived inside your head, and compression is lossy by definition. The better the model, the more that lossiness becomes the bottleneck.

## No Free Lunch Theorem

In machine learning, the *No Free Lunch* theorem says no single algorithm beats every other algorithm averaged over all possible problems — performance is always relative to a distribution. Software has its own version: there is no "best" app or website, only the best one for a particular scenario, and scenarios are often dynamic, fast-paced, and human-centered. The only lossless description of a product is the product itself; everything else is a sketch.

This is why "just describe what you want" rarely works beyond the trivial. Describing forces you to project a high-dimensional desire — taste, edge cases, business context, the history of why earlier attempts failed — onto a low-dimensional string of tokens. The LLM unprojects it back into code, but fills the missing dimensions with its priors, not yours.

LLMs offer an extremely good prior over what good code *looks like*, but that is the easier half of the problem. The harder half is defining the problem itself.

## Inevitable Info Loss

If we could describe exactly what we need, we would already have finished the app - and that, of course, is not what we want.

In my view, LLMs offer a wide menu of "options," each with a recommended "default value." Before the LLM era, we called these "best practices." This is why building a toy project with an AI assistant feels effortless: the defaults happen to match what we want.

But when the need is new or vague, the defaults are rarely the *right* values. And, (un)fortunately, every real need is a new need - unless you are reinventing the wheel. That is why we keep seeing vibe coders ship calendar apps, todo lists, and fitness trackers: these are not essentially new needs, and the LLM already knows what they are supposed to look like. It is tweaking, not inventing.

There is a deeper reason a model cannot pick the right default for you: it does not have the facts. If all relevant information truly flowed freely and could be processed instantly, the stock market would be a flat line — every fact already priced in. It isn't, because not all facts are public, and the ones that are public keep changing. The knowledge that drives software decisions behaves the same way: what your users actually did this quarter, why the previous architect picked option B over A, which competitor just changed pricing, what the on-call engineer learned at 3am last night. None of that is in any training set, and most of it will be different next month. An LLM trained on yesterday's *open* knowledge cannot make today's *private* decision for you.

## A Lesson From Half a Century Ago

The dream of an automatic code-generating machine is not new. Fred Brooks's *The Mythical Man-Month* — the foundational work of software engineering, written fifty years ago — already mapped out the limits of such a machine.

### Essential Complexity vs. Accidental Complexity

- Accidental complexity is the friction of *how* — boilerplate, plumbing, syntax, repetitive glue. LLMs eat through that effortlessly.
- Essential complexity is the friction of *what* — the irreducible difficulty of the problem itself, the ambiguities that demand resolution, the implicit and undiscovered requirements. Nothing about generating tokens makes that friction smaller.

### The Mythical Man-Month

Brooks's titular point is that man-hours and calendar time are not interchangeable: adding people to a late project makes it later, because communication overhead grows faster than headcount. An army of agents is no different. The bottleneck has never been hands or the size of agent team — it is the small number of people who actually understand what is being built and why, and that group does not get cheaper to coordinate when you add more workers to it, biological or otherwise.

## There Is No Real Zero-Human-in-the-Loop: The Sim2Real Problem

You can take the human out of development. You cannot take the human out of *requirement-finding*, because applications are ultimately used by people. Fully automating software engineering would require an agent that knows how humans react to the app — in advance, in every variation that matters. That is not a coding problem; it is a simulation problem.

Robotics calls a closely related problem *sim2real*: a policy trained in simulation rarely survives contact with the real world, because no simulator captures every quirk of friction, sensor noise, and physical edge case. Software has its own version - the gap between any model's internal picture of the user and how that user actually behaves. The difference is that "the world" here is not physics; it is taste, attention, mood, and a thousand other things even the user cannot articulate. Without a simulator that captures all of it, the only sensor we have is a real human noticing the wrong thing happen.

Fortunately for us, autoregressive drift seems unavoidable: a model imitating its training distribution compounds small errors into large divergences from what real users actually do, and any self-evolving system trained primarily on its own outputs inherits the same flaw. So the loop still needs *someone* to translate observation into change. The language does not matter — Rust, C++, Python, or prompts — only that a human stays in the loop where it touches reality.

## Rethinking Software Engineering

Even before the LLM era, the typical career path was familiar: spend a few years as a programmer, then move into product management. Why? Because the longer you ship code, the more you realize the bottleneck has never been your fingers. It is the gap between what someone wanted, what they asked for, and what eventually got built. PMs exist because that gap exists.

LLMs do not close the gap; they only shift where it shows up. The work of bridging intent and implementation - the work that pushed senior engineers toward PM in the first place - has only become more important. The titles will change, but the role will not.

## Why Cybersecurity?

You may have noticed that the major labs are pouring resources into making models better at cybersecurity. The reason is structural: in many security scenarios, the target is clear, the codebase is fixed, there is almost no human interaction, and you can know with certainty when the system is compromised. There is no room to argue about whether the agent succeeded.

That means oracles are easy to design. Anyone working in machine learning knows how much a clean oracle matters - especially under RL. Cybersecurity is one of the few domains where the feedback signal is unambiguous enough to bootstrap real capability gains. It is, in a sense, the *opposite* of building products for humans.

## Final Words

This is not meant to reassure software engineers. Many tasks do not require human interaction, and many users only want a "good enough" answer; LLMs serve those needs. Besides, are the things LLMs cannot do automatically things a software engineer *can* do? And are the things a software engineer can do automatically beyond a product manager's reach? SDE roles will definitely shrink massively and rapidly.

The narrower claim is this: there is still a place where humans add irreducible value — at the boundary where vague intent becomes concrete software — and you have to find your own way to live there, while the machines keep getting better at everything else.
