---
date: 2025-12-27T23:10:10-04:00
lastmod: 2025-12-27
showTableOfContents: false
tags: ['visualization', 'open problems', 'math']
title: 'Visualizing the Collatz Conjecture'
type: 'post'
---

I've been playing around with the Collatz conjecture. I wrote up this [**visualizer**](/collatz/#/) after noticing how nice the binary representation is. That's the main point of this post. I also did some data exploration yesterday [here](https://x.com/christyjestin/status/2004682306649469255?s=20) and [here](https://x.com/christyjestin/status/2004759229278912663?s=20). I feel like each time I look at the conjecture, I have some novel perspective, but the novel perspective just offers a fresh reason to say "Oh ye, this shit's gonna be so hard to prove."

I'd like to thank Github Copilot/Claude Haiku 4.5. This was my first time using it or any other AI coding tool directly in a codebase. I've always just used LLMs in the browser (so that I can largely control context), and I think I still largely prefer that workflow after today. There's one major exception: **styling**. Copilot wrote all of the CSS for the site. CSS is by far my biggest weakness, and it's the thing I despise the most about any web dev or GUI work. This project took the better part of a snowy day, but it would've taken much longer and ended up with a much uglier result if I didn't have Copilot to do the styling. Early on, I almost rage quit over some stupid styling stuff, and I only kept going because I promised myself that I'd make Copilot do all of the styling entirely on its own.

Copilot's autocomplete was also decent, and it wrote the JSX container logic for the bits, but it immediately broke whenever I tried getting it to do any more complex logic stuff, and I was pretty happy to write and engineer all of that myself. That said, I used up the entire Copilot free quota by the end of the day anyway. All in all, this experience has definitely made me much more amenable to web dev again.