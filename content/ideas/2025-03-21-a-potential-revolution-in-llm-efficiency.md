---
date: 2025-03-21T01:23:13-04:00
lastmod: 2025-03-21
showTableOfContents: false
tags: ['machine learning', 'LLMs', 'attention', 'mixture of experts', 'efficiency', 'sparse models', 'interpretability']
title: 'A Potential Revolution in LLM Efficiency'
type: 'post'
---

# Intro

To me, there's always been two criticisms of large language models that stood out among the rest:

-   LLMs use 100% of their capacity on every user query regardless of whether the query is about single digit addition or the anatomy of tardigrades.
-   LLMs don't have a way of making simple, intuitive updates to store new information.

> It'll seem like I forgot about this second point, but don't worry: I will get back to it eventually.

**Mixture of Experts (MoE)** — the approach that fuels DeepSeek and GPT 4 (the latter is technically a rumor, but it's an open secret in the AI industry) — is meant to fix the first issue. Instead of running the entire model on all inputs, the MoE architecture **breaks up the model** into several experts and passes each input through some fraction of these experts. For example, in the case of DeepSeek V3, each input is passed through 8 experts out of a total 256 experts as well as a single shared expert that's used for all inputs. This is a huge leap forward in **efficiency**, but I believe more gains are possible.

Here's why: thought requires both different **skills** and different **knowledge**. Coding and painting require different skills. On the other hand, while coding in Python and Java require the same (or similar) skills, they require different knowledge. The distinction here is between a problem solving method (skills) and the exact details or information required to apply that method (knowledge). If you're implementing a Python program, you'd need to understand general methods for coding (e.g. how to break down a problem into clear sequential steps) as well as the specifics of the Python language (e.g. which keywords and types are available in the Python language).

The current training paradigm of mixture of experts has no mechanism to distinguish between the two. The model's parameters (a.k.a. weights) are used to process inputs and generate outputs, and everything the model learns **(both skills and knowledge)** is stored in these weights. In my mind, these two components can be **separated functionally**: skills are more akin to **functions** i.e. they parse inputs to produce outputs while knowledge is **additional context** that is stored (in your memory) and retrieved to help perform tasks. For example, the mental model for programming in Python would be something like

```python
python_program = programming_skill(task, python_knowledge)
```

where the additional context of `python_knowledge` could be swapped out with `java_knowledge` to produce a Java program for the same task.

As it exists, the mixture of experts model is more akin to

```python
python_program = programming_skill(task_description_mentioning_python)
```

There is no architectural separation between skills and knowledge since all weights are directly used to run the forward pass (i.e. the function named `programming_skill` in our mental model) so the knowledge must be **baked into the function** (alongside knowledge about other programming languages). The relevant knowledge is then simply elicited whenever something in the input suggests that it's relevant.

Another possible representation for the mixture of experts architecture is the **combination of skill and knowledge** i.e.

```python
python_program = python_programming_skill(task)
```

But I think this is unlikely with the current number of experts since there just aren't enough experts to do this: instead I think current MoE models are somewhere in between the second and third forms, leaning much more towards the second form. Also note that this third form is not desirable — both because replicating the skill is a less efficient representation and because abstraction is generally seen as a good property in information systems i.e. common, abstracted representations are better representations.

# On Efficiency

I'm about to propose a small architectural change that will split out skills and knowledge, but first let's talk about why that's even desirable. My claim is this: if on an average task, you use a fraction \(n\) of all skills and a fraction \(m\) of all knowledge, then \(m\) is smaller than \(n\) and smaller by **several orders of magnitude**. The simplest argument for this claim is to say that there is a **many to one** mapping between knowledge and skills i.e. for each skill, there are several sets of knowledge that correspond to that skill. Our programming example illustrates this: there's many programming languages that all map to the same core programming skill, and you'd only use one or (very occasionally) two of those languages at a time. However, I think programming is actually a **relatively conservative example** with a lower than usual knowledge to skill ratio. If you consider an analogous skill for thinking about history, then there are far more sets of distinct historical knowledge than there are programming languages. I know that the concept of a "set of knowledge" is vague and fuzzy, but my point is that regardless how granular your sets are, the ratio between the total size of history knowledge and the total size of programming knowledge is the same. I don't think the "many to one" argument is perfect because knowledge and skills don't line up so neatly i.e. it's probably not the case that every piece of knowledge nicely maps to just one skill, but I think the model is largely correct and very good intuition.

I'm just using some nice round numbers to **illustrate the implication**: say for example, that on a given task, you use \(20\%\) of your skills and \(0.2\%\) of your knowledge. If your model does not distinguish between the two, AND all knowledge is baked into the skills, then you'd very simply use **\(\bold{20\%}\) of total capacity**. Suppose instead that your total capacity is split 50/50 between separate representations of skills and knowledge. Then you'd use \(20\% \text{\enspace of\enspace } 50\% = 10\%\) through skills and \(0.2\% \text{\enspace of\enspace } 50\% = 0.1\%\) through knowledge for a total utilization of \(10.1\%\) . However, I think that this 50/50 split is unrealistic, and that even a 20/80 skills/knowledge split would be generous to skills. Under this variation, you'd instead use \(20\% \text{\enspace of\enspace } 20\% = 4\%\) through skills and \(0.2\% \text{\enspace of\enspace } 80\% = 0.16\%\) through knowledge for a **total utilization of \(\bold{4.16\%}\)**. Again this is just an example: I completely made these numbers up, but I think they're definitely **directionally correct**.

Ok so great, it would be superb if we could split up skills and knowledge. Why do I even think this is possible? The **key lies in the mental model of skills being functions and knowledge being context**. In order for weights to capture knowledge, they **should not directly operate** on the model's inputs. Instead they should be retrieved when relevant and used as context for the skills to generate output. I know what you're thinking: that sounds like RAG (or retrieval augmented generation). What I have in mind **differs from RAG** in a few key ways that I will expand upon later. First here's my proposal:

> We take the mixture of experts architecture and keep the experts to implement the various skills. Then we add a massive \(n \times d\) vector as a **knowledge store** where \(n\) is the number of knowledge embeddings and \(d\) is the dimension of each embedding. When processing inputs, we choose some fraction of the experts and some fraction of the knowledge embeddings. The experts process the inputs like normal while the retrieved knowledge embeddings are used for **cross attention**. To recap, when an LLM processes text, new words are processed with the earlier input as context: this is called self attention since both the new input and earlier input are of the same form. Cross attention is the same idea with a slight difference: the context and the new input are of **different forms**. This allows us to use the knowledge embeddings as context to process text input despite the knowledge embeddings and text being different forms.

In mixture of experts, inputs are routed to various experts through a **learned gating function**: the \(k\) experts with the greatest gating values are chosen, and the gating values are reused (as weights) to take a weighted average of the outputs of the different experts. Attention natively has a similar weighting mechanism so instead of learning a separate gating function, we can **simply compute the attention weights** between the input and each one of the knowledge embeddings, then retrieve only the \(k\) knowledge embeddings with the greatest attention weights. These top \(k\) embeddings would then contain the \(k\) most relevant pieces of knowledge. We'd setup the embeddings such that computing this attention weight is extremely light relative to actually using the chosen embeddings since we'd want to the do the former over the entire knowledge store but only do the latter over a tiny subset. N.B. please do not confuse _attention weights_ with the _model's weights_: a model's weights are its parameters and are independent of input while attention weights are for an input-dependent, relative weighting of the context.

As with mixture of experts, each embedding will only be trained on inputs for which it is actually used. This reinforces the **specialization of embeddings** just like it does for experts. Additionally, in line with DeepSeek's MoE implementation, we can tinker with the attention weights to ensure that the embeddings are load balanced i.e. none of them are being used too often or too rarely.

### Why This Isn't RAG

Ok, is it a retrieval augmented generation scheme? Yes, absolutely. Is it close to existing RAG implementations? No, I think they're extremely different.

First, let's look at **how existing RAG implementations work**. LLMs are trained without any retrieval based schemes i.e. they learn to simply predict the next token given the previous tokens (for our purposes, token = word). Then when you deploy the model, you equip it with some sort of **external database**. For example, if you're using RAG to answer questions about biology, you might use a biology textbook. You'd first break up the source material (e.g. into paragraphs) and preprocess each chunk into a latent embedding that represents that chunk. A latent embedding is just a vector (i.e. a fixed length list) of decimal numbers, and you would create these embeddings with a separately pretrained encoder model e.g. a sentence transformer. Then when the base model is asked a question, it would first search the preprocessed database for chunks that might be relevant. Next, it'd retrieve the original text for each one of the most relevant chunks. Finally, the model would use that text as additional context to help generate output and answer the user's query. Thus the model's text _generation_ is _augmented_ through _retrieval_.

Ok, why do I think this is different from my proposal? RAG, as it exists, is a **postprocessing step to enhance models**. It is not all that successful in practice because two of the core tasks are very difficult: it is hard to efficiently represent chunks of text, and it is hard to come up with the right query to pull actually relevant info from the database. The whole setup is a **mix** of human interpretable data (text) and uninterpretable latent data (the latent embeddings), and the embeddings act as a **bottleneck** since you have to squeeze a whole lot of information (from the chunk of text) into a single vector. Granted, the third core task in the RAG pipeline — generating text in context — is perfect for LLMs and is pretty close to what the model does during training.

The big difference with my proposal for a knowledge store is that it's **native retrieval augmented generation**. From the jump, the model is trained to encode information as latent embeddings in a way that will be helpful _to itself_ for generation. There is **no mixing** of human interpretable and latent embeddings: **everything is of the model, for the model, and by the model**. Additionally, cross attention means that we can use whatever dimensions we want to store more or less information. Please ignore this sentence if you don't know how the attention mechanism works: for example, you could use small key vectors with large value vectors to keep retrieval efficient while still storing lots of information.

# The Bigger Picture = More Benefits

### Efficiency in Deployment

Beyond the inherent efficiency gains with this decoupled architecture, you can go even further after training. For example, you might be more conservative during training — pulling in more knowledge embeddings as context to ensure you have enough information to answer queries. But as the embeddings and the model's structure become more refined, we can slowly reduce the number of embeddings pulled in without much risk. Fewer embeddings in context means less processing for faster and cheaper deployment. Additionally once the trained embeddings are fixed, we can use data structures like k-d trees to make the retrieval process more efficient — regardless of whether you reduce the size of the context. As DeepSeek V3 and Flash Attention have shown, there are other hardware-aware changes you can make to your deployment scheme for large efficiency gains. For example, depending on your storage scheme, it might be useful to reorder the embeddings such that similar embeddings are close to each other to make retrieval (i.e. literally moving data) more efficient.

### Interpretability

This structure would be a huge boon for model interpretability. To understand what a knowledge embedding represents, you'd simply **check the queries where the embedding is used**. For example, if embedding 53 is always used on queries about Frederick the Great AND only on these queries, we can be sure that it is in some way related to Frederick the Great, and more experiments would only yield more understanding. This is a **double edged sword**: if the embeddings are so neatly interpretable, then it would also be very easy to remove embeddings about contentious subjects like historical atrocities (see the next section for a counterpoint). This would in itself be another strong interpretability mechanism: removing embeddings and seeing what abilities are lost will help you understand what was stored in the removed embeddings. This is known as an **ablation study** in AI and neuroscience since you are removing or ablating part of the model/brain.

### Continual Learning

We're finally back at point number 2 about making small, intuitive updates. The decoupled knowledge store gives us a **simple recipe for learning new information**: just add a few new embeddings and let the model train only the new embeddings. Any information that wasn't already known to the model must then be captured in the new embeddings. There is no ambiguity about where or how the model learned the new information because the rest of the model was frozen in place. This also provides a recipe for **relearning information lost to censorship**: if you identify factual gaps, then you can train new embeddings on relevant text to restore that knowledge.

I think that a form of continual learning is also the cornerstone of **true agentic models** that can even remotely approximate a teammate or employee. Here's why: no teammate or employee is ever able to perform well out of the gate. They go through an onboarding process and continue to learn on the job for an extended period. This is why an efficient, well structured way to learn after training is so vital. No amount of pretraining, scaling inference compute, or even standard finetuning will ever be strong enough to adapt to constantly evolving contexts: for that you need a **true, native continual learning solution**. Granted, if this setup works, then a model would never learn new skills through continual learning: only new knowledge. However, I'd argue that's fine for a lot of agentic use cases. I think that in many cases, existing models already have the necessary skills; they are just too brittle and inflexible to adapt as needed. At a first principles level, I think it's also pretty reasonable to only learn new knowledge. Skills are these overarching, broad concepts, and there aren't that many of them; they sort of must be trained at scale and unfreezing them after pretraining might inevitably lead to overfitting and/or catastrophic forgetting (yes, that's the actual technical term).

# Failure Cases

Let's talk about some potential failure cases:

-   The knowledge store structure would work in theory, but it doesn't **scale in practice** i.e. it would be too big and unruly for current hardware to handle. I previously thought this was a real concern, but I think there's a simple counterpoint. If the new decoupled architecture is the same total size as existing models, then the different structure shouldn't matter i.e. if you can run a top end standard LLM or standard MoE model, then you can run a model with this decoupled architecture. And I don't really see a case where the new architecture works but requires more size or parameters than existing LLMs for equivalent performance: decoupling should just be more efficient. I think there is one specific version of this concern that stands out: LLMs have this nice, deep, sequential architecture with many layers. The separate layers mean that you have to do less processing at each individual layer and that you need less memory. With something like the knowledge store, you might need to process a ton of memory at the same time even if the total size of both models is still comparable (frankly, I don't understand hardware very well, so apologies if I'm botching this). However, between DeepSeek V3's improvements of the MoE architecture and especially Flash Attention's optimization of the attention mechanism, I'm willing to bet that this is a solvable problem.
-   This is just **too new and too weird**: the scaling laws and several years worth of dense transformer specific techniques will not apply. Note that in the context of "sparse" architectures like mixture of experts, the standard transformer is called a dense transformer. The following fact is a big part of why I'm so confident in this idea:

    > The proposed approach is so native to the existing transformer paradigm.

    It is built entirely on attention. Additionally, ideas like cross attention aren't new and have been validated at extremely large scales with much trickier setups. For example, cross attention is the key to vision language models, which can process both images and text.

-   The architecture works, but it doesn't represent a real leap over mixture of experts because the **embeddings aren't so specialized and interpretable**: I think this is a **pretty fair concern**, but there's a lot that makes me think it's not the case. To be honest, I'm not convinced that existing mixture of experts models are all that specialized (just based on expert usage graphs from the DeepSeek V3 paper), and the paradigm isn't so mature yet. To me, it is easier to see how you can efficiently and effectively separate out different pieces of knowledge than you could do the same for different skills — simply because there are far more "sets of knowledge" than there are "sets of skills". There are a whole lot of tricks and hyperparameter tuning schemes that I think could make a significant difference in how well this architecture works. A good example is the idea I proposed earlier: reducing the context size over time as the embeddings become more refined. I think the question is less "can you find a configuration that produces the desired separation" and more so "will the configuration you find still be sufficiently expressive so that the model can actually learn at scale?" In other words,
    > if there is a **sizable tradeoff between separation and performance**, then this architecture will not be worthwhile.
-   I'm **wrong about the skills to knowledge ratio in practice**: I would find it extremely, extremely difficult to believe that I'm wrong about the skills to knowledge ratio at a conceptual level. And while an individual skill (i.e. an expert) is definitely much larger than a single knowledge embedding, I'm also quite sure that the total size of the experts will be less than the total size of the knowledge store. However, both biology and nonlinear math (in high dimensions) are **unpredictable**. It may be the case that even though skills hold **less information**, it requires **more resources (whether that's weights or neurons) to represent** skills than to represent knowledge. Again I am confident that — at a fundamental level — skills are quite low dimensional and simple, but neither biology or math are intuitive or (fully) optimized. For example, no one would intuitively expect that [Hasbulla and Shaq](https://ibb.co/yFNtQ2m7) (like all humans) share 99.9% of the same DNA.

# Conclusion

I really see this idea as a **goldilocks solution**: it is just close enough to existing ("dense") transformer and mixture of experts architectures that I see very little risk i.e. I think the failure cases are incredibly unlikely. At the same time,

> **it is different enough to be a true revolution: in efficiency, in interpretability, and in continual learning**.

#### References

Liu, Aixin, et al. "Deepseek-v3 technical report." arXiv preprint arXiv:2412.19437 (2024).

Dao, Tri, et al. "Flashattention: Fast and memory-efficient exact attention with io-awareness." Advances in neural information processing systems 35 (2022): 16344-16359.
