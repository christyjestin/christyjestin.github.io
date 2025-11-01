---
date: 2025-10-10T22:57:10-04:00
lastmod: 2025-10-10
showTableOfContents: false
tags: ['probability', 'loss functions', 'statistics']
title: 'Ranking under Uncertainty: The Ease of Gaussians'
type: 'post'
---

_I've been working on formulating a (larger) problem, and this is just a quick idea I came across in that process._

Here's the context: suppose we want to train a **ranking model** that produces a continuous preference score for different options. Now assume that the only data we have is a set of **pair-wise rankings** i.e. \(A > B\), \(C < D\), \(A > D\), and so on. Thus the model is tasked with ranking two choices \(A\) and \(B\), and all we know is which of the two is preferred i.e. we don't know anything about the actual difference between the two: \(A > B\) might mean that it's 10% better or 10x better.

There is a straightforward approach to this problem: you have a model \(r\) that produces a **logit** or a single real-valued score for each choice. Then we say that for any pair of choices \(A, B\), the **probability** that we prefer \(A\) is \\[\frac{e^{r(A)}}{e^{r(A)} + e^{r(B)}} = \frac{1}{1 + e^{r(B) - r(A)}} = \sigma(r(A) - r(B)).\\] Note that the sigmoid function ensures that our probability is actually in \([0, 1]\). Then to train the model \(r\), we simply maximize \(\sigma(r(A) - r(B))\) (or really \(\log(\sigma(r(A) - r(B)))\)) for positive samples (\(A > B\)), and we maximize \(\log(1 - \sigma(r(A) - r(B)))\) for negative samples (\(A < B\)). This simple formulation (known as binary cross entropy loss) is behind many powerful techniques like reinforcement learning through human feedback (RLHF).

There is one shortcoming of this formulation that I wanted to address: it has no room for **uncertainty**. The model produces a single scalar reward without quantifying how certain it is about this prediction. Quantifying uncertainty has some practical applications: at training time, we can reward the model for appropriately calibrating its confidence when making predictions i.e. it's less of a big deal if the model gets something wrong if it prefaced the prediction by saying it was uncertain. Additionally, at test time, we could use the uncertainty to adjust our decision making: for example, you might ignore uncertain predictions. In truth, the latter is often easier said than done as the learning process doesn't always preserve a useful notion of uncertainty.

Ok, now that we have some motivation, let's think about the solution. In order to express uncertainty, the ranking model needs to produce a **probability distribution** instead of a single value. The normal distribution is a natural choice with the model producing a mean \(\mu\) and a standard deviation \(\sigma\) to parameterize the dstribution \(\mathcal{N}(\mu, \sigma^2)\). Thus for a pair of choices \(A, B\), we would produce a pair of distributions \(X, Y\), and we can say that prefer \(A\) over \(B\) when \(X > Y\).

Thus the probability of preferring \(A\) is \(P(X > Y)\), and for general (independent) distributions \(X, Y\), we can express this with their respective PDFs and CDFs: \\[\int_{-\infty}^{\infty}f_{X}(x)F_{Y}(x)dx.\\] This form isn't pretty if you plug in the Gaussian PDF and CDF since we'd have to deal with an exponential and an error function. I was feeling quite daunted and tried formulating some alternative integrals: for example, there's a very natural connection to integration by parts:

\\[\int_{-\infty}^{\infty}f_{X}(x)F_{Y}(x)dx = F_{X}(x)F_{Y}(x)\bigg\vert_{-\infty}^{\infty} - \int_{-\infty}^{\infty}F_{X}(x)f_{Y}(x)dx.\\]

The first term is our \(P(X > Y)\), and the middle term evaluates to 1. The final term is an integral for \(P(X < Y)\), so in total we have \\[P(X > Y) = 1 - P(X < Y).\\] There are also some nice connections to indicator functions, LOTUS (the law of the unconscious statistician), and Adam's law.

Luckily, Gaussian distributions have some incredibly useful properties, and I didn't have to worry about any of this. There's one key observation here: \(P(X > Y) = P(X - Y > 0)\). In general, this wouldn't mean much because the distribution of \(X - Y\) could still be unruly, but in the Gaussian case, the linear combination of Gaussian random variables is also Gaussian: \\[X - Y \sim \mathcal{N}\big(\mu_X - \mu_Y, \sigma_X^2 + \sigma_Y^2\big).\\] Thus we can just use the Gaussian CDF to compute a closed form probability: \\[P(X > Y) = 1 - P(X < Y) = 1 - P(X - Y < 0) = 1 - F_{X-Y}(0).\\] Alright that's it: just a quick idea. Thanks for reading.