+++
author = "陈sir"
title = "outlook office 365 无法连接exchange server"
date = "2021-07-04"
description = "outlook office 365 无法连接exchange server"
tags = [
    "software-problem",
]
+++

迁移到 Office 365，Outlook 无法连接或 web 服务不起作用

您的邮箱将被迁移到 Office 365 后，可能会遇到以下问题：

Outlook 无法连接到 Exchange Server。

不能使用 Out of Office、 会议可用性、 邮件提示、 在线存档或自动发现所依赖的任何其他 web 服务功能等功能。

当您打开 Outlook 时，您会收到以下错误消息：

> Outlook 无法登录。 请验证您连接到网络并使用正确的服务器和邮箱名称。 Microsoft Exchange 信息服务配置文件中的缺少所需的信息。 修改您的个人资料，以确保您使用的正确的 Microsoft Exchange 信息服务。

### 原因
迁移前自动发现服务器的名称缓存在注册表中的 Outlook 配置文件时，会出现此问题。

### 解决方案
若要解决此问题，请使用下列方法之一。 方法 1:使用 ExcludeLastKnownGoodUrl 来防止 Outlook 使用上次已知的好自动发现 URL 按如下所述配置以下注册表子项之一：

HKEY_CURRENT_USER\Software\Microsoft\Office\x.0\Outlook\Autodiscover Dword 值： ExcludeLastKnownGoodUrl 值： 1 HKEY_CURRENT_USER\Software\Policies\Microsoft\Office\x.0\Outlook\Autodiscover Dword 值： ExcludeLastKnownGoodUrl 值： 1 注意x.0占位代表您的 Office 版本 (16.0办公室 2016年、 Office 365 和 Office 2019，于 15.0 = = 办公室 2013 年, 如果使用的是office 365 则是HKEY_CURRENT_USER\SOFTWARE\Microsoft\Office\Outlook)。 注意如果将ExcludeLastKnownGoodUrl值设置为 1，Outlook不使用最后一次使用自动发现 URL。 方法 2:创建新的 Outlook 配置文件

