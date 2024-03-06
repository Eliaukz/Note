---
title: 我的git常用命令
tags:
  - git
category: git
---
git官方网站[Git (git-scm.com)](https://git-scm.com/)
git文档[Git - Book (git-scm.com)](https://git-scm.com/book/zh/v2)
## git配置用户信息

```bash
# --global 表示全局配置信息
$ git config --global user.name "username"
$ git config --global user.email "useremail"
```
配置存储模式
```bash
git config --global credential.helper store
```

## git
### 初始化
```bash
git init <pathname>
```
### 添加文件
```bash
git add <filename>
```
### 提交文件
```bash
git commit -m 'commit log' 
```
### 查看分支状态
```bash
git status 
```
### 切换分支
```bash
git checkout [-b  newreponame]/reponame  
```
### clone文件
```bash
git clone url filename
```

### 获取远程分支
```bash
git fetch
# 合并分支
git merge reponame/分支名
```
### 其他
```bash
# 查看版本历史
git log
# 查看本地分支
git branch
# 查看更改的内容
git diff
```




