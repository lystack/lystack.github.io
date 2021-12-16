---
layout: wiki
title: Linux常见问题
cate1: linux
cate2:
description: 记录linux使用过程中遇到的问题
keywords: linux
---

# https证书认证问题
问题描述：使用 `wget/curl` 等工具时，如果对应的网址是 https，会报如下错误

`cannot verify www.xxx.com's certificate, issued by xxx`

CentOs 解决方案:

安装根证书管理包软件：
`yum install -y ca-certificates`

更新CA证书：
`update-ca-trust`