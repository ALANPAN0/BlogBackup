---
title: git命令
date: 2017-02-25 17:33:33
tags: [GIT]
---

常用git命令梳理篇
<!--more-->


> 阮一峰这篇[常用命令清单][1]清晰的总结了常用的命令，这边我就不在赘述了，这边主要补充并整理个人的一些命令

### 清单

`$ git commit --amend -m [message]`
 \# 使用一次新的commit，代替上一次的提交
\# **如果代码没有任何变化，则用来改写上一次commit的提交信息**

`$ git commit --amend [file1] [file2] ...`
 \# 重新做上次的commit， 并包括指定文件的新变化

`$ git branch [branch] [commit]`
\# 新建一个分支，指向指定commit

`$ git branch --set-upstream [branch] [remote-branch]`
\# 建立追踪关系，在现有分支与指定的远程分支之间

`$ git cherry-pick [commit]`
\# 选择一个commit，合并进当前分支

`$ git push -u [remote] [branch]`
\# 推送到remote仓库下的branch分支，并当前分支tracking到remote仓库下的branch分支

`$ git push <remote> <local branch name>:<remote branch to push into>`
\# 上传本地的分支到远程的分支

`$ git reflog/git log -g`
\# 查看所有的历史操作，这个命名比较实用。比如新来的同事经常误操作git导致不想发生的结果，这个时候你可以使用git reflog查看其进行的所有操作，然后进行reset

### 常见问题

- git每次要求你输入密码，Enter passphrase for key '/Users/XY/.ssh/id\_rsa':？  
	直接敲`ssh-add`然后输入密码即可， 详情[点击][2]


- git rebase 冲突?
	1. 解决一个补丁的应用冲突后，标记冲突已解决 git add -u
	2. 继续rebase   git rebase —continue
	3. push到远端就ok了，add后不需要commit
	4. git rebase —about 放弃rebase  
		git rebase —skip   忽略本地冲突的补丁


- git fetch 和 git pull 区别？  
	pull = fetch + merge，git pull 相当于 git fetch 加上一个git merge的操作  
	git fetch 创建并更新所有远端分支的本地远端分支，创建远端分支时，会自动获取新加入的分支，这个操作并不会改变本地的工作区，详情[点击][3]

- git revert 和 git reset 区别？  
	revert：通过创建一次新的 commit 来撤销一次 commit 所做出的修改。这种撤销的方式是安全的，因为它并不修改 commit history  
	  
	reset：还原index的状态或者修改本地分支HEAD的位置。比如， 某个提交之后的代码都不要了，就可以在本地直接reset至指定的commit  
	补充：  
	 `git reset --soft`  
	\# staged snapshot 和 working directory 都未被改变  
	`git reset --mixed`  
	\# staged snapshot 被更新， working directory 未被更改  
	`git reset --hard`  
	\# staged snapshot 和 working directory 都将回退  
	这些标记经常和HEAD一起使用。例如，git reset --mixed HEAD可撤销所有缓存改动，但是保留他们在工作目录下。git reset --hard HEAD可彻底删除没有提交的改动  


### 盗用一下😂  
![][image-1]

[1]:	http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html
[2]:	http://unix.stackexchange.com/questions/12195/how-to-avoid-being-asked-passphrase-each-tim%E2%80%A6
[3]:	http://stackoverflow.com/questions/292357/what-is-the-difference-between-git-pull-and-git-fetch

[image-1]:	http://7xq5ax.com1.z0.glb.clouddn.com/e55477eejw1f07e4ffienj21cv2funfq.jpg