---
layout:     post
show_meta: true
title:      Using Git Rebase to cherry pick
header:     Using Git Rebase to cherry pick
date:       2016-04-04 21:00:00
summary:  Using git rebase to select specific commits from a separate branch.
categories: git
author: Chee Yeo
---

While working on a project recently, I created a separate feature branch to start developing new features. During the process, the master branch I forked from had certain features which would cause my current development branch to break if I pulled from it and yet I needed my specific commits to be brought into it cleanly i.e. to be replayed on top of the HEAD of the master branch.

Normally, I would just do a rebase of the master branch into my development branch but since my development branch has now diverged and also include breaking changes to the master branch, it would not be acceptable.

The solution I came up with was to select the commits I wanted from my development branch up to last commit, create a new branch off it and then rebase it onto the master branch.

Say we have two commits from the feature branch, 82cada and 92ecb3 and 62ecb3 is the head.

{% highlight bash linenos %}
# from the development branch
git checkout -b newbranch 92ecb3

# select commit from 82cada up to 92ecb3
git rebase --onto master 82cada^
{% endhighlight %}
