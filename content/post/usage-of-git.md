---
title: "Git笔记"
date: 2018-06-02T23:41:24+08:00
draft: false
comments: true
keywords: ["git"]
tags: ["git"]
toc: true
autoCollapseToc: false
---

> 整理一下最近学习的git知识，以及平时常用的git功能。

## .git

使用git init或clone一个远端仓，会在本地建立一个.git目录。**这个目录是git仓的全部，把.git拷贝到其他目录下，就能在该目录下建立一个一模一样的git仓**。

## 缓存区（staged）

* 对working derictoy中的文件做的改动，他们的状态是unstaged
* 使用 `git add`/`git rm`/`git mv` 将其送入缓存区（staged）
* 使用 `git commit` 提交缓存区中记录的改动。
* `git diff {filename}` 可以查看unstaged和staged中文件的不同
* `git diff --staged {filename}` 可以查看staged中的文件和原文件的不同
* 注意staged和`stash`的区别

## 上游分支

`git clone`可以通过参数 `-b` 来指定clone远端仓库到本地后拉取哪条分支，不指定则默认拉取`master`；远端仓库中必须存在同名分支，作为本地分支的上游分支。

通过`git branch -vv` 或 `git status` 命令可以查看本地分支相比上游分支领先/落后多少个commit。

`git checkout -b {local_branch} {remote_branch}`用来创建并切换分支，并指定该分支的上游分支。

## revert和reset

`git reset`把HEAD指针指向到某一个commit id，这次commit之后的所有commit都会被删除。

`git revert`用来撤销某一次commit带来的变化，不会影响其他commit。revert本身也需要commit。

非fast-forward形式合并两条分支时，git会自动生成一个合并提交。如果想回退某条分支的merge操作，可以revert这次合并提交的commit，git会让你选择留下这次合并提交的哪一个父分支，另一个父分支所作的改动会被回退。

## 如何修改一次历史commit

执行`git rebase -i {commitid}^`（commitid是想要修改的那次提交），git会以commitid的前一次提交作为base，采用交互式的方式，重新提交后面的每一次commit，将想要修改的那一次的提交命令设置为edit即可。
