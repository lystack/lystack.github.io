---
layout: wiki
title: Git命令
cate1: Git
cate2:
description: Git
keywords: Git
---

# 更改仓库地址
```
git remote set-url origin new.git.url/here/xxx.git
git submodule sync
git remote show origin
```

# 基本配置
```
git config --global user.name <Your-Name>
git config --global user.email <Your-Email>
git config --global credential.helper store
git config --global core.autocrlf false
git config --global core.fileMode false
git config --global core.quotepath false
git config --global core.ignorecase false
```

# 命令别命
```
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
```
# 批量更新代码
```
for file in ./*; do if test -d $file; then echo pull $file && cd $file && git pull && cd ..; fi;done
```