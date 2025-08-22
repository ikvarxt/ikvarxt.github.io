---
title: git dev tips
date: 2025-08-23 04:03:17
permalink: git-dev-tips/
tags:
  - Tips
---

## .netrc

这个技巧是从公司内部的 gitlab 配置指南中学到的，主要功能是可以非常轻松的记住密码 token，比如 GitHub personal access toke。
用法：在用户主目录下创建 .netrc 文件, 内容如下：

```
machine github.com
login your_username
password your_access_token
```
只需要简单的这样一个文件，后面使用 https 拉取代码，就会自动登录，比配置什么 ssh 密钥方便太多了。

## gitconfig includeIf

在公司的电脑上面我们设定 git 全局用户名为公司的用户名，但我们可能希望在维护开源项目时使用自己的 GitHub 用户名，那么就可以将开源项目放到一个 opensource 目录下，在 `~/.gitconfig` 中配置如下：

```
[includeIf "gitdir:~/opensource/"]
    path = ~/opensource/.gitconfig
```

在 ~/opensource/.gitconfig 配置里我们就可以定义如下，
```
[user]
    name = ikvarxt
    email = hi@ikvarxt.me
```
这样在 ~/opensource 目录下的所有 git 仓库里面，我们都会以 ikvarxt 这个名称来提交了。切换到其他目录，则会使用全局设置。

舒服。

