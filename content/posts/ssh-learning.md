---
title: ssh learning
categories:
  - MEMO
  - Linux
tags:
  - Linux
  - Chinese
  - ssh
abbrlink: a5271c15
date: 2020-04-13 13:55:49
---

```python
# @Time    : 2020-04-13
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

# 关于ssh的配置

每天都要通过ssh登陆远程主机，所以一些偷懒的配置是必不可少的。ssh是基于非对称加密的RSA算法，连接过程也包括了host的认知，user的认证几个过程，这些知识包含很多密码学的内容，有很多博客都有讲。我就单记录一下自己的一些配置。

<!-- more -->

## ssh公钥认证和代理转发

基本逻辑是将client的公钥存储在远程host上，以后连接时候就不用输入host的账户密码了。

首先用`ssh-keygen`生成client机器上的密钥对：

``` shell
$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ~/.ssh/id_rsa -P ''
```
其中`-t`指定密钥类型，`-b`指定密钥长度，`-C`指定注释，`-f`指定私钥文件，`-P`指定私钥的密码，这里设置为不需密码

然后用`ssh-copy-id`，把公钥发送到远程主机上。

` $ ssh-copy-id [-i [identity_file]] [user@]machine`

如果有多个密钥对或者没有使用默认公钥路径，就要用`-i`参数指定，公钥会存储在远程host的user home目录下的`~/.ssh/authorized_keys`
这样以后再登陆就不需要输入密码了，当然如果设置了私钥的密码`passphrase`，还是需要输入的。

当client有多个密钥对时，即使分发了自己的公钥到不同的host，client也不知道要读取哪一个私钥文件`id_rsa`，所以还是要使用`ssh -i key`选项去指定要配对的私钥文件。所以就有了引入`ssh-agent`的需求，它就可以记录哪一个私钥对应哪些远程主机。

运行`ssh-agent`

` $ eval "$(ssh-agent -s)"`

之后查看环境变量是否有`SSH_AUTH_SOCK`和`SSH_AGENT_PID`，如果导入了，就说明运行成功
```shell
$ ssh-add
$ ssh-add ~/.ssh/id_rsa_1
$ ssh-add ~/.ssh/id_rsa_2
```

完成。

## 基于ssh的一些命令

基本的ssh命令，一些常用的参数：

```
ssh [option] [user]@[hostname] [command]

-A              ：开启ssh agent forwarding，不安全，官方建议在确信远程主机可靠的情况下再开启。(等下会讲什么是agent forwarding)
-a              ：关闭ssh agent forwarding
-b bind_address ：系统有多个IP时可以绑定用于ssh的地址
-E log          ：debug日志输出文件
-F config       ：配置文件，默认为~/.ssh/config
-i key          ：指定私钥文件，默认为~/.ssh/id_rsa
-l username     ：指定登陆的用户名
-p port         ：指定端口号
-t              ：强制分配伪终端
-X              ：开启X11 forwarding，同样，可能被攻击。
-Y              ：开启trusted X11 forwarding。和`-X`的区别，有安全方面的考虑，也有历史遗留的因素，建议使用-X。
```
### scp

scp是基于ssh的文件传输命令。
```
scp [-12BCpqrv] [-l limit] [-o ssh_option] [-P port] [[user@]host1:]src_file ... [[user@]host2:]dest_file
-C              ：先压缩再复制
-p              ：保留源文件的属性
-r              ：递归复制，传输目录
```