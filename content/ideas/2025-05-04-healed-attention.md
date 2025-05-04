---
date: 2025-05-04T12:16:13-04:00
lastmod: 2025-05-04
showTableOfContents: false
tags: ['attention', 'context', 'caching', 'reasoning']
title: 'Healed Attention'
type: 'post'
---

_I wrote this as a research proposal to send to a friend, but figured it's worth publishing given that I've already gone through the effort._

# Background

## LLMs

Generative large language models (LLMs) are autoregressive and causal. This means that they generate one token at a time, and the \(n\)th token is analyzed in the context of only the previous \(n-1\) tokens i.e. the value of \(n+1\)th and later tokens have no impact on token \(n\). This means that when computing attention (the core mechanism of these models), representations of previous tokens can be cached and reused.

## Preprompts

Since LLMs are by design highly dependent on the context, useful behaviors can be induced by prompting the model with instructions or by giving useful information as context. Versatile prompts can be pre-computed and reused as long as they are used at the beginning of LLM queries. This idea has been exploited by every LLM based tool in the form of system prompts, which give useful context specific to that tool. It is critical that these prompts are at the beginning of queries since then (and only then) their attention computation is unaffected by the user's actual input.

# The Idea

Just want to say up front that this is not an architectural change or even one that requires new training (or at least not necessarily). My proposal is that \*_full attention is not necessary in every case8_. Instead you can pre-compute passages separately and fix the breaks in attention by generating more text to "heal" the gaps.

## An Example

Consider five chunks of text: \(A\), \(B\), \(C\), \(D\), and \(E\) that appear in that order. For example, \(A\) could be a user query about some academic subject; \(B\), \(C\), and \(D\) could be relevant passages from a textbook, and \(E\) could be the model's response. Under the current paradigm, there is no reason to pre-compute (or pre-process) \(B\), \(C\), and \(D\) — no matter how often they show up. The reason is that the attention computation changes when the context changes: because \(A\) is something novel, there's no way to account for it ahead of time. Even if you restructured the text to be \(BCDAE\) so that the reusable sections come before the user query, we'd still have the same problem. For this particular query, we might want \(B\), \(C\), and \(D\) (in that order), but for another problem, we might want different passages like \(XYD\). The context of \(D\) has changed, so we have to recompute things.

## But Why?

This doesn't line up with human intuition: you don't have to re-read a book every time you think about it in a new context. Humans have some concept of **scope**: instead of considering absolutely everything available as context, we process things with some limited, relevant scope and the knowledge learned in that limited scope can be reapplied pretty versatilely.

You can apply this idea to LLMs through masking: just like how earlier tokens are processed without knowing about later tokens, we can restrict the context for each section. Recall that each of our chunks (\(A\)-\(E\)) are themselves composed of many tokens. Since \(A\) is at the beginning, the tokens of \(A\) already only attend to themselves, but we can do the same for \(B\), \(C\), and \(D\). For example, when processing \(D\), we just mask out \(A\), \(B\), and \(C\). Since \(D\) is an actual self-contained and coherent passage, the limited scope still has enough information and grounding for the model to understand \(D\).

This is a huge improvement in **efficiency and latency**: since we're masking the attention, processing \(D\) in the context of the user query and the other passages is equivalent to processing it by itself. Thus we can pre-compute passages like \(B\), \(C\), and \(D\) and reuse those vectors in any context. For example, if there's a new user query \(Z\), the passages won't care because \(Z\) is getting masked anyway. Same deal if you need to swap out \(B\) for another source that's more relevant to the new query: \(C\) and \(D\) don't care.

## Why the fuck would this work?

Obviously, you still need to be able to use the user query as context, and you do need a way to relate the different passages to each other. The solution is simple: \(E\). After pulling in different sources, you need to synthesize an answer \(E\). If you just let \(E\) attend to all previous passages, then you can bring everything together to synthesize a complete response. Now granted, that means \(E\) is attending to a whole lot, and that step will still be computationally expensive.

It's still a big improvement because of how attention works: attention scales quadratically in the context length. Under the current paradigm: if we gave the model more sources before having it answer the query, then the computation would scale quadratically in the number of sources. By using healed attention, the computation would instead **scale linearly in the number of sources**.

Let's walk through our example (\(ABCDE\)) at a high level to see how: first we process the text under the current paradigm. When processing \(A\), there is no earlier context: the tokens of \(A\) just need to attend to themselves, so we just use one unit of compute. When processing \(B\), the tokens need to attend to themselves as well as the earlier tokens of \(A\). This expends two units of compute. Similarly, \(C\) attends to itself, \(B\), and \(A\), so we need 3. \(D\) requires 4, and \(E\) requires 5. That adds up to 15, and the general expression for \(n\) chunks is \(\frac{(n+1)n}{2}\) units of compute which is \(O(n^2)\) or quadratic.

Instead consider healed attention. \(A\), \(B\), \(C\), and \(D\) are processed without additional context, so they each require one unit of compute. \(E\) has everything in context, so it requires 5 units just like it does in the current paradigm. That's a total of 9 units of compute, and the general expression for \(n\) chunks is \(2n-1\) which is \(O(n)\) or linear. 9 and 15 might not sound so different, but remember: the difference between linear and quadratic only becomes starker as \(n\) rises. For example, if we instead use 18 sources (which is 20 chunks including the user query and the model's response), then the compute required in the current paradigm is 210 units while healed attention only needs 39 units. Moreover, 18 of those units (one for each source) can be pre-computed and reused.

Not only is this more efficient in terms of compute, it's also a big, big change in latency. LLMs are **sequential**: they have to process one token fully before looking at the next (this is more nuanced in training, but we're just talking about inference right now). This means that those 210 units cannot be sped up by working on multiple at the same time. 210 units of compute means 210 units of time. Under the healed attention paradigm, the first 19 chunks are each self-contained so they can be processed simultaneously even if you're not caching. The final chunk is processed as normal and does require 20 units of time, but that's fine because that chunk is what ties everything together.

Thus the current paradigm requires 210 units of compute and will take 210 units of time to finish running. Healed attention with caching requires 21 units of online compute and 18 units of offline pre-compute with a latency of 21 units of time. Finally, healed attention without caching will require 39 units of online compute and 21 units of time.

Now granted, it's quite possible that your final synthesis text has to be longer in the healed paradigm to bring everything together. That's totally fine: the gap between the two paradigms gives you a huge allowance in terms of both compute and time: you can generate a whole lot more tokens and still have massive gains in efficiency. It is essentially guaranteed that this approach will be more efficient in a world where it works. The question is just: does it work?

## Saving Grace

A big factor that might have screwed up this idea is **positional embeddings**. These embeddings modify token representations to give the model some information about where two tokens appear while trying to understand how they relate to one another. For example, the words "hot" and "dog" mean one thing when they're next to each other and something very different when they're 20 words apart. Luckily, modern LLMs use rotary position embeddings (RoPE) which only encode where tokens are relative to each other rather than where they are in absolute terms. This means that changing the context has no effect on the computation: even though the absolute positions are changing, the chunk is moving together, and the relative positions of tokens do not change.

## Reduced Price Lunch

You might be thinking that this sounds too good to be true. The catch is that if you are pre-computing passages and caching them, then you'll need a good amount of **memory**. An obvious choice here is to use a Least Recently Used (LRU) cache which will keep the most popular passages in our pre-compute cache. It's not a free lunch because of the greater memory requirements, but it's still a significant net gain. Also as I mentioned, you can get efficiency and latency gains even without pre-computing, and in some applications, pre-computing doesn't make sense anyway.

# Applications

There's three applications of this idea that immediately stand out.

## Low Hanging Fruit

The first is the example that we've been discussing: **retrieval augmented generation (RAG)**. In this technique, relevant sources are retrieved from some knowledge base and passed to the model as additional context. These sources could be articles, passages from a textbook, or even example solutions to similar problems.

## Natively Scoped

The second is **code**. This one is a bit interesting because you could do interesting things during pre-computation. For example, when processing some code from a library, you might use the entire library as context to analyze each file. Then if/when you import and use the code from a specific file, you can use just the vector representation of that file i.e. just the tokens from that file. These tokens are still enriched because they were pre-processed in their original text. In some sense, when you use the enriched tokens, you're more so bringing ideas into your new context rather than just bringing text into the new context.

This concept is also extremely natural for code. Code is written with **abstraction** in mind — or at least the most used libraries are — and programming languages have some concept like scoping or namespaces to help separate out concerns and keep the context manageable. In the sort of Platonic ideal, you don't need to know anything about the underlying implementation: the interface (API) is all you need to know. Obviously, there's plenty of code that doesn't fit this ideal, and much of the user generated code that is processed by a model will not be perfectly abstracted. This is where having that **extra local context** from pre-processing will help.

## Gold Mine

The third is **reasoning models**. Very little is known about these models. We do have an open source implementation in DeepSeek's R1, but R1 used the somewhat simple/"primitive" GRPO technique, and there's speculation that models like OpenAI's o series do something more sophisticated. One suggestion is **Monte Carlo Tree Search (MCTS)**. In this technique, instead of generating a full chain of thought in one go, MCTS would generate multiple chains of thought simultaneously and in stages. You could do something like:

1. Initialize \(n\) separate chains of thought for the same problem
2. Generate the next step for each partial chain
3. Evaluate all of your partial chains and re-rank them
4. Keep your best \(\frac{n}{2}\) chains and duplicate them
5. Return to step 2

Through this process, you ensure that all of your chains are productive: for example, if one chain gets stuck, then you'd just drop it and give that spot to a more promising chain of thought. By branching off multiple paths from one chain (in step 4), you also ensure that your search has enough breadth and can find interesting paths.

There's another version of this where instead of having a formal graph with explicit branching and reranking, you use **recombination**. This is a metaphor from genetics: what you want is for some ideas from one chain to transfer over to another. For example, you can just generate some synthesis text to analyze and summarize the chains i.e. to figure out what's working and what's not. Then based on this synthesis text, you can generate \(n\) fresh chains of thought that implicitly build on top of the prior progress.

Ok, so that all sounds nice, but what the fuck does that have to do with pre-compute? Take a look at the **synchronization step**: each chain is being generated on its own, but at the re-rank/recombination step, all of the chains have to be brought together into one context. Why? Because a common context is the only way you can relate the different chains to each other. This means that all of the text generated by the separate chains will have to processed again, together, in a fresh context. Sounds familiar, doesn't it? What if we instead just took the token representations from the existing chains without having to reprocess the tokens?

This is not just another efficiency gain: some solution like this is **critical for the so-called "test time compute" paradigm**. At its core, this kind of technique is trying to leverage **parallelism**: it's exploring multiple threads at once and spreading out its resources. The core concept is that if you're doing multiple things at the same time, then you can get more done in less time. The only problem is that LLMs are painfully sequential, so it doesn't really matter (from a computational perspective) that you're generating multiple chains at the same time. During the recombination step, you are putting everything back into a single sequence and redoing all of the compute — which is a **huge hit to latency**.

There is a reason to use multiple chains anyway: **exploration**. Having distinct branches allows the model to consider more options, and this is why I think reasoning models might still use MCTS. It's also the case that this synchronization step is critical: it allows you to transfer insights and more efficiently utilize resources. It's almost certainly better than an approach like consensus@n where you just let multiple chains of thought generate answers independently and then have them vote. While this likely has some performance boosts (through reducing variance), there will be plenty of questions where the most frequent answer is still not the correct one because at the end of the day, if your reasoning is wrong, then doing your bad reasoning over and over again won't magically fix things. To get over the hump, you have to compare the different lines of reasoning and deliberate over why one line is better than another — not just blindly look at the different answers.

With healed attention, you get exploration as well as the latency and efficiency gains traditionally associated with parallelism. Scaling test time compute without true parallelism is untenable: it will require too much compute, and users can't sit around waiting for an LLM to process 30,000 tokens in sequence to answer everyday questions.
