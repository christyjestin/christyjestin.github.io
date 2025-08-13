---
date: 2025-08-12T22:10:10-04:00
lastmod: 2025-08-12
showTableOfContents: false
tags: ['continual learning', 'machine learning', 'benchmarks']
title: 'Continual Learning Benchmark Proposal'
type: 'post'
---

I'm gonna try to make a benchmark for **continual learning**. I have a ton of uncertainty about how this should be done. This is an initial proposal laying out what I'm thinking right now, and I'd love any and all **[feedback](https://forms.gle/77RaaHtYJgeswcK6A)**. Skip ahead to the **[actual proposal](#a-modest-proposal)** here. Alright first,

# TF is Continual Learning?

Currently, most state of the art (SOTA) neural networks (NNs) like large language models (LLMs) are trained in-house and **static once deployed**. During training, their parameters or weights are iteratively updated to improve performance on various tasks. Then these weights are frozen when the model is deployed i.e. the model no longer updates in response to new data. Now, training is become increasingly richer and varied with multiple phases that have different strategies. For example, you might pretrain with self-supervised learning (SSL) on next-token prediction and post-train on reinforcement learning with verifiable rewards (RLVR), but ultimately training is done entirely in house before the model's released.

In contrast, a continual learning system would _continue_ to learn after deployment: it would respond to new data by updating itself to **improve performance in a persistent fashion**. Humans (and animals) are extremely capable because of our ability to continually learn: we are able to pick up new skills and knowledge, and we can do this "on the job" without always requiring dedicated training phase. This includes everything from a boxer making clever adjustments mid fight to someone using context clues to understand a new slang word.

# Why Now?

Continual learning has had a resurgence in the AI zeitgeist thanks to my favorite AI influencer: Dwarkesh Patel. Dwarkesh — who interviews many leading researchers on his podcast — says that in his view, [continual learning](https://www.dwarkesh.com/p/timelines-june-2025) is the big hurdle to artificial general intelligence (AGI). He came to this conclusion based on his own experience trying to work with models like Gemini. He notes that the big difference between working with Gemini and working with an actual human co-worker is that a human co-worker can learn on the job. They're able to **adapt to feedback** and learn best practices or style or "the way we do things around here". In short, a human co-worker is **much more capable 6 months into the job than they were on day 1**. On the other hand, LLMs are very capable on day 1, but they do not improve further over time. Now Dwarkesh is certainly not the first or only person to have this experience or to make this complaint, and I think there is general interest in figuring out how to get past this hurdle.

# Doesn't [X] solve this?

There's a few common counters (i.e. potential solutions) to this problem.

## Long Context

The first is long context: instead of updating weights in response to feedback, you could just add it to a model's context window so that the model's future outputs take the feedback into consideration. This already works today — to some extent — but the real problem with long context is **attention itself**. Attention is the primary mechanism behind LLMs, and as Dwarkesh notes, attention has quadratic computational complexity. This means that as sequences get longer, each additional output requires more and more compute.

Additionally, at an intuitive level, long context is like trying to **consider a million things at once**. We observe that this is empirically a problem in models (the phenomenon is known as "context rot"), and it's also easy to see how this can happen by just looking at the math behind attention. Another conundrum is that both training and evaluation are very expensive with long context. In practice, models are initially trained with a shorter, fixed context. Then the context window is gradually extended till we get the extreme two million token context windows that are available today. Part of the struggle here is the dirth of high-quality long context training data where the model actually needs to utilize long context.

Now, I should say, it is definitely the case that the machine learning (ML) community has only scratched the surface on long context, and we could definitely be using it in much better and smarter ways. It is still likely the case that long context has fundamental limits — just based on how the math works.

## Recommender Systems

We do have an example of cutting edge continual learning systems: recommendation engines. These models are updated frequently with new data, and they are trained **incrementally** i.e. these models are not retrained from scratch — or at least, not often. While these models receive frequent updates, the updates utilize **large amounts of data** since these recommendation engines are deployed at massive scale (think YouTube or TikTok). Even in the cases where they do not have a ton of new data on a specific user, they are still able to predict how that user will respond to new content. This is because all of the recorded interactions are fit into a single framework explaining the relationship between users and content.

As far as I understand, the scale of data is critical to ensuring that recommender systems are able to meaningfully learn from new data without also going way off the rails and forgetting all of their existing knowledge. I can't find the tweet now, but I did see a researcher making the case that this is what continual learning is really about: **sample efficiency**. Can models learn from a small amount of data/feedback without the massive scale of recommender systems?

I should say that it might be more accurate to describe recommendation engines as "online learning" systems, but online and continual learning are certainly related.

## Reinforcement Finetuning (lots and lots of it)

A third counter to the continual learning problem is reinforcement finetuning: basically anytime the model gets some feedback, it should spin up an a new RL environment and use a ton of synthetic data and data augmentations to drill itself **until the feedback no longer applies**. For example, if you complained to an LLM about its use of em dashes, it could have itself generate text on various different subjects until it learns to stop using the dash in its writing. The output of this process would be a new, updated model that preserves most of the original model's behavior while addressing the user's feedback.

I would say that given what I know about reinforcement learning, this would be a very difficult process to operationalize — especially in a general way. Data augmentations would be a critical component, and you'd need to achieve incredible diversity. That said, while it would be undoubtedly difficult to pull off, reinforcement finetuning would be a valid solution to continual learning, and it very well could be **_the solution_**. The procedure we discussed is pretty similar to the methods that do well on the ARC AGI benchmark. That benchmark is meant to measure fluid intelligence or skill acquisition — a concept that is definitely related to continual learning but not quite the same. In my opinion, they emphasize very different aspects of learning (elaborate composition vs delicate adaptation), but I don't want to go off on a tangent here.

By the way, I want to note that the same tradeoff (from recommendation engines) rears its head again for reinforcement finetuning: **can a model learn something new and useful while preserving most if not all of its pre-existing knowledge/ability?** This question has always been central to all forms of reinforcement learning, and it's still important in cutting edge techniques like RLVR that apply RL to LLMs.

# A Modest Proposal

Here is what I have in mind: consider a model \(f\) parametrized by \(\theta\). It is given an input \(x\) and produces an output \(y=f(x;\theta)\). A grader function produces natural language feedback \(e(x,y)\) and a numerical reward \(r(x,y\)). The goal of a continual learning system \(CL\) is to produce a better parametrization \(\theta' = CL(x, y, \theta, e(x, y), r(x,y))\) s.t. \(r(x,y') \geq r(x,y)\) where \(y'=f(x;\theta')\).

Note that \(\theta\) is an all encompassing parametrization that goes beyond just the weights: it can include model components like the system prompt or even architectural details. We make \(\theta\) as broad as possible so that the benchmark is **agnostic to how the problem gets solved**. The notation \(e\) is adopted from the error signal \(e\) in the feedback control literature.

The benchmark will primarily measure four axes:

1. The change in performance on the benchmark suite: \[\sum_{i}\sum_{x\in\mathcal{X}_i}{r\big(x, f(x;\theta^{final}_{i})\big) - r\big(x, f(x;\theta)\big)}\] where each \(\mathcal{X}_i\) is a set of related tasks that should be solved using the same parametrization \(\theta_i\)
2. The efficiency of the \(CL\) system i.e. the gain in performance relative to the number of interactions or "environment steps"
3. The change in the model's general capabilities
4. The change in the model's efficiency

# Why This Metric

Criteria 1 and 2 are straightforward and standard for environment-based evaluation suites. Criteria 3 and 4 are more specific to the problem of continual learning. Criteria 3 is about whether the \(CL\) system **preserves the base model's capabilities** i.e. if it was able to do X before, is it still capable of the same? If there has been a regression in performance, how severe is the regression?

Criteria 4 is about **overhead**: you wouldn't want the model to use 10x as much compute or time to gain the new ability, and even if you are fine with some increase in compute or time, you'd at least want to understand the **tradeoff between performance and efficiency**. This is somewhat of a philosophical point, but it does intuitively fit with continual learning in humans. Your colleague who is more proficient after 6 months on the job does not require more time to do the task — in fact, they should be more efficient. You wouldn't be happy if they were only able to do the tasks now because they spend hours scouring through the C++ standard or say, waiting for the LLM to do it for them. The same is true for models. A reasonable solution could use some extra chain of thought — or it could append to the context window — but you wouldn't want a pure brute force solution, like one that just abuses the 2 million token context window. Or maybe you're fine with that, but again, you'd at least want to understand that efficiency tradeoff.

I initially considered having hard cutoffs for losses in efficiency and generalization (e.g. at least 80% of the original efficiency), but I think this is the wrong approach. It's more useful to see the tradeoff (or the **Pareto frontier**) since different use cases will have different requirements, and even if I wanted to make a hard requirement, there isn't a singular universal metric anyway.

On that note, you probably can't use a single, fixed benchmark to test how much of the model's generality is preserved. For example, if the CL task requires the model to adapt to a new dialect of Javascript, you probably don't have to worry as much about preserving the model's creative writing capacity because this is an orthogonal ability. On the other hand, you might want to ensure that the model is still capable of writing Typescript since this is a closer skill that may be in conflict. In order to capture this, you'd want a benchmark that is **"in-domain" or related to the continual learning task**. For example, you could use SWE-Bench-477 to check if the model can still write simple Django.

There is one final ability that I'd like to test: how well can you apply the new knowledge in a **cross-domain setting**. For example, if you learn to translate between a small artificial language and English, can you now also translate between the language and German? Can you write poetry in the language? In my opinion, LLMs are surprisingly strong at this type of cross-domain repurposing/interpolation, so I am a little less worried, but this is a very impressive ability and one that is tough to measure.

# Why Language Feedback

There's a few reasons why I think it's important to use language feedback — as opposed to just the numerical reward or even categorical or multi-class errors:

1. Language feedback is **richer** and **denser**: both factors have been very important in RL.
2. It's also more **intuitive** for human researchers: we are pretty good at coming up with and understanding natural language feedback.
3. Finally, it's just more **native** and **versatile**. For example, if the task allows/involves coding or tool use, then error messages, request statuses, and generic console output can all be included in the natural language feedback without any modification. Something like LLM-as-a-Judge or the vaunted "universal verifier" could also slot in easily here.

# What's Next

Here's how I'm planning to construct the benchmark:

1. Crowdsource possible ideas for evals — I'm especially curious what people wish models were able to learn based on their own day-to-day experience of using LLMs
2. Setup pipelines to test existing models and ensure that naive ICL isn't enough to trivially solve them
3. Choose sufficiently hard problems and use naive ICL as a baseline for the benchmark
4. Split the suite into train, validation, and held out private test sets

I'd absolutely love any and all feedback **[here](https://forms.gle/77RaaHtYJgeswcK6A)** whether it's tasks you'd like to see or advice on making a good benchmark or something else.
