---
layout: wiki
title: Mac安装node.js
categories: node.js
description: Mac安装node.js。
keywords: Mac node.js
---

强烈推荐使用`nvm`管理`Node.js`。它可以对`Node.js`进行多版本的管理。

```sh
# 安装nvm
$> brew install nvm

# 创建nvm缓存
$> mkdir ~/.nvm

# 将nvm配置加入zsh
export NVM_DIR="$HOME/.nvm"
source /usr/local/opt/nvm/nvm.sh

# 试验命令
$> nvm --version

# 查看可选node.js版本
$> nvm ls-remote --lts

# 安装指定node.js版本
$> nvm install v8.9.4
```
