---
layout: post
title: Git Cheatsheet
date: 2019-11-12 00:58:27
tags: [git]
categories: dev-tool 
---

### 首篇blog献给我最最最喜欢的小梁！

<br>

> 工欲善其事，必先利其器。

Since this blog does not go through sufficient polishing period, there might be some mistakes or some outdated content, feel free to contact me for any `bug` you find. Thanks in advance! Another issue is that some naming convention is not quite consistent throughout this blog because this is my first time to write English blog. 


## Introduction 
Git is a version control system used to facilitate the collaboration between multiple developers by using version control (everyone can controls their version to make sure all the features can be developed concurrently).  

If you are not familiar with basic operations of Git, then probably you can search for `git book` to learn more about Git basics. This blog will focus on the tricks and problems during my usage of Git in my daily work. 

You may notice that several commands appear several times in different sections, this is the expected behavior since those commands will function differently with different arguments. The `Git commands` part will introduce commands by functions, hence some commands might be diverged into different parts. 

I would like to There are three kinds of changes in local environment. 
1. changes in the staging area (changes to be committed in next commit) (added into `index`)
2. changes not in the staging area but still tracked
3. changes not tracked or ignored 

## Memo first
+ words in `<>` is variable name
+ phrase in `this format` and start with `git` are command and its parameters 
+ `<remote-name>` for example: origin
+ `<branch-name>` / `<local-branch-name>` / `<remote-branch-name>`: master


## Git commands

### Simple execution (normal execution)
#### git init 
#### git add 
#### git commit 
* `-a`: stage all the tracked files
* `-m`: attach the commit message in commit command, no need to enter text editor to type commit msg
    
#### git rm
* no arguments: the file will go away and not tracked
* `--cached`: still keep the file in hard drive but not have Git track 

#### git mv: seldomly used 
#### git checkout
* no arguments: switch to an existing branch
* `-b`: switch to a new local branch with same file status of current branch 

#### git tag 
* `-a <tag-name> -m <tagging-msg>`: create a tag with tagging msg
* `<tag-name>`: short command 

#### git config
* `--global alias.<alias-name> <command-name>`: create alias for one command 
    For example, `$ git config --global alias.last 'log -1 HEAD'` will set `git last` to perform as `git log -1 HEAD`
* `--global pull.rebase true / false`: `true` will set the default pull method to `rebase` while `false` will set to `merge`

#### git clean
* `-df`: remove all the files that are untracked

#### git stash 
* no arguments / `push`: push tracked changes that are not yet ready to commit into stash stack 
* `list`: view the list of stash
* `apply` / `apply stash{<version-number>}`: apply changes that you stashed into stack previously or some older version specified by `<version-number>`
* `apply --index`: only reapply staged changes. 
* `pop`: apply changes and pop the saved changes out from stash stack

Actually there are some cases that you may make the changes in one branch and want those changes to take effect in another branch. `git stash` is a command for you. 




### Checking local status 
This part is like GET request in HTTP perspective, it will not do any changes to the file status, it will only check status 

#### git status
#### git diff 
* no arguments: see changes not yet staged
* `--staged` / `--cached`: staged changes that will go into last commit 

#### git log
* no arguments: display history only with commit msg and hash in reverse chronological order 
* `--graph`: display branch and merge history in graph (tree)
* `--patch / -p`: display history with file change summary
* `-{number}`: display `number` logs
* `--stat`: display history with brief summary
* `--pretty={format}`: pre built-in format or customized format defined by us 

#### git tag 
* no arguments: list specific points in a repo's history 
* `-l <tag-sub-name>`: filter according to tag name 


#### git show 
* `<tag-name>`: show tag with commits 



### Error handling (undo after abnormal execution)
#### When executing such commands, be careful since not all the undo operations can be undoed again 
It is quite annoying that sometimes you do the wrong operation and your local development branch becomes messy. Even worse your changes have been pushed to remote branch. The worst is your changes have been merged into mainline (If you wanna change something, it may pollute others' work). 

#### git commit
* `--amend`: undo the last commit, can add forgotten file or modify commit message 


#### git reset (this could be a dangerous command with some arguments like `--hard` since you probably lose your work)
* `HEAD <file-name>`: unstage a staged file 
* `--soft <commit-hash>`: change head pointer and put previous changes into staged area
* `--mixed <commit-hash>`: the default mode, change head pointer and put previous changes into unstaged area
* `--hard <commit-hash>`: change head pointer and remove all the uncommitted changes 

For `git reset` details, can refer to this [question](https://stackoverflow.com/questions/3528245/whats-the-difference-between-git-reset-mixed-soft-and-hard) for further understanding. 

#### git revert 
* `<commit-hash>`: revert a commit and generate a commit about this reverting operation 

#### git checkout 
* `-- <file-name> / <file-name>`: discard changes that you made to the file <== could be dangerous


### Work with remotes (Fetching and Uploading)
#### git clone 
* no arguments: clone all the content in remote and add remote branch implicitly
* `-o <remote-name>`: default remote name is `origin`, can change to other names during clone process

#### git remote 
* no arguments: check upstreaming branch 
* `-v`: show full url of upstreaming branch(push and pull)
* `add <short-name> <url>`: add upstreaming branch explicitly 
* `rename <old-remote-name> <new-remote-name>`
* `remove <short-name>`: delete remote reference, all remote branches are also deleted 
* `show origin`: remote branches

#### git fetch 
* no arguments: fetch all the changes on the server, this command will not modify anything in local, just be ready for you to operate
* `<remote-name>`: fetch all info from remote, for example, `git fetch origin`


#### git pull == git fetch + git merge from upstreaming branch 
* no arguments: fetch commits in upstreaming branch and merge into local branch 
* `--rebase`: == `git fetch` + `git rebase` in this case 

#### git push
* no arguments: push to the default remote branch 
* `<remote-name> <branch-name>`: push to remote branch if local branch name and remote branch name are the same
* `<remote-name> <local-branch-name>:<remote-branch-name>`: push to remote branch if local branch name and remote branch name are different 
* `<remote-name> --delete <remote-branch-name>`: delete remote branch, remove pointer from server
* `--force`: rewrite upstreaming branch history according to local history

#### git checkout 
* `-b <local-branch-name>`: create a new branch and switch to it == git branch <branch-name> + git checkout <branch-name>
* `-b <local-branch-name> <remote-name>/<remote-branch-name>`: create a local branch that sync with remote branch <remote-branch-name>
* `--track <remote-name>/<remote-branch-name>`: set up current local branch to track remote branch with branch name <remote-branch-name>

<br>
`git merge` and `git rebase` are two ways to integrate changes from other branches or local to current branch while `rebase` maintains a cleaner history. However, it is dangerous to use `git rebase` blindly without fully understand the effect of this command. 
According to Git book, 

> Do not rebase commits that exist outside your repository and people may have based work on them.

Rebasing changes the commit history and if other people already merged the older version of commit history and you push newer version, it will become messy. 

#### git merge 
* `<branch-name>`: merge another branch into current branch
    - fast-forward: fast-forward simply means move the HEAD pointer forward, Git regards it as a linear history, no diverging and merging required 
* `--no-ff <branch-name>`: merge without fast forward, will always generate a merge commit even if it is fast forward merge 

#### git rebase 
* `<branch-name>`: copy commits and merge another branch into current branch, can rebase on top of force-pushed rebase work
* `<branch-a-name> <branch-b-name>`: make `branch-b` replay changes in `branch-a` 
* `--onto <branch-a-name> <branch-b-name> <branch-c-name>`: take commits from `branch-c` diverged from `branch-b` and apply direcly to `branch-a` (a bit tricky, I don't like this command)

Usually public branches use `git merge` while personal branch use `git rebase`. Rebase will make commit history cleaner by rewriting the whole commit history. However, it will pollute other people's history if you rebase + push. merge will leave a merge commit in the history. 

For public branch that someone already rebased his/her commits into it and we already pulled before, use `git rebase` to merge remote branch code into your branch rather than `git merge` to avoid duplicate commits. 

#### git branch 
* no arguments: list all the local branches 
* `-r`: list all the remote branches 
* `-d <branch-name>`: delete branch 
* `-D <branch-name>`: force delete branch (may lose changes that haven't commited into stable branches)
* `-v`: see the last commit of each local branch 
* `-vv`: list all local branches with tracking branch and `head` pointer info
* `--no-merged`: see the branches that contain unmerged commits 
* `-u / --set-upstream`: set upsteam branch 


## Git branching 
According to Git Book 

> Branching means you diverge from the main line of development and continue to do work without messing with that main line. 

The statement from Git Book states the core concept of branching, which is to take care of personal development while still maintain the easy-to-merge connection with main line. The core of `Git branching` is actually diverge and merge. To find a way to diverge and merge conveniently and lightly, Linus Trovalds use concept of pointers to make it work. 

alias of git


## Git implementation 
Git stores its data as a sequence of snapchats of the whole filesystem. It does not copy the whole files but stores the reference of each file whenever user commits something. So the whole git history is like a stream of snapshots. 

There are three types of files in git file system, `commit`, `tree` and `blob`. `Branch` is actually a pointer to different commits and moving the HEAD of each branch is actually changing the position of pointer, which is relatively lightweight and convenient compared to other VCS. `branch` is a file that contains the 40 character SHA-1 checksum of the commit that it points to. 

`fast-forward` is a concept that applys when integrating changes. Simply treat it as linear merge, do diverging between branches. 


// When git communicate between local repo and remote repo(GitHub or Gitlab), there are two protocols that it can use. `https` and `ssh`. 

There are 4 protocols used to transfer data. `Local`, `http`, `ssh` and `Git`. 


One more thing: Comparison of Gitlab and GitHub 

If you got any questions, feel free to contact me via any communication tools 

Reference: 
1. [Git book](https://git-scm.com/book/en/v2)
2. [Atlassian Git tutorial](https://www.atlassian.com/git/tutorials)
3. [Git full reference](https://git-scm.com/docs)
4. [Stackoverflow git reset question](https://stackoverflow.com/questions/3528245/whats-the-difference-between-git-reset-mixed-soft-and-hard)
