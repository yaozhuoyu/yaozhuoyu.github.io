---
layout: post
title: "git如何解决冲突"
date: 2013-10-22 16:00
comments: true
categories: git
tags: git
---

因为开发中代码管理使用的为svn，所以只有在玩github的时候才会去使用git。但是发现自己很多git命令还是不很属性，所以决定记下来方便总结。这篇文章主讲在git合并branch的时候如何解决冲突。
<!--more-->

单讲命令不容易理解，本文根据实际例子讲述。

##1.setting up the repository
下面例子将创建一个本地的repository，创建一个简单的文件README,并向里面添加内容，提交。
``` bash
mkdir conflict-sandbox
cd conflict-sandbox
git init
# Initialized empty Git repository in /Users/tekkub/tmp/conflict-sandbox/.git/

echo "rawr" > README.md
git add README.md

git commit -m 'Initial commit'
# [master (root-commit) ff50266] Initial commit
#  1 file changed, 1 insertion(+)
#  create mode 100644 README.md
```

##2. creating an edition collision

创建一个branch，并向README文件中添加内容，commit。
``` bash
git checkout -b branch-a
# Switched to a new branch 'branch-a'

echo "changed in branch A" >> README.md

git add README.md

git commit -m 'changed README for branch A'
# [branch-a a8280d5] changed README for branch A
#  1 file changed, 1 insertion(+)
```

现在在基于master创建一个新的branch，在添加一行，commit。然后合并branch-a的时候就出现冲突了。
``` bash
git checkout -b branch-b master
# Switched to a new branch 'branch-b'

echo "changed in branch B" >> README.md

git commit -m 'changed README for branch B'
# [branch-b 06bc5e8] changed README for branch B
#  1 file changed, 1 insertion(+)

git merge branch-a
# Auto-merging README.md
# CONFLICT (content): Merge conflict in README.md
# Automatic merge failed; fix conflicts and then commit the result.
```
运行`git status`，可以看到冲突
``` bash
git status
# # On branch branch-b
# # You have unmerged paths.
# #   (fix conflicts and run "git commit")
# #
# # Unmerged paths:
# #   (use "git add ..." to mark resolution)
# #
# # both modified:      README.md
# #
# no changes added to commit (use "git add" and/or "git commit -a")
```
打开文件README，可以看到如下：
```
rawr
<<<<<<< HEAD
changed in branch B
=======
changed in branch A
>>>>>>> branch-a
```
然后按需求解决冲突，我们可以这样解决：
```
rawr
changed in both branches
```
解决之后commit就可以了。
``` bash
git add README.md

git commit
# [branch-b a99bbcd] Merge branch 'branch-a' into branch-b

git show | head
# commit a99bbcd8f212d490e9371b59522398eec25d75d1
# Merge: 06bc5e8 a8280d5
# Author: tekkub 
# Date:   Sat Jun 1 18:11:04 2013 -0700

#     Merge branch 'branch-a' into branch-b
# 
#     Conflicts:
#         README.md
```

##3. resolving a removed file conflict
有另外一种常见的冲突，就是在一个branch中，我们编辑了一个文件，但是在另外一个branch中，我们将此删除。当合并的时候，git不知道我们是想保持此文件为最新编辑的，还是想要删除此文件。下面例子将按此两种情况分别解决冲突。

(1)按第一种情况，我们想要保持此文件为最新编辑的。
在README文件中添加一行，commit。
``` bash
echo 'an edit we want to keep' >> README.md

git add README.md

git commit -m 'another commit on branch b'
# [branch-b 7e8b679] another commit on branch b
#  1 file changed, 1 insertion(+)
```

在master的基础上在创建一个新的分支c，删除文件，提交。
``` bash
git checkout -b branch-c master
# Switched to a new branch 'branch-c'

git rm README.md
# rm 'README.md'

ls
# [no output]

git commit -m 'removing the README'
# [branch-c 4c80a63] removing the README
#  1 file changed, 1 deletion(-)
#  delete mode 100644 README.md
```

现在开始merge，git会重新创建此文件。
``` bash
git merge branch-b
# CONFLICT (modify/delete): README.md deleted in HEAD and modified in branch-b. Version branch-b of README.md left in tree.
# Automatic merge failed; fix conflicts and then commit the result.

git status
# # On branch branch-c
# # You have unmerged paths.
# #   (fix conflicts and run "git commit")
# #
# # Unmerged paths:
# #   (use "git add/rm ..." as appropriate to mark resolution)
# #
# # deleted by us:      README.md
# #
# no changes added to commit (use "git add" and/or "git commit -a")

ls
# README.md
```
通过重新添加，commit，解决冲突
``` bash
git add README.md

git commit
# [branch-c 9bc3b01] Merge branch 'branch-b' into branch-c

git show | head
# commit 9bc3b0130fe0178359d51243b5b882076a12f554
# Merge: 4c80a63 7e8b679
# Author: tekkub :bear:  
# Date:   Sat Jun 1 18:39:40 2013 -0700

#     Merge branch 'branch-b' into branch-c
# 
#     Conflicts:
#         README.md
```

(2)按第二种情况，通过删除file来解决冲突。

``` bash
echo 'an edit we do not want to keep' >> README.md

git add README.md

git commit -m 'edited on branch C'
# [branch-c fcc1093] edited on branch C
#  1 file changed, 1 insertion(+)

git checkout -b branch-d master
# Switched to a new branch 'branch-d'

git rm README.md
# rm 'README.md'

git commit -m 'removing the README again'
# [branch-d 211261b] removing the README again
#  1 file changed, 1 deletion(-)
#  delete mode 100644 README.md
```
合并分支c：
``` bash
git merge branch-c
# CONFLICT (modify/delete): README.md deleted in HEAD and modified in branch-c. Version branch-c of README.md left in tree.
# Automatic merge failed; fix conflicts and then commit the result.

git status
# # On branch branch-d
# # You have unmerged paths.
# #   (fix conflicts and run "git commit")
# #
# # Unmerged paths:
# #   (use "git add/rm ..." as appropriate to mark resolution)
# #
# #   deleted by us:      README.md
# #
# no changes added to commit (use "git add" and/or "git commit -a")
```
现在我们想要删除文件，通过命令`git rm`，在通过ls确认。
``` bash
git rm README.md
# README.md: needs merge
# rm 'README.md'

ls
# [no output]
```
最后，commit。
```
git commit
# [branch-d 6f89e49] Merge branch 'branch-c' into branch-d

git show | head
# commit 6f89e49189ba3a2b7440fc434f351cb041b3999e
# Merge: 211261b fcc1093
# Author: tekkub :bear:  
# Date:   Sat Jun 1 18:43:01 2013 -0700

#     Merge branch 'branch-c' into branch-d
# 
#     Conflicts:
#         README.md
```
本文摘抄自github官网的帮助文档，[地址>>>>>>](https://help.github.com/articles/resolving-a-merge-conflict-from-the-command-line)
