---
author: ["Paul Wong"]
title: "Git Tricks"
date: "2019-01-10"
description: "Git Tips and Tricks"
summary: "This post has some useful Git tips and tricks"
tags: ["git", "sorting", "snippet"]
categories: ["git", "snippet"]
ShowToc: true
TocOpen: true
---

[<img src="https://imgs.xkcd.com/comics/git.png" alt="Git is hard">](https://xkcd.com/1597/)

## Config changes

### :rocket: A better git log :point_left:

```bash
% git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

### Fast-forward only for pulls

This will help avoid merge conflicts on pulls. [Why?](https://blog.sffc.xyz/post/185195398930/why-you-should-use-git-pull-ff-only)

> In its default mode, git pull is shorthand for git fetch followed by git merge FETCH_HEAD.

```bash
% git config --global pull.ff only
```

## Add tracked files only

```bash
% git add -u
% git commit -a -m '<insert your comments here>'
% git push
% git pull origin master
```

## Get files from repository

### Go to the root directory you want the rep directory

```bash
% git clone https://code.server.com/pwong/<repo_name>.git
% cd <rep_name>
% git pull
```

## Add files to git hub

```bash
% git add <filenames>
% git commit -m 'add your comments here'
% get push
```

## Get updated files from hub

```bash
% git pull
```

## Rebase with squashing

```bash
% git rebase -i
```

## Show names only on commit

```bash
% git diff-tree --no-commit-id --name-only -r 7c255b07b
```

## Show what a commit added

Commit hash and add `^!`

Example

```bash
% git diff 07d167f^!
```

File names only

```bash
% git diff --name-only c022600^!
```

## Diff a branch

```bash
% git diff master...<your_branch>
```

## Bundle

Bundle allows you to get git changes from one machine to another.

On the source machine.

```bash
% git bundle create file.bundle HEAD
```

On the destination machine.

```bash
% git pull file.bundle HEAD
```

## Error: Fatal: Not possible to fast-forward, aborting

```bash
% git pull --rebase
```

## Resolving merge conflicts for the entire file

If you face merge conflicts, but you know that one version of the file is the “right” version, there is a nice shorthand using the flags --ours or --theirs

```bash
% git merge feature/some-new-feature
CONFLICT (content): Merge conflict in conflict.txt
Automatic merge failed; fix conflicts and then commit.
```

```bash
% git checkout --theirs conflict.txt
% git add conflict.txt
% git commit
```

This is effectively the same as going through the conflicting file and slicing and dicing all the `<<<<<<<<<<<<` `>>>>>>>>>>>>` stuff to pick one side of the merge exclusively but without all the headache and error potential

Note theirs/ours is from the perspective of your current branch (i.e. a conflict resulting from `% git merge feature-branch` means `% git checkout --theirs <file>` is the same as `% git checkout feature-branch --<file>`)

## Delete a branch

Local

```bash
% git branch -D <local-branch>
```

Remote

```bash
% git push origin --delete <remote-branch-name>
```
