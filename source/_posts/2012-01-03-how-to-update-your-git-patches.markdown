---
layout: post
title: 'Resolved: 如何更新 git 补丁'
categories:
- Z-Turn
tags: []
published: true
comments: true
---
使用 git 制作补丁时，经常发现补丁需要修改。如果只是最后一次 commit 需要修改，那就好办，用下面的方法就可以搞定：

``` bash
git reset HEAD^
# edit edit edit
git commit -a -s -c ORIG_HEAD
git format-patch --subject-prefix="PATCH v2"
```

但是如果是一系列补丁中的中间几个补丁需要修改，该怎么办呢？

<del datetime="2012-01-03T16:12:57+00:00">笨办法已经被删掉>.<</del>

<!--more-->
**Update 1:** 非常感谢 wangcong 的指点！`git rebase -i` 可以出色地完成这个任务，方法在 man-page 里面有详述，见 ___INTERACTIVE MODE___ 和 ___SPLITTING COMMITS___ 部分。

假设当前所在分支名为 __topic__ ，做了3个补丁，打算提交给邮件列表的时候发现中间一个补丁需要更新，此时使用下列命令：

``` bash
git rebase -i HEAD~4
```

出现编辑窗口如下：

```
pick 21732a8 mm/swapping01: new testcase
pick c3751d1 hugemmap01: clean-up format
pick d75449c mm/swapping01: change sleep time to 5 sec
```

此时需要修改 `commit c3751d1`，则把 `pick` 改为 `edit` 即可。保存退出后 git 会 rebase 到需要修改的 commit 处停止，此时可以直接修改内容，然后执行：

``` bash
git add xxx # mark as resolved
git commit --amend
```

或者

```
git reset --soft HEAD^
#edit edit edit
git commit -a -s -c ORIG_HEAD
```

之后执行 `git rebase --continue` 即可。

突然想起来一年前就请教过 wangcong 这个问题，只不过当时对 `git rebase` 完全不了解，后来就忘了，惭愧 -_-|||。

**Update 2**: 同样感谢 Iven 的指点，可以去看看他的文章: [http://www.kissuki.com/?p=135](http://www.kissuki.com/?p=135)
