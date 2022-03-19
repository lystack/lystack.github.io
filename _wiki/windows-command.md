---
layout: wiki
title: Windows常用命令
cate1: windows
cate2:
description: 记录windows常用命令，方便查阅
keywords: windows
---

# 端口占用
```
netstat -aon|findstr "8081" // 最后一项为pid
tasklist|findstr "23212" // 根据pid查看进程
taskkill /T /F /PID 23212  // 杀死进程
```