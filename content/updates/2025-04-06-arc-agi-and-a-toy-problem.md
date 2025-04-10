---
date: 2025-04-06T18:58:10-04:00
lastmod: 2025-04-06
showTableOfContents: false
tags: ['arc agi', 'benchmarks', 'machine learning']
title: 'ARC AGI and a Toy Problem'
type: 'post'
---

I'm very excited to start working on the ARC AGI [benchmark](https://arcprize.org/arc-agi). I think the requisite capabilities are both really important and quite different from what existing models are trained to do — and that **significant progress can be made on the benchmark without an ungodly amount of compute**. However, I believe that the puzzles are a bit too complicated i.e. in order to do well on the benchmark, you have to come up with good solutions to the underlying search/program synthesis/pattern finding problem and come up with the right architecture, features, etc. to apply this solution to the computationally challenging puzzles (see an example puzzle below). Instead I want to decouple these two aspects by first working on a similar but simpler problem.

![image](https://github-production-user-asset-6210df.s3.amazonaws.com/52580002/430746430-4d92eb75-1683-421c-8da4-a53e6c5f9fd0.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA%2F20250407%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20250407T004858Z&X-Amz-Expires=300&X-Amz-Signature=ae4248e0918d0c13acdda06ff15cd406c5b1dc9176379342547123f272d75a5b&X-Amz-SignedHeaders=host)

First, let's talk about the ARC AGI puzzles. You're given a series of example input-output pairs of grids and then asked to predict the output for a new input grid. The challenge is to find an underlying rule that explains how you mutate an input grid into an output grid. However, to do so, you must come up with good representations for both individual grids and the transforms between grids as well as feature extraction methods that suit these representations. This is hard when you're working in three dimensions (two dimensions for x and y position and a third for color). And I think it might be a critical bottleneck: you could come up with a great search algorithm that is kneecapped by poor representation learning methods.

I believe that the right approach is to first work on a toy problem without the representation challenges, then try to adapt a solution to the grid puzzles after validating it on the toy problem. I think there are a few key characteristics that are desirable in a toy problem:

-   It should have the same structure i.e. finding a common rule from a few examples
-   You should be able to scalably generate many, many puzzles without human labelling
-   There should be a reasonable curriculum learning scheme to slowly ramp up the difficulty of the problem
-   There should be a natural way to give partial credit i.e. some sense of how close you are even if you don't nail the solution

There's a toy problem structure that came to mind immediately. First choose some category \(\mathcal{X}\). Then choose some subset \(\mathcal{S}\subset \mathcal{X}\) with common underlying characteristics. The task is as follows: given a few examples \(\{s_1, s_2, \ldots, s_n\}\) from \(\mathcal{S}\), and a few potential members \(x_1, x_2, \ldots, x_n\) (one of which is actually part of \(\mathcal{S}\)), choose the member \(x_i\) that is most likely to also belong to S. Alternatively you could present single candidate \(x\) and ask if it also belongs in \(\mathcal{S}\). In both versions, you are able to give partial rewards because the model would be predicting a probability, and a higher probability for the right answer would mean a higher reward even if you, for example, don't cross the 50% threshold.

There's a factorization based version of this structure that is easy to implement. The category \(\mathcal{X}\) is all natural numbers, and a subset \(\mathcal{S}\) is defined by some common factor p. For example, if your common factor is 17, your examples might be 153, 51, 340, and you might ask if 192 belongs in the set. Note that this problem becomes more difficult with factors that are larger as well as example numbers that are large and/or have many factors. This gives us a natural curriculum.

I will be posting progress updates for my work on this problem as well as the actual ARC AGI benchmark.
