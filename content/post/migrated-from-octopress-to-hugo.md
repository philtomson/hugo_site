+++
title = "Migrated blog from Octopress to Hugo"
date  = "2018-06-11"
slug  = "2018/06/11/migrated-blog-from-octopress-to-hugo"
Categories = []
+++

I went to add a new blog post last week and then tried to run octorpress to generate the static html for the blog as I had been doing in the past. But of course it wasn't so simple. I'd upgraded my dev machine to Ubuntu 18.04 a few weeks back and Octopress wasn't happy with the version of Ruby I had. When I went to try to build the old version of Ruby that it wanted, that build failed (likely something is more strict in gcc-7.x now). 

Since Octopress seems to be moribund (no updates in several years) I first tried just going to Jekyll. But the Jekyll's default theme minima is... well, ugly. When I tried some nicer looking themes it seems they weren't setup for blogging and that led to problems.

Finally asked on the #pdxtech IRC channel and someone suggested [Hugo](https://gohugo.io/). And it even has a migration path from Octopress... of course that didn't mean that everything worked out of the box. I had to make changes to get code snippets to highlight. It also didn't like that I had "C++" in the title of one of my posts. And of course I had to make changes to get disqus to work (and it may not be working still). Some small tweaks were needed to get the equations rendering in MathJax. But here it is a few hours later (ok, several hours later) and things are mostly working.

...now I just need to remember what I was going to blog about last week.
