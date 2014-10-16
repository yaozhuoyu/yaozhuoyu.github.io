---
layout: post
title: "关于git push的时候提示warning: push.default is unset的问题"
date: 2013-10-23 13:40
comments: true
categories: git
tags: git
---
今天在学习git的时候，使用了git push命令，git提示了一段警告，如下：
```
warning: push.default is unset; its implicit value is changing in
Git 2.0 from 'matching' to 'simple'. To squelch this message
and maintain the current behavior after the default changes, use:

  git config --global push.default matching

To squelch this message and adopt the new behavior now, use:

  git config --global push.default simple

See 'git help config' and search for 'push.default' for further information.
(the 'simple' mode was introduced in Git 1.7.11. Use the similar mode
'current' instead of 'simple' if you sometimes use older versions of Git)

Everything up-to-date
```
然后决定查个究竟。
<!--more-->

最后找到了原因所在。我们知道，当我们使用git push命令的时候，意思就把所有本地的local branches push到remote相同名字的branch上，此中方式称为matching。

而另一种方式称为simple，此种方式只会把当前的分支push到远端对应的分支(此远端分支就是调用git pull可以获取内容的分支)。

显然simple方式是比较好的，但是git 2.0版本之前默认为matching，git在2.0版本开始就改为simple了。

所以为了解决此waring,我们只需输入命令`git config --global push.default simple`，回车就行。



