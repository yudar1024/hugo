+++
author = "陈 sir"
title = "windows terminal 保持 ssh session"
date = "2021-06-12"
description = "windows terminal 保持 ssh alive"
tags = [
    "windows terminal",
    "windows",
    "ssh",
]
categories = [
    "windows",
]

image = "/images/article.jpg"
+++

在windows 用户目录下创建 .ssh 文件夹，或者可以尝试连接 SSH 随意一个服务器即可自动创建此文件夹。

在其中创建配置文件 config ，写入以下两行。
```
Host *
  ServerAliveInterval 40
```