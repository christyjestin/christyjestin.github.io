---
date: 2025-11-01T1:10:10-04:00
lastmod: 2025-11-01
showTableOfContents: false
tags: ['sorting', 'complexity', 'algorithms']
title: 'The Sorting Saga'
type: 'post'
---

_This was originally a LinkedIn post. Each link has the technical details of the different approaches._

Been playing around with sorting algorithms over the last couple days. It's been a surprisingly strange experience that started with me musing about whether there's a [better way](https://x.com/christyjestin/status/1983698270464680070) to do random_sort.

First day, I thought I had a (usably) **linear** sorting algorithm; that sent me into an identity crisis for a good three minutes as I tried to figure out why it didn't work. Eventually I realized that parallelism was just hiding compute: [Link](https://x.com/christyjestin/status/1983728336179491181).

Second day, I very enthusiastically wrote an algorithm called **bucket_sort** that had pretty good complexity in most cases. At the very end, I pasted my algorithm into Claude and asked if there was a similar known algorithm. Turns out it already existed...with the same name ¯\\_(ツ)_/¯: [Link](https://x.com/christyjestin/status/1984088067712676142).

Third day, I came up with **diffusion_sort**. The complexity isn't great, but the core idea is neat, and the gradual sorting process produces a beautiful graph: [Link](https://x.com/christyjestin/status/1984491861566763101).

![heatmap](https://i.ibb.co/M5hyxcLw/heatmap.png)

Alright, that's it: I hope :)
