---
layout: post
title: "How to Fix Non-existent 'Unstaged Changes' Issue in Chromium"
date: 29-11-2017
comments: true
---
I updated my Chromium source code in my machine and `gclient` was failing to sync. A similar problem has been mentioned in
[[1](https://bugs.chromium.org/p/chromium/issues/detail?id=506040)], 
[[2](https://bugs.chromium.org/p/chromium/issues/detail?id=584742)], and
[[3](https://groups.google.com/a/chromium.org/forum/#!topic/chromium-dev/b68HNfnWfZQ)].

It seems that I messed around with the Chromium project and I ended up detaching its git repositories from their HEADs.
``` bash
$ git status
You are in 'detached HEAD' state...
```
I devised a solution to this problem by dividing into three stages.

1. Remove delete `depot_tools` and clone new version without changing its location.
```bash
rm -rf depot_tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```
2. Clean all the projects in `src` directory of the Chromium project by recursively visiting each project. 
Also [checkout the `master` branch](https://stackoverflow.com/questions/10228760/fix-a-git-detached-head) after cleaning:
```bash
export HM=$PWD
find `pwd` -type d -name ".git" | sed  s/.git$//g |  \
   while read r; do cd $r; git reset --hard HEAD; git checkout master; done
```

3. Update all branches and call `gclient sync` again
```bash
git rebase-update && gclient sync
```
If everything goes well, you can rebuild your Chromium code.
