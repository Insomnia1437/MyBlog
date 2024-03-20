---
title: Shell Script Learning
categories:
  - MEMO
  - Linux
tags:
  - Linux
  - Shell
  - Chinese
abbrlink: b81d8b84
date: 2019-12-07 16:12:17
---

```python
# @Time    : 2019-12-07
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

# Shell 脚本编程

> 一直以来只是使用bash或者zsh，对于shell脚本并不了解，上次组会导师告知可以通过shell script进入epics IOC的shell，进而通过**dbl**命令获取所有IOC的所有PV，虽然只是获取了PV名，进一步获取PV Type，Value还需要进一步执行**dbpr**等命令，不过可以先试着玩一下。
<!-- more -->
关于Shell脚本一些基本介绍此处就忽略了，只记录一些我还不太熟悉的。忽略掉的包括：
- 运行方法
- 注释
- 变量类型，定义与使用

## 字符串
### 单引号
```shell
str='this is a string'
```

- 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的
- 单引号字串中不能出现单引号（对单引号使用转义符后也不行）

### 双引号
```shell
your_name='test'
str="Hello, I know your are \"$your_name\"! \n"
```

- 双引号里可以有变量
- 双引号里可以出现转义字符

### 字符串长度
```shell
string="abcd"
echo ${#string} #输出：4
```

### 查找字符串长度
```shell
string="All work no play makes Jack a dull boy"
echo `expr index "$string" io`  # 查找字符 i 或 o 的位置(哪个字母先出现就计算哪个)：
```

## 数组

```shell
array_name=(value0 value1 value2 value3)

# 使用 @ 或者 * 符号可以获取数组中的所有元素
echo ${array_name[@]}

# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
# 取得数组单个元素的长度
lengthn=${#array_name[n]}
```
|参数	|说明   |
|:---------------:|:----|
| $#	 |传递到脚本的参数个数|
| $*	 |以一个单字符串显示所有向脚本传递的参数。如"$*"用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数。|
|$$	 |脚本运行的当前进程ID号|
|$!	 |后台运行的最后一个进程的ID号|
|$@	 |与$*相同，但是使用时加引号，并在引号中返回每个参数。如"$@"用「"」括起来的情况、以"$1" "$2" … "$n" 的形式输出所有参数。|
|$-	 |显示Shell使用的当前选项，与set命令功能相同。|
|$?	 |显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误。|

## 运算符

> 常见的一些运算符不列出或者不多加说明了。

> **表达式和运算符之间要用空格**

> 算术运算的‘=’是赋值，‘==’是比较，字符串运算‘=’是比较。


运算符| 说明
:----:|:----
-eq|相等
-ne|不相等
-gt|...
-lt|...
-ge|...
-le|...
!|布尔非
-o|布尔或
-a|布尔与
&&|逻辑AND
\|\||逻辑OR
-z|字符串长度为0？
-n|字符串长度不为0？
$|字符串为空？
-b file | blcok device?
-c file | character device? [关于这俩的解释](https://unix.stackexchange.com/questions/60034/what-are-character-special-and-block-special-files-in-a-unix-system)
-d file | 目录
-f file | 普通文件
-g file | 是否设置SGID
-k file | 是否设置Sticky Bit
-p file | 是否命名管道（Named pipe or FIFO）
-u file | 是否设置SUID
-r file | 文件可读
-w file | 可写
-x file | 可执行
-s file | 文件为空
-e file | exist？
-S file | socket?
-L file | exist and link?


## Shell 重定向

> 文件描述符 0 通常是标准输入（STDIN），1 是标准输出（STDOUT），2 是标准错误输出（STDERR） i.e. 将 stdout 和 stderr 合并后重定向到 file: `command >> file 2>&1`

> 屏蔽 stdout 和 stderr: `command > /dev/null 2>&1`


命令|说明
:----:|:----
command > file | 输出重定向到file
command < file | 输入重定向到file
command >> file | 输出追加重定向到file
n > file | 文件描述符n重定向到file
n >> file | 文件描述符n追加重定向到file
n >& m | 输出文件n m合并
n <& m | 输入文件n m合并
<<tag  | tag之间的内容作为输入

待补充...


> **20240320更新：**


## Default values

- `${foo:-val}` 	$foo, or val if unset (or null)
- `${foo:=val}` 	Set $foo to val if unset (or null)
- `${foo:+val}` 	val if $foo is set (and not null)
- `${foo:?message}` 	Show error message and exit if $foo is unset (or null)

> Omitting the : removes the (non)nullity checks, e.g. ${foo-val} expands to val if unset otherwise $foo.

在[这里](https://devhints.io/bash)看到对于shell中的默认值, 有两种写法: `${foo-val}`, 对于带不带`:`二者的区别有点难以理解. 测试后发现区别在于

- 当`foo`未定义, 二者相同
- 当 `foo=""`定义为空, 带`:`的情况下, `foo=val`; 不带情况下`foo=""`.

也就是说, 带`:`应该是更好的习惯.