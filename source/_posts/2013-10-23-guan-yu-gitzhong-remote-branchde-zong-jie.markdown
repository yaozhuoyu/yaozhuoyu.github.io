---
layout: post
title: "关于git中remote branch的总结"
date: 2013-10-23 17:31
comments: true
categories: git
tags: git
---
这是一篇关于git remote branch的总结，其中讲述了和remote branch相关的内容，包括创建，删除等等。
<!--more-->
##1.什么是远程分支
远程分支（remote branch）是对远程仓库中的分支的索引。它们是一些无法移动的本地分支；只有在 Git 进行网络交互时才会更新。远程分支就像是书签，提醒着你上次连接远程仓库时上面各分支的位置。一般我们使用`(远程仓库名)/(分支名)` 这样的形式表示远程分支。更多的概念不如例子直接，下面还主要按例子来讲述。
下面以github上面的AFNetworking库为例子，首先到网址 [link](https://github.com/AFNetworking/AFNetworking) fork一份，这样自己就有一份了，git地址为 https://github.com/yourname/AFNetworking.git ，打开命令行git clone https://github.com/yourname/AFNetworking.git ，这样在本地就有一份你自己啦。
执行命令`git branch -a`，查看所有分支：
``` bash
* master
  remotes/origin/0.10.x
  remotes/origin/1.x
  remotes/origin/HEAD -> origin/master
  remotes/origin/assets
  remotes/origin/master
```
可以看到有5个分支，4个远程分支，origin/0.10.x,origin/1.x,origin/assets,origin/master，本地分支为master，也是当前分支。

使用`git branch -vv`命令，此命令会列出本地分支最近的一次提交，以及跟踪的远程分支，得到结果如下：
``` bash
* master e2d5bb0 [origin/master] Merge pull request #1485 from brunokoga/master
```
可以看到，本地分支跟踪了远程分支origin/master。
查看更详细的信息可以通过命令：`git remote show origin`，结果如下：
```bash
* remote origin
  Fetch URL: https://github.com/yourname/AFNetworking.git
  Push  URL: https://github.com/yourname/AFNetworking.git
  HEAD branch: master
  Remote branches:
    0.10.x tracked
    1.x    tracked
    assets tracked
    master tracked
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (up to date)
```
其实本地分支master 跟踪 origin/master是在我们clone 的时候自动帮我们设置好的。
当我们git clone https://github.com/yourname/AFNetworking.git 的时候，git会自动为我们将此远程仓库命名为 origin，并下载其中所有的数据，建立一个指向它的master分支的指针，在本地命名为 origin/master。但无法在本地更改其数据。接着，git为我们建立一个属于自己的本地master分支，始于origin上master分支相同的位置，可以就此开始工作。

#2.创建跟踪分支
如何我们想在本地创建一个分支，去track origin/1.x分支，则使用如下命令：`git checkout -b local1x origin/1.x`，这会切换到新建的 local1x 本地分支，其内容同远程分支 origin/1.x 一致，这样你就可以在里面继续开发了。像local1x这种远程分支 checkout 出来的本地分支，称为 跟踪分支 (tracking branch)。跟踪分支是一种和某个远程分支有直接联系的本地分支。

我们在运行命令`git remote show origin`看看：
```bash
* remote origin
  Fetch URL: https://github.com/yourname/AFNetworking.git
  Push  URL: https://github.com/yourname/AFNetworking.git
  HEAD branch: master
  Remote branches:
    0.10.x tracked
    1.x    tracked
    assets tracked
    master tracked
  Local branches configured for 'git pull':
    local1x merges with remote 1.x
    master  merges with remote master
  Local ref configured for 'git push':
    master pushes to master (up to date)
```
可以看到`Local branches configured for 'git pull': local1x merges with remote 1.x`,意思就是在local1x分支下运行`git pull`命令，可以直接合并远程的分支1.x，

```bash
MacBook-Pro:AFNetworking $ git branch
* local1x
  master
MacBook-Pro:AFNetworking $ git pull
Already up-to-date.
```
但是如果我们运行git push呢？
```bash
MacBook-Pro:AFNetworking $ git push
fatal: The upstream branch of your current branch does not match
the name of your current branch.  To push to the upstream branch
on the remote, use

    git push origin HEAD:1.x

To push to the branch of the same name on the remote, use

    git push origin local1x
```
提示已经很清楚了，因为刚刚建立跟踪分支的时候，名字不一样，所以在此push的时候要写上分支名字。

如果我们执行命令:`git push origin local1x`,结果如下：
```bash
MacBook-Pro:AFNetworking $ git push origin local1x
Total 0 (delta 0), reused 0 (delta 0)
To https://github.com/yourname/AFNetworking.git
 * [new branch]      local1x -> local1x
```
竟然给我创建在远端创建了一个新的local1x分支，赶快运行一下命令`git remote show origin`看看。
```bash
yaozhuoyumatoMacBook-Pro:AFNetworking yzy$ git remote show origin
* remote origin
  Fetch URL: https://github.com/yourname/AFNetworking.git
  Push  URL: https://github.com/yourname/AFNetworking.git
  HEAD branch: master
  Remote branches:
    0.10.x  tracked
    1.x     tracked
    assets  tracked
    local1x tracked
    master  tracked
  Local branches configured for 'git pull':
    local1x merges with remote 1.x
    master  merges with remote master
  Local refs configured for 'git push':
    local1x pushes to local1x (up to date)
    master  pushes to master  (up to date)
```
现在才明白，当我们运行命令`git push origin local1x`，会去检查服务器有没有local1x分支，没有就会创建一个新的，并在我们下次在此branch运行git push的时候，会自动的push到远程分支。

因此，创建远程分支的命令为: `git push origin 远程分支名`

现在我们看到当在local1x分支下，运行git pull 命令，拉取的为远程分支1.x上面的数据，而运行git push命令，是推送到远程分支local1x上，为了一致起见，我们可以重新设置git pull默认拉取的远程分支，`git branch --set-upstream-to=origin/local1x local1x`

```bash
MacBook-Pro:AFNetworking $git branch --set-upstream-to=origin/local1x local1x
Branch local1x set up to track remote branch local1x from origin.
```

大概理解了吧？
因此，我们在创建跟踪分支的时候，最好其的名字和远程分支一样，则这样git push和git pull会默认给我们设置好。

运行命令`git checkout -b 1.x origin/1.x`，如下：
```bash
MacBook-Pro:AFNetworking $ git remote show origin
* remote origin
  Fetch URL: https://github.com/yourname/AFNetworking.git
  Push  URL: https://github.com/yourname/AFNetworking.git
  HEAD branch: master
  Remote branches:
    0.10.x  tracked
    1.x     tracked
    assets  tracked
    local1x tracked
    master  tracked
  Local branches configured for 'git pull':
    1.x     merges with remote 1.x
    local1x merges with remote local1x
    master  merges with remote master
  Local refs configured for 'git push':
    1.x     pushes to 1.x     (up to date)
    local1x pushes to local1x (up to date)
    master  pushes to master  (up to date)
```
要注意：`git checkout -b 1.x origin/1.x`和命令`git branch --track  1.x origin/1.x`是等价的，可以相互替换。

##3.删除远程分支
命令 `git push origin :local1x`删除远程分支local1x，如下
```bash
MacBook-Pro:AFNetworking $ git push origin :local1x
To https://github.com/yourname/AFNetworking.git
 - [deleted]         local1x
```
远程分支大概常用的操作就这么多，以后遇到在补充吧。


