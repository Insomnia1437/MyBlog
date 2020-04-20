---
title: Raspberry pi
date: 2020-04-17 15:13:34
categories:
 - Memo
 - Raspberry Pi
tags:
 - Raspberry Pi
 - English
---

```python
# @Time    : 2020-04-17
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

# Raspberry pi

Some memo during the usage of Raspberry Pi

<!-- more -->

## Using a proxy

`vim /etc/environment`

```shell
export http_proxy="http://yourproxy:prot"
export https_proxy="https://yourproxy:prot"
export no_proxy="localhost, 127.0.0.1"
```

Then run `sudo visudo`, add below line:

`Defaults    env_keep+="http_proxy https_proxy no_proxy"`

Reboot

## EPICS related

`EPICS_HOST_ARCH` should be `linux-arm`