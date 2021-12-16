---
layout: wiki
title: JSON
cate1: linux
cate2:
description: JSON
keywords: linux
---

# 磁盘相关
## 磁盘剩余空间
`df -h`
## 当前目录磁盘大小
`du -h --max-depth=1`
## 磁盘空间占用最大的前10个文件或文件夹
`du -hsx * | sort -rh | head -10`
## 文件是否被进程占用导致无法删除
`lsof | grep deleted`

# 进程相关
## 被系统killed掉的进程
`dmesg | egrep -i -B100 'killed process'`

`egrep -i 'killed process' /var/log/messages`

`egrep -i -r 'killed process' /var/log`