---
date: 2025-02-17T16:49:10-05:00
lastmod: 2025-02-17
showTableOfContents: false
tags: ['machine learning', 'prompt engineering', 'DeepSeek', 'LLMs']
title: 'DeepSeek and Explicit Learning'
type: 'post'
---

# Two Types of Learning

There is a marked difference between how **young children** learn and how you learn later in life. Kids are able to soak up experiences and data, find the underlying patterns, and understand them **implicitly**. Adults and even older children learn through **explicit rules and examples** with a more structured but less intuitive understanding of new concepts.

The clearest example of this is language learning: my native language has a complex case system (English no longer has cases, but just imagine 'he vs him vs his' for every single word and with more versions). Although I spoke it fluently on a daily basis, I didn't notice the case system and would not have been able to articulate it. That only changed when I formally learned about cases and declensions in my Latin classes and realized that the same patterns appeared in my native tongue.

I'll refer to the two modes as implicit and explicit learning, respectively, going forward. Asking which of these two modes of learning is better is the _wrong question_. Rather they represent a **tradeoff**: explicit learning is more efficient, but it's only possible after a certain level of ability gained through implicit learning. Implicit learning has a higher ceiling, and explicit learning doesn't build the same level or type of intuition. Finally, it is easier to transfer or teach explicit learnings.

The most important observation is that humans require both. Existing machine learning techniques are in the implicit mode: they require large amounts of data and compute and learn from pure association without any explicit teaching.

In the machine learning community, attempts at explicitly instilling principles of thought/reasoning into models are largely frowned upon. The conventional wisdom is that training should be about **"unlocking compute"** i.e. providing the setup to a. best utilize available compute and b. ensure that more compute means more performance i.e. performance gains should not plateau at a certain level of compute (or at least a level of attainable compute). _The Bitter Lesson_ by the legendary Richard Sutton is the canonical example of this argument.

I think that DeepSeek's approach might help open up a new lane of performance boosts that operate in the explicit learning paradigm **WHILE** still unlocking compute.

# DeepSeek

By Deepseek's approach, I'm referring to two - maybe three - ideas used in the R1 reasoning model.

The first is using **reinforcement learning** as a method to "unlock compute". R1 improves by trying to solve math and coding problems many times with its performance improving over time. It learns by positively reinforcing successful attempts and negatively reinforcing unsuccessful ones.

The second is measuring performance through **automated testing** and **in relation** to previous attempts. During its reasoning training, R1 only solves problems with clear right and wrong answers so that it can be graded automatically. Moreover, it only considers how well it's doing relative to previous attempts — not in absolute terms. Intuitively, suppose the model scored 71% on attempt number 11. If it averaged 70% on the first 10 attempts, then this go-around isn't much different, and the model won't change much. However, if the average up till now was 50%, attempt 11 is a significant improvement, and the model will reinforce whatever it did in attempt 11. The opposite is true if the model was averaging 85% on the last 10 attempts.

The third is relying on **humans** to teach **"taste"**. At several points in the training process, R1 utilizes extensive human curation of data. For example, the first iteration of R1 (called R1-Zero) produced many outputs that got to the right answer but weren't readable e.g. it switched languages part way through a response. Annotators went through these initial "pure RL" outputs and selected useful examples. In the first stage of R1's training, the model is re-initialized and learns from only the selected examples. R1 has several more stages of training, some of which also involve human curation.

I think these first two ideas are definitely applicable. The third one is more up in the air.

# LLMs

First, a bit of context: large language models (LLMs) literally just take in some text and predict the next token (for our purposes, token = word). They can still be very useful because all sorts of different problems can be structured as next token prediction tasks. For example, I could have an LLM solve $$65 \div 13$$ by passing in the input, "65 divided by 13 is equal to" and having the LLM predict the next token. LLMs _are_ very good at next token prediction which means they're very sensitive to small changes in the context. In particular, models can be **"prompted"** to do useful things by adding certain details to your input. For example, instead of simply asking an LLM to answer a question, you could mention that the response should be written in rhyming iambic pentameter. The seminal example of a useful prompt is "Let's think step by step" (Kojima et al., 2022). Simply adding this one sentence made models much more likely to correctly solve math problems. The model is, as always, just trying to guess the next token, but given the context of "Let's think step by step", it is more likely to carefully reason through a problem (step by step), which in turn leads to the correct final answer more often. You can never be certain how or why a prompt works because these models are still black boxes, and there is no way to ensure that a certain behavior is actually happening under the hood.

# The Idea

Ok, finally, here's the idea: we enable explicit learning through automated prompt engineering.

-   First, you pass some context to a model e.g. some info about the types of problems that you're trying to solve, and you ask the model to output a prompt that encourages useful behavior for solving these types of problems. The model will give you a reasonable suggestion: prompting is a well known technique in LLMs, and any model released since say GPT-4 should have information about prompting in its training data.
-   Use this prompt to attempt to a solve a suite of tasks/problems. The tasks/problems must have some automatable notion of completeness/correctness. DeepSeek did this for math/coding problems, but it should, in theory, be more widely applicable. For example, if the task involves sending an email to a certain address, you could check if the email was received.
-   Grade the prompt based on how it did across the whole suite of tasks/problems
-   Use the relative performance (i.e. how did this prompt do compared to other prompts for the same task/problem set) to encourage prompts with better performance.
-   Repeat steps 1-4

After reaching a suitable level of performance, use the winning prompt to do the specific class of tasks/problems.

# The Upside

Why do I think this is so great:

-   A key problem in reinforcement learning (and in machine learning in general) is ensuring that the model is able to extrapolate and learn something **broadly applicable** instead of learning minute details or tricks specific to the training data. Think about the difference between becoming an expert in a field and memorizing the answers to a single test. By iterating on the overall prompt - rather than the model's behavior on individual problems - we ensure that the learned behavior is a. **utilized across the board** and b. **general**. The prompt can be kinda, maybe, sort of thought of as an _algorithm_. I say "kinda, maybe, sort of" because there's nothing requiring the model to actually execute the prompt and especially execute it as intended: it is simply trying to predict the most likely next token. With sufficiently sophisticated language understanding, that should mean that it executes the algorithm ("like a human would"), but there are no guarantees: LLMs are fundamentally stochastic and fickle.
-   This training paradigm switches over from the implicit learning regime to **explicit learning**. Again, this likely has a lower ceiling than if you were able to push implicit learning to its full extent, but it should be more **efficient** (and in many cases, data constraints mean that you can't hit the ceiling anyway). It's also much easier to **transfer** the findings (the prompt) from one learner (i.e. one LLM) to another than it would be transfer any performance gains that require an actual update to the model's weights: you literally just copy and paste the prompt — though there will definitely still be variation from LLM to LLM even with the same prompt. And like we noted in the discussion about implicit vs explicit learning in humans, the base models will need to have a certain level of performance from standard (implicit) training to be able to utilize prompts: a model that doesn't understand basic math will not be able to solve AIME problems no matter how good the prompt is. Most importantly, humans can actually **interpret** prompts and understand what the model is doing (or at least what we're nicely asking the model to do). On the other hand, it is (currently) impossible to interpret changes in model weights, and by extension, it's impossible to understand performance gains from implicit learning.
-   You're tuning the model for specific tasks, but the changes are extremely lightweight, and your tuning strategy is scalable because evaluation is automated. Thus you can actually **utilize more compute**.
-   Adding new capabilities would only require a new task/problem suite for that field or role. This could enable the resulting models to be far **more specialized** than the broad math/coding specialization of models like R1.
-   LLMs (or at least generative LLMs) are causal. This means that the model only considers past tokens while interpreting the current token. For example, in the sentence "This is a blue day.", the model's context for the word blue would only be "This is a", and it would not see the word "day". This means if you have two queries with the same preamble, the model's computations are identical up until the divergence. This in turn means that the model can cache (i.e. save and reuse) its computation on the identical first section. For example, if the model is processing the sentences "What is the weather today at 4pm?" and "What is the weather tomorrow at 6pm?", it would only have to process the divergent component ("tomorrow at 6pm?") for the second sentence. This means that our prompt can be **quite large and contain quite a lot of info without a ton of cost**. Longer prompts will definitely still require more compute, but a. the cache will help quite a bit regardless, and b. there's a world where the longer, more powerful prompt means that the model will require less exploration and lengthy deliberation to get to the right answer (this is currently the case with reasoning models). If this tradeoff exists and we can move length from the output to the prompt, it would be a significant cost reduction relative to existing reasoning models because the general prompt can be cached while the specific output (the model's deliberation on a particular problem) cannot.

## The Upside: Human in the Loop

The final reason is based on the third DeepSeek insight: human taste/intervention. This step is perhaps the most tricky but also the closest to existing products that are built on top of LLMs. A nimble way to incorporate human feedback is:

-   Run the prompt + model on a suite of problems
-   Have a knowledgeable human peruse the model's failures and make a suggestion
-   Pass the feedback to the LLM and ask it to incorporate the tips into a prompt
-   Test the resulting prompts in the normal prompt iteration pipeline

I think this particular component has huge potential. It allows **humans to teach without being prescriptive or rigid** i.e. without the usual concerns about imparting human knowledge to models. The prompt generated from feedback will be treated no different than prompts generated entirely by the model: it will **only be used** if it produces a useful change that drives better performance. This step could also be automated with LLMs processing failure cases and generating feedback for themselves, but I think the real upside is enabling humans to guide the model. After all, ideas like best practices are fundamentally concepts that are **distilled from collective human experience** and passed down explicitly rather than discovered by an individual experimenter. I also think this is a completely **reasonable workflow**: you would generally expect to lead new co-workers through some level of onboarding and on the job training before they really takeoff. This would be the equivalent startup process for an LLM.

I will hopefully be working on this idea in some capacity in the near future, but this approach requires different people to create different training setups for their own use case, and I'd love for others to try implementing it.

#### References

Sutton, Richard. "The bitter lesson." Incomplete Ideas (blog) 13.1 (2019): 38.

Guo, Daya, et al. "Deepseek-r1: Incentivizing reasoning capability in llms via reinforcement learning." arXiv preprint arXiv:2501.12948 (2025).

Kojima, Takeshi, et al. "Large language models are zero-shot reasoners." Advances in neural information processing systems 35 (2022): 22199-22213.
