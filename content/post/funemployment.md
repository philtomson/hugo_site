+++
title = "FUNemployment"
date = "2017-02-02"
slug = "2017/02/02/funemployment"
Categories = []
+++

I finished up a 16-month LLVM contracting gig at the end of 2016. Got the flu a couple of weeks ago and have been pretty much out of commission until today when I finally had enough mental clarity and energy to get this blog going again.

Since I last posted in 2014 it seems that Octopress has been updated. I had to go back to my old desktop machine and find where all the blog-related files were and transfer them to my current desktop machine. Of course I tried looking at the Octopress documentation to figure out how it works since I'd completely forgotten in the last ~2.5 years. And of course, the docs no longer cover the old version of Octopress. So I went with the newer version and tried moving things over. For the most part it works. The twitter button that used to be over there on the right doesn't seem to want to show up anymore and I have no idea why (it's right there in the \_config.yml). And pygments now seems to want a space between the language name and the ']' - so whereas '[lang:ocaml]' used to work for the code highlighting incantation, now it seems to want '[lang:ocaml ]' ... weird.

So now that the blog _mostly_ seems to be working again, I'm going to be looking into BNNs [(Binarized Neural Networks)](https://arxiv.org/pdf/1602.02830v3.pdf.). Weights in BNNs are represented as binary values and instead of matrix multiplications (as is common in regular neural nets) the main operation on binary weights is XNOR which is a lot faster and more amenable for implementation in hardware like FPGAs. But more on BNNs later, I hope to start blogging about my BNN investigations which is why I wanted to get the blog going again.

EDIT: Well, that didn't last long. A few days after writing this entry I got a call from the company I was contracting for previously and they had a new project which lasted till close to the end of the year.
