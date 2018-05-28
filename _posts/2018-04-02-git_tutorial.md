---
title: "Oh no, another merge error?"
date: 2018-04-2
excerpt: "A git tutorial for people with commitment issues"
---
I guess I'll preface this with - Git is great! As a version control tool, Git helps your break your code into manageable chunks for review, track changes in your codebase, and allows for seamless collaboration between multiple individuals.

However, most of the time at work, I am writing code for multiple projects at once across multiple repos. As such, I often find myself in a situation where I now have code for multiple issues that I need to commit all at once, and more often than not, this results in a merge error. Either that or I just do something stupid like forgetting to rebase to master, and everything breaks.

Below, I'll go over some commands that have been able to get me out of 99% of git issues that I've had.

## Git Basics (What you should be doing)
Of course we'll start out with what you should be doing so you don't run into issues in the first place. Use `git status` liberally to make sure you're covering all your bases. 

1\. __Merge all differences from origin/master onto your code__  
You should always start with this!
```bash
git rebase origin/master
```

2\. __Create a new branch__
```bash
git checkout -b feature/my_cool_feature
```

3\. __Stage and Commit your changes__
```bash
git add my_cool_feature.py
git commit -m 'Helping The Man meet his EBITDA goals'
```

4\. __Push your changes upstream to origin__
```bash
git push -u origin feature/my_cool_feature
```

## Fixing your less serious commit error
If you want to change the parameters of your last commit, it is a relatively easy fix using the `--amend` argument
```bash
#Pull up your editor to fix the commit
git commit --amend

#Change author
git commit --amend --author 'Matt Guan <email@email.com>'

#Change message
git commit --amend -m 'New commit message'
```

## Fixing your more serious merge error
Below is the easiest, most full proof way I've fond for fixing any merge error.  
1\. __First stash your work. Consider backing it up locally too.__
```bash
git stash
git stash list
```

2\. __Nuke your entire branch (Trust me, this is the easiest way)__
```bash
git reset --hard origin/master
```

3\. __If you managed to commit already and need to kill it__  
i.e if your git hosting service requires commits to contain a certain keyword, you committed locally, but you can't push it upstream
```bash
git reset --hard HEAD^
```

4\. __Finally, just apply the stash back on like nothing happened__
```
git stash apply stash@{0}
```

Of course, there are many other ways to fix merge conflicts, but I've found this way to be the easy, foolproof, and require the least amount of thought. Thanks for reading!