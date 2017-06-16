---
layout: post
title: 使得git workflow条理清晰
categories: [git]
description: git gitflow
keywords: git gitflow
---

## 选择git workflow

`git`不仅仅是一个工具，也是一门学问。作为一个程序员，当然不应该仅仅是会用`git`而已，还要研究其中的门门道道。
当然，“会用`git`”这个门槛也不低啊。

很多程序员用了很多年的`git`，然而对`git`的了解并不深入，会用的操作也是简单的几条，只是将3天速成的`git`经验
使用了很多年而已。

`gitflow`就是社区中逐渐总结形成的`git`哲学，最近看了篇[文章](https://github.com/oldratlee/translations/blob/master/git-workflows-and-tutorials/README.md)，
总结了几种常见的`gitflow`的模式和演示。

主要分为：
* [集中式工作流](https://github.com/oldratlee/translations/blob/master/git-workflows-and-tutorials/workflow-centralized.md)
* [功能分支工作流](https://github.com/oldratlee/translations/blob/master/git-workflows-and-tutorials/workflow-feature-branch.md)
* [Gitflow工作流](https://github.com/oldratlee/translations/blob/master/git-workflows-and-tutorials/workflow-gitflow.md)
* [Forking工作流](https://github.com/oldratlee/translations/blob/master/git-workflows-and-tutorials/workflow-forking.md)
* [Pull Request工作流](https://github.com/oldratlee/translations/blob/master/git-workflows-and-tutorials/pull-request.md)

个人认为比较适合企业内部开发团队的方式是：

1. 在项目初期，模块结构、实现依赖都还不十分稳定时采用`集中式工作流`，全部人在一起快速搭建起项目的框架，划分好模块，
将各个模块的边界迅速划清。
2. 当框架稳定后，进入`功能分支工作流`，这样能够保证主分支的持续稳定。
3. 如果团队内部有良好的code review习惯的，可以进入`Pull Request工作流`(在`gitlab`上称为`merge request`)。
4. 商业项目可以进入`Gitflow工作流`，但是个人认为这个比较重，不太适合互联网公司的敏捷形式，有些互联网公司内部
项目不太会有`release`这个动作存在。

## 良好的commit习惯 
当然选好`workflow`只是开始的第一步，要使得项目`workflow`清晰，还要依赖参与的每一个的努力。你可以找一些公司内部
的项目，进入执行`git log --oneline --decorate --graph`。你可能会发现一些项目的`workflow`杂乱无章，惨不忍睹。

比较常见的问题有：
* 毫无意义的commit message，你从这个commit message中根本看不出他要做什么，当有一个bug code 新增时，你很难通过
提交历史来判断大概是哪些提交可能会导致这个bug。我见过最夸张的是，修改了500行代码，commit message只有一个"a"，
`韩红看了想打人.jpg`。这点可以参考[Commit messages are not titles](http://antirez.com/news/90)
* 编译不过的commit，不要以为只有新人会这样，很多老手也这样。主要是个人责任心不强或者认为`git commit`就是起一个
`save`的意义，这是大促特错的想法，`git commit`应该保持一定的语义，表明你完成了某件事，当你手中有WIP的代码，需要
临时切换上下文去做另一件事时，应该使用`git stash`或者diff+patch等方式来解决。
* 遗漏的文件，责任心不强的表现，每次commit前先`git status`检查一下就能避免的问题。
* 不必要的merge，这个主要是使用者知道的git操作比较少，完全可以使用`git rebase`命令来避免，可以参照[集中式工作流](https://github.com/oldratlee/translations/blob/master/git-workflows-and-tutorials/workflow-centralized.md)
中的用法。

## 清理聚合分支上的commit
当你一不小心犯了创造了很多垃圾commit时，还是有很多方法进行挽救了。一下就列出一些场景。

_注意：以下场景均适用于还未`git push`之前的挽救，当已经push到远程分支时，只能采取一些分支互换的方式，或者
`git push -f`的方式，不太推荐强制推送的方式。_

### 修改刚提交的commit

当你刚commit后就发现有文件遗漏了或者缺失几行代码了，当你还未`git push`时应该：
```shell
git add lostfile              # 添加遗漏的文件，或者添加新修改的代码文件
git commit --amend -m "xxxxx" # 修正上次commit，这次commit会和上一次的commit合并成一个commit
```

或者

```shell
git reset --mix HEAD^ # 将上次commit退回暂存区
git status            # 你会看到你上次的commit的内容退回到了暂存区
xxxx                  # 进行一些修复
git add xxxx
git commit -m "xxxx"  # 重新提交
```

### 修改非刚提交的commit
当你连续几个commit时，发现其中有一个commit有问题时，但是这个commit不是最后一个commit，不好使用上面的方法。
此时可以：

```shell
git log             # 查看出问题的commit sha1值，比如有问题的sha1为2a99858ebd0127049750eef9a9573fd9650aed41
                    # 该commit的前一个sha1为070aa8b810337dbb6f38cc354d7378f16c603c3c
xxxxxx              # 做一些修复
git add             # 添加修改
git commit --squash fec308ab5ac31ccb502f8b4e751eaa2f91d0939e  # 此时会产生一个message为"squash!"开头的commit
git rebase --autosquash 070aa8b810337dbb6f38cc354d7378f16c603c3c # 此时会弹出编辑框，此时只要save即可，当然这
                                                                 # 个编辑框还有很多其他作用，可以参考后面的附录
git log             # 此时可以看到message为"squash!"开头的commit已经与之前有问题的commit合并了。
```

## 参考
* [Keep your branch clean](http://fle.github.io/git-tip-keep-your-branch-clean-with-fixup-and-autosquash.html)
