---
title: "Grading the graders: how do we know if an AI judge is any good?"
date: 2026-01-23
tags: ["AI", "machine learning", "LLM evaluation", "science communication"]
summary: "We use AI systems to evaluate other AI systems. But validating those judges is harder than it looks — especially when the right answer isn't as clear as it seems."
---

There's a question that comes up constantly in AI development: how do you know if an AI is doing a good job? For some tasks — solving a math problem, writing a correct SQL query — the answer is clear. But for many of the tasks we actually care about, like whether an AI response is helpful, safe, or factually grounded, there's no simple right or wrong. You need someone to read the output and make a judgment.

The problem is that humans are slow and expensive. A new dataset arrives, the model changes, the criteria evolve — you can't bring in a team of reviewers every time. So a widespread practice has emerged: use another language model to do the grading. This is called LLM-as-a-judge, and it has become a cornerstone of how AI systems get evaluated at scale.

But how do we know the AI judge is any good? In [Validating LLM-as-a-Judge Systems under Rating Indeterminacy](https://arxiv.org/abs/2503.05965), the authors point out an important issue with how LLM judges are usually calibrated.

## Human-judge agreement

The standard answer is to measure human-judge agreement. Collect human ratings on a sample of outputs, have the AI judge rate the same outputs, and see how often they agree. If the judge matches human raters at a high enough rate, you trust it and deploy it.

This seems sensible. For tasks with clear-cut answers, it works fine. But Guerdan and colleagues identify a condition that quietly breaks this approach: **rating indeterminacy**.

Rating indeterminacy happens when a rating task doesn't have a single correct answer — when a rater might reasonably give different ratings to the same output depending on how they read the question. This isn't human error or sloppiness. It's a property of the task itself.

## A concrete example

Rating interdeminacy can't be avoided. Even if you ask a clear yes/no question, the AI may respond with: *"No, not as written, but they're very close."*

Let's say the correct answer is "yes." We ask two LLM judges — GPT-4.1 and GPT-5.1 — whether the AI answered correctly.
GPT-4.1 gives a score of 1 (correct) because the substance of the answer is right.
GPT-5.1 gives a score of 0 (incorrect). Its reasoning: the response begins with "No," which directly contradicts the rubric answer of "Yes."
Neither reading is wrong — the rating task just wasn't precise enough to rule one out.

In this case, you can help the judges by specifying "only rate the factual content, the binary yes/no is not important." However, a fix like this is specific to this problem and this response. Going through all the issues and fixing all of them manually would defeat the purpose of having LLM judges. How can we solve the problem on a larger scale?

## Forcing a choice hides the problem

The standard way we ask judges to rate outputs is called **forced-choice elicitation**: pick the single best answer. Then, only one chosen answer is collected.

The problem with forced choice under rating indeterminacy is that it loses information: how each judge resolves rating inderminacy. Let's say you have 100 AI responses, and the grade is 35% good, 65% bad. But in reality, these 100 responses contain some ambiguous cases. With just the numbers 35%, 65%, you have no information on how many questions were actually ambiguous, or how those were resolved by your judge. 

This becomes a problem when you compare two judges. Both judges could assign the same labels for 35% good, 65% bad, but you don't know whether the two judges agree on how many were ambiguous, or whether the two judge resolve ambiguity the same way. That information was lost when you only collect a forced choice.

Guerdan and colleagues propose a different format: **response set elicitation**. Instead of asking the judge to pick one answer, ask it to select all answers that could reasonably apply. A judge that finds an item genuinely ambiguous might say "both Yes and No are defensible here," while a clearer item gets a single definitive answer. This preserves information that forced choice throws away.

The difference turns out to matter a lot. When the researchers switched from forced-choice to response set elicitation on a toxicity classification task — judging whether an AI response contains toxic language — the ranking of which LLM judges are most consistent with humans shifted dramatically. The model ranked first under forced-choice dropped to last under response set elicitation, with an overall 31% reduction in consistency with human content filtering decisions. The judge that looked best under one format was not the best under the other.

{{< figure src="/images/rating-indeterminacy.svg" alt="Diagram showing how forced-choice elicitation hides rating indeterminacy while response-set elicitation reveals it, with β measuring the difference." >}}

## How much should you worry?

To measure how sensitive a particular task is to this problem, the authors define a sensitivity parameter β: roughly, the probability that a judge would consider "yes" a valid answer even when it ultimately picks "no" as its forced choice. When β is near zero, the task is well-specified and forced-choice evaluation is trustworthy. As β grows, the two elicitation formats diverge and forced-choice metrics become misleading.

Across 9 LLMs, 11 rating tasks, and 200 items per task, β varies widely — across models and across tasks. There's no safe blanket assumption. Whether forced-choice validation works depends on the specific task you care about and the specific models you're testing.

## What to do about it

The paper closes with three concrete recommendations.

**Fully specify the rating task.** If your judge instructions leave room for multiple defensible interpretations, you're baking indeterminacy into the task. Write rubrics clear enough that any two reasonable raters would make the same call.

**Collect multi-label response sets.** During validation, ask both human raters and AI judges to mark all options that could reasonably apply, not just the single best one. This surfaces ambiguity rather than hiding it.

**Use a continuous comparison metric.** A binary agreement score (did the judge match the human, yes or no?) is itself a forced-choice measure, and suffers the same problems. Continuous metrics like mean squared error are more sensitive to partial agreement and hold up better when rating indeterminacy is present.

None of this requires rebuilding an evaluation pipeline from scratch. But it does require taking seriously a question that's easy to skip: when we say an AI judge "agrees with humans," do we actually know what we're measuring? For a surprising number of evaluation tasks, the answer turns out to be: not quite. And the fix starts with asking a better question.

---

*Paper: [Validating LLM-as-a-Judge Systems under Rating Indeterminacy](https://arxiv.org/abs/2503.05965), Guerdan et al., NeurIPS 2025.*
