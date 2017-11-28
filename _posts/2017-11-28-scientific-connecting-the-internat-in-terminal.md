---
layout: post
title: 在终端/shell中科学上网
categories: [tool]
description: tool
keywords: tool
---

目前很多常见的科学上网工具主要适用于浏览器，但是在Shell中无法直接使用，因此需要在Shell中配置
代理所需的环境变量。但是在Shell中配置全局的环境变量会代理一个问题，那就是平时在工作中需要用到
很多内网的环境，如果统一都走代理则这些内网资源都无法访问(在电脑网络配置中配置的自动代理`PAC`，
只能够对浏览器生效)。因此需要一个智能代理，根据访问的资源来决定是否访问远程代理。

经过查访，找到一款智能代理，[COW](https://github.com/cyfdecyf/cow)，其根据目标连接超时时间判断
该目标是否被BAN，然后将被BAN的目标请求转发到配置的二级代理上去。因此我们可以使用该代理为Shell的
全局代理，如果是内网的地址其会直接转发到内网，如果目标连接超时，其会将我们的科学上网代理上去。

## 安装

可以参考[快速开始](https://github.com/cyfdecyf/cow#快速开始)

## 配置

下载[配置模板](https://github.com/cyfdecyf/cow/blob/master/doc/sample-config/rc)

然后修改:
```
listen = http://127.0.0.1:7777  # 本地代理监听地址
proxy = socks5://127.0.0.1:1080 # 二级代理使用socks5
```

## 开机启动(macOS)
```shell
cat << EOF > ~/Library/LaunchAgents/com.neal.cow.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd" >
<plist version="1.0">
<dict>
     <key>Label</key>
     <string>com.neal.cow</string>
     <key>ProgramArguments</key>
     <array>
      <string>/Users/xxx/dev/app/bin/cow</string>
          <string>-rc</string>
      <string>/Users/xxx/.cow/rc</string>
     </array>
     <key>KeepAlive</key>
     <true/>
</dict>
</plist>
EOF

launchctl load -w  ~/Library/LaunchAgents/com.neal.cow.plist
```

## 设置Shell的全局代理
```shell
# only for zsh 其他的shell方法类似
# 在`~/.zshrc`中追加以下两行：
export http_proxy=http://127.0.0.1:7777
export https_proxy=http://127.0.0.1:7777
```
