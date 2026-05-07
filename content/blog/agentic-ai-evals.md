---
title: "Agentic AI evals: lessons from real life"
date: 2026-05-07
tags: ["AI", "machine learning", "LLM evaluation"]
summary: "AI products can change under your feet. Here's what I learned about measuring whether they do what you think they should."
---

If you've worked on an AI product, you've probably felt this: fix one thing and something else breaks. Add a new feature and behavior you thought was settled starts acting differently. Swap the base model, and the product feels like a different product.

There's a reason for this. Most agentic AI products are mostly scaffolding, but the core logic is driven by an LLM that is stochastic and outside your control. Traditional code is self-documenting: reading it tells you what it will do when you run it. An LLM call in your code does not do that. Any change in the prompt can have unexpected effects on the outcome, and you won't find out until you run the product.

Unit tests and integration tests tell you whether the scaffolding works. They say nothing about whether the AI does what you want it to do. For that, you need evals. For the past eight months, building them has been my full-time job at a startup. Here's what I learned.

## What do people mean by "evals"?

When people say "evals," they can mean very different things. Some people want an A/B test comparing two prompt versions. Some want a benchmark against a competitor. Others want a regression suite that catches performance drops or production issues. Or they could want a gate on feature development to know when to call it done.

I think evals can serve all of those goals if you use the right approach for each one. But there's no magic eval that accomplishes everything simultaneously. The key realization is that every measurement costs something. Experiments have to be designed carefully if you want to detect a real signal. These are well-known principles in any empirical science. Measuring AI is no different.

## Where things go wrong in real life, and how to fix them

**Evals are expensive.** Traditional tests take seconds and cost nothing. Evals cost money. A thorough internal benchmark can cost hundreds of dollars per run, which changes how often you can afford to use them.

This also means your eval set needs its own quality control, something you would never think about for unit tests. Give benchmark tasks a lifecycle. Review them regularly and retire tasks when they stop providing signal. A task the model now reliably gets right is only useful for regression checking. A task with ambiguous ground truth is worse: it's adding noise every time you run it.

To manage this, I built an internal tool where everyone could see and interact with our eval tasks. Each task had a status (Draft, Active, Flagged, Archived) and anyone at the company could raise issues with a task directly. Out of everything we tried, this had the biggest impact on eval quality.

**Unverified LLM judges.** Using an LLM to grade your AI's outputs is common, but a common pitfall is to use the judge score without checking whether the judge is doing a good job. Issues are easy to miss without a systematic check. The judge prompt might be vague. Or it might ask the LLM judge to do something they are not designed for, like five numeric comparisons with percentage tolerances, custom if-then logic to arrive at a final score, or assume domain-specific knowledge that generic LLMs do not have. You get a number back, but you don't know if it is meaningful.

This becomes visible when you switch judges. Switching to a different LLM as the judge changed our scores by more than 10%, a bigger shift than anything we saw from changing the product itself. Which judge was right? To find out, I created human-annotated datasets: I graded AI responses myself to produce ground truth labels, then measured human-judge agreement for each LLM judge and picked the one that matched best. That gave us a reliable baseline, and also surfaced problems with the judge prompts that we were able to fix.

**Don't train on your evals.** The obvious version: don't use your eval set as training data. The more subtle version is a manual loop where someone tweaks the prompt, reruns evals, and repeats until the score goes up. That loop is prompt-tuning on your eval set. This is easy to fall into without realizing it. After enough iterations, the system prompt can grow to include specific references to eval questions and instructions for how to handle them. At that point, you're no longer measuring the product, you're measuring familiarity with your benchmark.

The fix is the same one used in any kind of model training: hold out a portion of your eval set that you never touch during development.

**LLMs are stochastic.** Your AI agent won't give you the same answer twice. If you run each eval task only once, you are only capturing one outcome out of many possible ones. Always run with repetition. How many? Enough that the variance in your estimate is smaller than the effect you're trying to detect. You can always add more repetitions.

## What to actually aim for

The pitfalls above are all about doing evals correctly. But before worrying about execution, it helps to have the right design principles: the things you won't compromise on. I think of these as axioms: like in math, they're foundational rules you accept upfront and build everything else on top of.

**Axiom 1: Evals should produce actionable insights.**

There's a more fundamental question to ask when you design an eval: what will someone do with this result? At a research lab, a good eval might expose a weakness in the underlying model and stop there. At a company, that's not enough. You can't fix the model. You can fix your prompts, your architecture, or change to another model. Evals should produce something a person on your team can act on.

This rules out a lot of common choices. Most eval platforms ship with pre-built judges for accuracy, helpfulness, or conciseness. These are easy to run and easy to report, but they rarely tell you what to fix. If helpfulness drops by 3%, what does that mean? A percentage score tells you something is wrong. Failure analysis tells you what and why. In order to do that, there needs to be a process to dig into the root cause of failures found in evals. Which type of user question is the agent not handling? Evals should always facilitate root cause analysis so that it can be actionable.

**Axiom 2: Eval is an empirical science.**

All the principles of empirical science apply here. Always look at your data. Experiment design matters: if you're comparing two versions of a product, you need to control for everything else or you won't know what caused the difference. And error bars matter. A score without a measure of variance is incomplete. It doesn't tell you how much of what you're seeing is noise, which means you can't tell what is signal.

Looking back at the pitfalls in the previous section, they all trace back to this. Running each eval task only once means ignoring variance. Using an unverified LLM judge means trusting a measurement instrument without validating it. Training on your eval set is the train/test split problem. None of these are AI-specific. They're the same issues that come up in any empirical science, and the solutions are the same too.

---

*A lot of what I described here resonates with Hamel Husain's [The Revenge of the Data Scientist](https://hamel.dev/blog/posts/revenge/), which makes the case that eval work is fundamentally a data science problem. Worth reading if you want to go deeper.*
