---
layout: post
title: 使用meld作为git的辅助工具
categories: [git, meld]
description: meld git
keywords: meld, git
---

_`meld`的使用说明可以搜到很多，但是其中说法有所差别。特此重新撰写一份，使用的meld版本为3.16.0_

## 为什么使用meld
当同一个文件被多人编辑时，经常会出现冲突的问题，因此就需要解决冲突。

但是git内置的diff、merge工具比较孱弱，经常会发生一些问题，例如

#### 删除的代码被人合并时又加了回来
删除的代码被人合并时又加了回来，我想这种场景使用git的团队都遇见过。如果出现这种问题时，
解决冲突的人又是一个粗心的家伙，同时代码有没编译出错，则很难再发现产生了这个异常。只有
当bug再次出现时，才会发现“这个bug我明明修复了，怎么代码又回滚了？”。

下面详细说明一下这个问题出现的原因。

当git上原始代码为:
```
conflict begin
Hello world!!!
Here is a misuse code. 
conflict end
```

当用户A、B同时下载该代码同时开发时。

用户A发现了一个bug，将bug移除，提交代码，代码变更为:
```
conflict begin
Hello world!!!
conflict end
```

同时用户B添加了一个新的功能，代码变更为:
```
conflict begin
Hello world!!!
Here is a misuse code. 
Here is a new feature.
conflict end
```

然后B提交代码时会产生一个冲突，需要解决，则git默认的merge工具则显示冲突代码为:
```
conflict begin
Hello world!!!
<<<<<<< HEAD
Here is a misuse code.
Here is a new feature.
=======
>>>>>>> e84872b222b7a9d8a3e8745ea3c9a3e85237503c
conflict end
```

然而用户B可能对git的merge显示并不熟悉、或者B是一个粗心的人。此时用户B看到`<<<<<<<`
和`=======`之间是自己本地提交的新功能代码，则B将merge合并提示信息删除后，代码变为:

```
conflict begin
Hello world!!!
Here is a misuse code.
Here is a new feature.
conflict end
```

然后B将代码提交，此时可见，被删除的bug代码又回来了，如果此时没有人专门review代码，
大家并没有感觉到上面不对，只有在再次出现bug时才会发现该问题。

## 三路合并工具
最容易避免以上问题出现的方式就是采用三路合并工具。场景的三路合并工具有`kdiff3`、
`meld`、`Beyond Compare`等。在此推荐`meld`，该工具具有良好的跨平台能力，无论你使用
`Window x`/`Linux`/`Mac OS`都可以很容易的一键安装该软件，同时免费使用，而且启动速度
快于`Beyond Compare`等等收费软件。

#### mac 安装 meld
执行以下命令一键安装
```
brew install caskroom/cask/meld
```

#### git 配置
修改本地的`~/.gitconfig`配置文件，加入以下几行配置
```
[merge]
        tool = meld
	conflictstyle = diff3
[mergetool "meld"]
        cmd = meld $LOCAL $BASE $REMOTE --output=$MERGED --auto-merge
```

当用户运行`git pull`或者`git pull --rebase`产生冲突时，执行`git mergetool`就会弹出`meld`
的三路合并界面:
![meld-3-way-merge](/images/posts/tools/meld-3-way-merge.png)

可见界面一共分为3栏，左边一栏为你本地当前文件内容，右面一栏为远端服务器上当前文件内容。
中一栏为本地当前文件原始内容。可见中间一栏meld已经帮你自动合并了，检查无误就可以直接保存
退出了。然后meld帮你自动保存了一份git原始冲突文件`xxx.orig`，检查无误可以删除此文件。

注：当不同行冲突时，meld会帮你自动合并，当同一行冲突时，meld会以git 冲突文件的格式显示在
中间栏，此时你比较判断`<<<<<<< HEAD`/`=========`/`>>>>>>>>`3行直接的差异手动合并即可。
