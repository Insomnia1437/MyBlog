---
title: Python版本管理与虚拟环境
categories:
  - Software
tags:
  - Linux
  - Python
  - Chinese
abbrlink: 7a8ba2cd
date: 2020-01-22 00:12:07
---

```python
# @Time    : 2019-12-07
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

> 对于控制领域而言，目前常用的epics Python support有CaChannel, caffi, caproto, Cothread, pvaPy和 PyEpics 3. 不同的用户有不同的偏好软件，而哪怕是同一个软件也存在版本更迭。Python环境的管理一直是个大难题，包括Python版本和库依赖管理，在尝试了几种方式后，目前选择了`pyenv+pyenv-virtualenv`的方式。
<!-- more -->
## pyenv

[pyenv at GitHub](https://github.com/pyenv/pyenv)

简单介绍下pyenv的工作原理：

操作系统在解析`PATH`时按照从左到右的顺序，若找到特定的可执行文件`executable file`则停止。

pyenv就是通过在`PATH`中添加`shims`，可翻译为`垫片`，来覆盖不同版本的Python路径。例如若`/usr/local/bin`中有`python2.7`，而你在`$(pyenv root)/shims`下通过pyenv安装了`python3.6`，你的程序就会使用`python3.6`。

`$(pyenv root)/shims:/usr/local/bin:/usr/bin:/bin`

在安装pyenv后，默认安装在`~/`目录，即当前用户的根目录，安装之后可以通过pyenv安装不同版本的Python，Python安装目录都在pyenv目录下：

- $(pyenv root)/versions/2.7.8/
- $(pyenv root)/versions/3.4.2/
- $(pyenv root)/versions/pypy-2.4.0/

### 安装

按照github repository的介绍，通过git checkout pyenv到本地, 如果在服务器上安装，别忘记设置代理服务器的环境变量：`http_proxy`和`https_proxy`。

```shell
$ git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

然后在你的shell启动脚本中设置环境变量，我使用的是zsh
```shell
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
```

初始化pyenv
```shell
$ echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.zshrc
```

然后重启shell就能生效了
`exec "$SHELL"` 或者 `source ~/.zshrc`

### 使用
之后就可以使用pyenv了，支持的命令如下。（virtualenv等下安装）

```
activate            exec                init                rehash              uninstall           version-file-read   versions            virtualenv-prefix
commands            global              install             root                version             version-file-write  virtualenv          virtualenvs
completions         help                local               shell               --version           version-name        virtualenv-delete   whence
deactivate          hooks               prefix              shims               version-file        version-origin      virtualenv-init     which
```

[常用的命令介绍](https://github.com/pyenv/pyenv/blob/master/COMMANDS.md)

安装特定版本的Python，更新shims（每次安装或卸载版本都要rehash进行更新），查看系统中的Python版本。
```
$ pyenv install 3.7.6
$ pyenv rehash
$ pyenv versions
  system
* 3.7.6 (set by /home/sdcswd/.pyenv/version)
  miniconda3-3.19.0
```

可以通过`local` `shell` `global`这三个命令灵活的改变Python的版本。

- global: 设置全局的Python版本。
- local: 设置当前目录下的Python版本，会覆盖`global`的设置，可分别设置不同项目的Python版本。极其有用。
- shell: 设置当前shell的Python版本，退出shell就没了，一旦设置就会覆盖`local`和`global`。（至今没用过...

```shell
$ pyenv local 2.7.6
$ pyenv local --unset
$ pyenv global 2.7.6
$ pyenv shell pypy-2.2.1
```

## pyenv-virtualenv
[pyenv-virtualenv at GitHub](https://github.com/pyenv/pyenv-virtualenv)

pyenv-virtualenv是用来管理Python虚拟环境的，对不同的项目设置虚拟环境可以灵活的为不同的项目创建干净独立的运行环境。

### 安装

安装基本和pyenv一样，从github下载
```shell
$ git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
```

可以选择是否设置`pyenv virtualenv-init`进行自动激活虚拟环境，即进入某个设置了虚拟环境的目录就激活虚拟环境，我感觉还是非常方便的，建议设置。
```shell
$ echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.zshrc
```

### 使用

创建虚拟环境，列出虚拟环境（目录和上面不一样是因为我换了另一台电脑），激活和退出（如果上面设置了就不用自己激活了）
```shell
$ pyenv virtualenv 2.7.10 my-virtual-env-2.7.10
$ pyenv virtualenvs
    miniconda3-3.19.0 (created from /Users/sdcswd/.pyenv/versions/miniconda3-3.19.0)  
    miniconda3-3.19.0/envs/PV_crawler (created from /Users/sdcswd/.pyenv/versions/miniconda3-3.19.0/envs/PV_crawler)  
    miniconda3-3.19.0/envs/pydm-environment (created from /Users/sdcswd/.pyenv/versions/miniconda3-3.19.0/envs/pydm-environment)
$ pyenv activate <name>
$ pyenv deactivate
```

如果习惯使用conda，也可以使用conda来创建和查看虚拟环境，命令`pyenv virtualenvs`和`conda env list`都可以显示系统中的虚拟环境。

## PyEepics

然后安装epics Python库，我习惯使用pyepics。

使用pip安装
`pip install pyepics`，然后就能使用EPICS了。

```shell
$ python
Python 3.7.6 (default, Jan 13 2020, 16:58:46)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-39)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import epics
>>> epics.caget("LIiEV:BGATE:LER:gun")
63.0
```