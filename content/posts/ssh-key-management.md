---
title: ssh key management
categories:
  - MEMO
  - ssh
tags:
  - Shell
abbrlink: b6ce2bc4
date: 2021-05-25 06:34:09
---

```python
# @Time    : 2021-05-25
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```
# ssh key

> 反正就是突发奇想，决定解决下杂乱无章ssh key管理方案。首先所里内网服务器过多，以及一堆跳板服务器用来访问不同子网，所以ssh key登录所内网的方案会很麻烦，老老实实用密码登录（~~可能这也是我一直没有使用ssh key的一个原因~~）。但最近vps机器增多，导致使用密码登录会很闹心。

<!-- more -->
需求:
多个客户端和多个服务器。
GitHub和内网的GitLab

[ssh config](https://man.archlinux.org/man/ssh_config.5)

在wsl内使用ssh agent似乎存在bug，会出现私钥文件权限的识别错误，在这个[issue](https://github.com/Microsoft/WSL/issues/3181)内有详细讨论。虽然这个讨论是三年前（2018）的，但我依然遇到了这个问题。也许wsl2修复了，但我没有测试。

------------------
最后的方案是使用一服务器一key的方式，把key存在bitwarden里，然后复制到需要使用的机器上。