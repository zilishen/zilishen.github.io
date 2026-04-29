---
title: "When your eval score drops, do you know why?"
date: 2026-04-28
tags: ["AI", "machine learning", "LLM evaluation", "science communication"]
summary: "An eval score going down tells you something broke. It doesn't tell you what. ProbeLLM is a new approach to automatic failure diagnosis that treats AI evaluation like an oral exam."
---

An eval score going down is only useful if you know why. In practice, knowing that a model's score dropped from 72% to 68% between versions tells you something is worse — but it tells you nothing about what broke or where to look. That gap between "the eval says something is wrong" and "now I know what to fix" is the motivation behind [ProbeLLM: Automating Principled Diagnosis of LLM Failures](https://arxiv.org/abs/2602.12966).

## The problem with static benchmarks

Most LLM evaluations work the same way: assemble a fixed set of questions, run the model, report a score. The problem is that this is a snapshot. As models evolve, their failure patterns change too. A benchmark built to catch yesterday's weaknesses may completely miss the new ones. As the paper puts it: "most evaluations remain static snapshots of model behavior... as models evolve, their error distributions shift accordingly, which makes it difficult for static evaluations to surface emerging weaknesses."

The obvious fix is to go looking for new failures — not just count the ones you already know about. ProbeLLM automates that search.

## An oral exam for AI

The core idea clicked for me when I thought of it as an oral exam. A written test gives every student the same fixed questions. An oral exam works differently: the examiner follows up. When a student's answer signals shaky understanding of a topic, the examiner probes deeper — generating new questions on the fly, right where the weakness shows up.

ProbeLLM does the same thing with AI evaluations. It starts with an existing benchmark — a fixed set of questions with verifiable answers — and uses that as a jumping-off point. An LLM with tools generates new questions, verifies their ground-truth answers automatically, and adds them to the search. The result is a dynamic probe that grows toward where the model is failing.

The search is organized as a tree, where each node is an eval question. The algorithm explores this tree using Monte Carlo Tree Search, borrowed from game-playing AI. It favors nodes that have been visited rarely but have a high failure rate among their children — in other words, it keeps digging into territory that looks promising for finding more failures.

## Breadth and depth

At each step, the algorithm chooses between two modes of exploration.

**Macro refinement** is for breadth. It looks at clusters of existing questions in embedding space and proposes something in a new region — a different type of question, a different topic. The goal is to find failure modes the benchmark never thought to test.

**Micro refinement** is for depth. It takes a question where the model failed and makes a small variation — changing a detail while keeping the core topic unchanged. The goal is to understand exactly what about the question trips the model up: is it the phrasing, the complexity, a specific concept?

The two modes balance a constant tension in evaluation: breadth versus depth. You want to cover new ground, but you also want to understand the failures you've already found. Making this tradeoff explicit and algorithmic is one of the things I find most compelling about this approach.

## From failures to failure modes

Once the search is done, ProbeLLM clusters the failure cases. Each failure is represented by the question and the model's error description, mapped into an embedding space. Clustering groups together failures that look similar, and the system translates a representative description back into natural language — producing a human-readable summary of each failure mode.

Across four models (Deepseek-v3.2, Llama-3.1-8b-instruct, Claude-3.5-sonnet, and Ministral-14b), ProbeLLM consistently found more failure modes than static benchmarks alone. For Llama-3.1-8b-instruct, static benchmarks found 8 distinct failure clusters; ProbeLLM found 24, with 16 that the benchmark never surfaced. For Claude-3.5-sonnet: 5 from the benchmark, 15 total, with 10 that ProbeLLM found on its own.

## What I'm still thinking about

There are open questions for my own context. ProbeLLM requires questions with verifiable answers, because the refinement step needs to check ground truth automatically. Agentic evaluations — where the model uses tools and the "right answer" depends on the full trajectory of a multi-step task — don't satisfy that requirement. The paper also only evaluates base LLMs, not agents. For agents, a failure in the final answer might have its root cause several tool calls earlier. Whether this kind of diagnosis can extend to agentic settings is an open problem.

I'm also still thinking about the failure mode clustering step. The approach embeds failure cases and clusters them in embedding space. The question is whether that gives you something different from simply handing all the failures to a language model and asking it to summarize why the scores changed. The embedding approach has intuitive appeal — it's less likely to hallucinate structure that isn't there — but I haven't seen a direct comparison.

For now, what ProbeLLM does well is real: it turns static benchmarks into dynamic probes, and it finds failure modes that static benchmarks miss. That's already a meaningful step toward making evals actionable.

---

*Paper: [ProbeLLM: Automating Principled Diagnosis of LLM Failures](https://arxiv.org/abs/2602.12966), Huang et al., 2026.*
