---
date: 2025-02-17T16:49:10-05:00
# description: ""
# image: ""
lastmod: 2025-02-17
showTableOfContents: false
# tags: ["",]
title: 'Clyde and RajaRavi'
type: 'post'
---

I've gone into some of the technical details on both of these projects in their respective READMEs ([clyde](https://github.com/christyjestin/clyde/blob/main/README.md), [rajaravi](https://github.com/christyjestin/rajaravi/blob/main/README.md)), so I'll talk more about the process here.

My goal in both projects was to write an algorithm that could take in an arbitrary image and produce a variation that looked like a painting. Every algorithm was based around the idea that good paintings simplify the reference material and don't have as much detail as a photograph. There are 4 final algorithms that came out of this effort (pixel k-means, patch k-means, and gradients in `clyde` and patches in `rajaravi`). Pixel k-means worked out of the box and ran quickly, but all of the others required a lot of engineering and patiently waiting (up to 2-3 mins in the worst cases) for the pipeline to run. And much of the engineering effort isn't reflected in the final codebase — at one point, I rewrote the entire patches algorithm to run with the accelerator `numba` only to discover that `numba` did not support a key data structure (for no apparent reason). I did use my new favorite library `jax` in clyde.

A couple takeaways from the project:

-   There is more to paintings that just simplifying the colors (the next concept I tried to add was hard vs soft edges, but I couldn't come up with any satisfying methods).
-   It is very difficult for clever code to beat fundamental lower-level limitations: the fundamental limitation of the algorithms in `rajaravi` is that they were resistant to parallelization, and no amount of tricks could get past that — everything was immediately faster when I could parallelize everything, remove some for loops, and use `jax` in `clyde`.
-   Set hard deadlines for yourself to avoid spending months on the same idea (maybe, idk, I'm still conflicted about whether I should've spent this much time, and it's also not the first time I've been maybe overcommitted to a project)

I do think I found a good "inductive bias" in the gradients algorithm. The problem is that it has a ton of tunable parameters, and I didn't find a satisfying generic configuration. However, I am still confident that the algorithm would do a great job filling in the gaps if a human just selected anchor points instead of the algorithm doing it.

Some example generated images:
![image](https://github.com/user-attachments/assets/5935b256-9f99-4d77-98c0-c8ac78045040)

![image](https://github.com/user-attachments/assets/ab5576e9-5034-403e-8481-291a2f5a1e9a)

![image](https://github.com/user-attachments/assets/c62e5000-57a9-40cb-b256-ab6b5f707c34)
