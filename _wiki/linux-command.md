---
layout: wiki
title: Linux常用命令
cate1: linux
cate2:
description: 记录linux常用命令，方便查阅
keywords: linux
---

# 磁盘
## 磁盘剩余空间
`df -h`
## 当前目录磁盘大小
`du -h --max-depth=1`
## 磁盘空间占用最大的前10个文件或文件夹
`du -hsx * | sort -rh | head -10`
## 文件是否被进程占用导致无法删除
`lsof | grep deleted`

# 进程
## 被系统killed掉的进程
`dmesg | egrep -i -B100 'killed process'`

`egrep -i 'killed process' /var/log/messages`

`egrep -i -r 'killed process' /var/log`

# awk
## 去除重复行
`cat file.txt | awk '!a[$0]++'`
## 删除首尾行
### 删除首行
```
cat file.txt | awk 'NR>1 {print $0}'
cat file.txt | sed 1d
```

awk删除开头n行只需要 NR>n 即可

sed删除开头n行需要 1,nd 即可
### 删除尾行
```
cat file.txt | awk 'NR>1 {print line}{line=$0}'
cat file.txt | sed '$d'
````

### 删除首尾行
`cat file.txt | awk 'NR>2 {print line}{line=$0}'`

### 更简单的方式
`cat file.txt | head -n -2`

负数表示显示到倒数第几行

# vim
## 删除行
```
:g/pattern/d

:%s/pattern//g
e.g :%s/.*_id.*$\n//g  注意要匹配换行符一起删除
```
vim对一次编辑的行数有限制，可通过 `@:`重复执行命令，也可`10@:`重复执行10次

# 遍历
```
for dir in `ls`; do cd $dir && git pull && cd ..; done
```