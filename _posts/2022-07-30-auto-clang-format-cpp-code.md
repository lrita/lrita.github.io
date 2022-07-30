---
layout: post
title: 使用clang-format在CI中自动格式化C++代码
categories: [c++]
description: c++
keywords: c++ clang-format
---

一个几十人同时参与开发的C++项目，每周大约合并PR40-60个，人员水平参差不齐，~~一些人写了好几年，连IDE都不会配置，完全当文本编辑器在用~~，一些基本要求反复强调但是仍然很难达成一致。比如代码格式的问题，实在令人头疼。所以最好还是能够自动化解决比较好。

- 首先是确定使用`clang-format`来格式化代码，把格式上的设置写入配置文件`.clang-format`放在项目根目录上，然后安装`clang-format`和配合git的脚本`git-clang-format`。
- 其次是每次只更新新增代码，避免比必要的更新历史代码记录
- 然后就是如何只更新新增代码，其具体步骤如下

```sh
# 适用于macOS手动执行
git co xx_squash
git reset --soft master
git clang-format --style=file
git co xx
git add -u
git commit -m "xx"
git push
git b -D xx_squash
```

在macOS和linux具体步骤还不一样，可能是2个环境安装的`git-clang-format`版本不同。以下命令可以写成一个脚本，放在CI里自动化执行。
```sh
#!/bin/bash
# 适用于linux
M_SHA1=$(git rev-parse origin/master)
git clang-format --style file "$M_SHA1"
if [[ "$(git status --short | wc -l)" == "0" ]]; then
  exit 0
fi
git add -u
git commit -m "格式化代码"
git push "https://${CI_PRIVATE_USER}:${CI_PRIVATE_TOKEN}@${CI_REPOSITORY_URL#*@}" "HEAD:${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME}"
# 其中 CI_PRIVATE_USER，CI_PRIVATE_TOKEN 用于push代码的权限，可以在CI自定义变量中配置。
```