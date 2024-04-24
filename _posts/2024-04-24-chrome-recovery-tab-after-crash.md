---
layout: post
title: 在宕机后修复丢失的Chrome中打开的页面
categories: [tool]
description: tool chrome recovery miss tab
keywords: tool chrome recovery miss tab
---

当机器宕机以后，重新打开Chrome浏览器时，会提示重新打开`最近关闭的标签页`，从而恢复之前的记录，或者可以通过按快捷键`Cmd + Shift + T (Mac)`/`Control + Shift + T (Windows)`来进行。

但是有的时候宕机之后，Chrome内部损坏，可能会丢失`最近关闭的标签页`这个数据，从而造成无法使用Chrome前台功能正常恢复。

有时打开的标签页可能已经数日，已经无法通过浏览器历史召回，此时就可以通过以下方法进行找回：

### 首先找到会话数据

参考这个[Chrome代码提交](https://source.chromium.org/chromium/chromium/src/+/e27098ba216a56402f42f98c1689c6bfe139a4c0)透露出的信息，在Chrome每次创建一个会话时，创建一个记录当前会话页面的`Session_${timestamp}`文件，当有`最近关闭的标签页`数据时，会创建一个`Tabs_${timestamp}`文件。这些文件被存储在：
```sh
# MacOS在
~/Library/Application\ Support/Google/Chrome/Default/Sessions
# 当Chrome升级时，会备份在这，当需要恢复早先的记录可以考虑使用
~/Library/Application\ Support/Google/Chrome/Snapshots/123.0.6312.124/Default/Sessions

# 注意：以下2个未验证，参考 https://chromium.googlesource.com/chromium/src/+/HEAD/docs/user_data_dir.md#user-cache-directory
# Windows在
C:\Users[USERNAME]\AppData\Local\Google\Chrome\User Data\Default\Sessions
# Linux在
/home/$USER/.config/google-chrome/Default/Sessions
```

### 解析出Tab对应的URL

当我们找到这些文件时，我们直接打开阅读，会发现是乱码(二进制文件)，这是Chrome的`SNSS`文件存储格式，需要对应的工具进行解析，经过一番查找，大部分工具都是过时，不能支持最新版的Chrome，只有[ccl_chrome_indexeddb](https://github.com/cclgroupltd/ccl_chrome_indexeddb)这个工具还在迭代中，可以支持。

```sh
# 下载工具：
git clone https://github.com/cclgroupltd/ccl_chrome_indexeddb.git
cd ccl_chrome_indexeddb
# 提取URL到1.txt
python3 ./ccl_chromium_snss2.py ~/Library/Application\ Support/Google/Chrome/Default/Sessions/Tabs_13355542992273182 | rg -o --pcre2 "(?<= url=')https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)(?=')"  > 1.txt
# 查看结果
cat 1.txt
```

然后我们就可以从提取到的标签页对应的链接来恢复页面了。