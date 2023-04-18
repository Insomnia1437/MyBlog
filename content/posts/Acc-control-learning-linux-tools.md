---
title: 粒子加速器控制-Linux命令整理
categories:
  - Accelerator Control
tags:
  - Linux
  - Chinese
  - Control
abbrlink: a44e3b13
date: 2020-02-07 17:50:46
---

```python
# @Time    : 2020-02-07
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```
> 本系列主目录：
>  [粒子加速器控制](/posts/acc-control-learning-catalog)
------
> 这周把前段时间积累的一些工作给完成了，包括：
>
> - 使用Elastic Stack对EPICS控制系统网络进行监控与可视化
> - 编写了程序调用caSnooper自动检测网络内PV请求状况并发送邮件提醒，源码已托管在GitHub
> - 对定时系统Bucket Selection的程序处理逻辑bug fix，还没有更新到运行环境
> - 花了一晚上更新了博客，添加了一部分功能，本想添加豆瓣读书和电影模块，但似乎模块有bug，已提交了issue，等作者解决。在源码中添加了Travis CI支持，以后可以在页面上写博客了。
> - 这几天发生了很糟糕的事情，哀悼之余着手用Amazon 免费的云服务器EC2搭了一个v2ray的服务，便于仍在墙内的朋友去接受多样的信息。
>
<!-- more -->
>
> 这个周末决定先努力把我常用的Linux命令以及一些不熟悉的命令给整理一下，如果有时间的话再读一读科大师兄早就发给我的定时系统的论文，以及几个同门师兄的博士毕业论文，看看对于自己的课题有没有什么启发；如果还有时间就再去研究下 [https://github.com/ChannelFinder](https://github.com/ChannelFinder)  似乎BNL和ANL都在使用，正好看看是自己造轮子还是复用。本文内容主要基于 [此文章](https://linuxtools-rst.readthedocs.io/zh_CN/latest/)，向作者表示感谢！
>
> 
>
> 最后，引用陶渊明的诗寄托哀思。
>
> 《拟挽歌辞》 其三
>
> 荒草何茫茫，  白杨亦萧萧。
> 严霜九月中，  送我出远郊。
> 四面无人居，  高坟正嶕峣。
> 马为仰天鸣，  风为自萧条。
> 幽室一已闭，  千年不复朝。
> 千年不复朝，  贤达无奈何。
> 向来相送人，  各自还其家。
> 亲戚或余悲，  他人亦已歌。
> 死去何所道，  托体同山阿。

------

几个很好的查询网站

- 显示命令以及参数的作用 [https://www.explainshell.com/](https://www.explainshell.com/)
- 学习正则表达式 [https://github.com/ziishaned/learn-regex](https://github.com/ziishaned/learn-regex)
- 测试正则表达式 [https://regex101.com/](https://regex101.com/)
## 文件与目录管理

```shell
# 查找文件或目录
find ./ -name "core*" | xargs file

# 查找到文件之后执行某些命令， -exec参数后跟command，以semicolon结尾，同时加backslash转义
find ./ -name "*.o" -exec rm {} \;
# 或者
find ./ -name "*.o" | xargs rm

# 查找文件内容
egrep linux *
# -i 忽略大小写，-n显示行号，-H显示命中行以及文件名，-C显示前后几行的内容
find ./ -name "*.log" | xargs grep -inH -c2 "linux"
find ./ -name "*.log" -exec grep -inH -c2 "linux" {} \;

# 管道和重定向
ls /proc/* > list 2>&l
ls /proc/* &> list
# 清空文件
:> a.txt
```

## 文本处理

```shell
# 查找txt和log文件
find . \( -name "*.txt" -o -name "*.log" \) -print
# 否定参数
find . ! -name "*.txt"
# 七天内被访问过的文件 atime mtime ctime, amin mmin cmin，
find . -type f -atime -7
# 权限
find . -type f -perm 777
find . -type f -name "*.php" ! -perm 644
find . -type f -user tom
find . -type f -group sunk
# exec
find . -type f -mtime +30 -name "*.log" -exec cp {} old \;

# xargs -n多行输出
echo "1 2 3" | xargs -n1
# -d指定分隔符
echo "nameXnameXnameXname" | xargs -dX -n2

# 递归搜索
grep "class" . -R -n
# 排序 -n按数字 -d按字典序 -r逆序 -k N按第N列 -t指定分隔符 -b忽略行首空格 -u去重复
sort -nrk 1 data.txt
# 行出现次数
sort file.txt | uniq -c
# 重复行
sort file.txt | uniq -d

# tr
cat file | tr -d '0-9'
cat text | tr '\t' ' '
# 只保留数字 -c代表补集
echo "aa.,a 1 b#$bb 2 c*/cc 3 ddd 4" | tr -d -c '0-9 \n'
# -s表示压缩
echo "thissss is      a text linnnnnnne." | tr -s ' sn'
# 通用字符
# [:alnum:]：字母和数字
# [:alpha:]：字母
# [:cntrl:]：控制（非打印）字符
# [:digit:]：数字
# [:graph:]：图形字符
# [:lower:]：小写字母
# [:print:]：可打印字符
# [:punct:]：标点符号
# [:space:]：空白字符
# [:upper:]：大写字母
# [:xdigit:]：十六进制字符

# cut截取非第二列
cut -f2 --complement filename
# 指定分隔符
cut -f1,3 -d";" filename
# -b以byte为单位，-c以character为单位 
cut -c-5 file

# 拼接文件 注意cat是直接连接俩文件 paste是每一行连接在一起
paste file1 file2 -d ","
# cat
cat f1 f2
1
2
a
b
# paste
paste f1 f2
1       a
2       b

# sed 替换每一行的第2处匹配
sed 's/text/replace_text/2' file
# 替换第3到10行第二个以后的匹配
sed '3,10s/text/replace_text/2g'
# 替换全局匹配 默认输出替换后的内容 -i会覆盖原文件
sed 's/text/replace_text/g' file
# 移除空白行 /d代表删除
sed '/^$/d' file
# 变量转换
echo this is en example | sed 's/\w+/[&]/g'
[this]  [is] [en] [example]
# 如果想给markdown中有些关键词加粗，可以直接使用sed命令
echo 'go to directory /usr/bin' | sed 's/\/usr\/bin/**&**/g'
go to directory **/usr/bin**
# 可以实现批量注释，在第三行到第六行开头加上`#`
sed '3,6s/^/#/g' file
# 在function和return之间的所有行，行首加上`#`
sed '/function/,/return/s/^/#/' file
# 子串匹配标记 \1 \2等表示capturing group（正则表达式概念） 
echo "/proc/cpuinfo" | sed 's,/\(.*\)/\(.*\),\1:\2,'
proc:cpuinfo


# awk
# NF 表示多少个字段 -F指定分隔符
# NR 表示记录数量 即行号
awk -F ':' '{print $1, $(NF-1)}' /etc/passwd | head
root /root
daemon /usr/sbin
bin /bin
sys /dev
sync /bin

# 文件有多少行 等同于 wc -l file
awk 'END {print NR}' file
# 文本过滤
awk 'NR < 5' file
awk 'NR==1,NR==4 {print}' file
awk '/cpu/'

# 内置函数
tolower()：字符转为小写。
length()：返回字符串长度。
substr()：返回子字符串。
sin()：正弦。
cos()：余弦。
sqrt()：平方根。
rand()：随机数。

# 打印指定列 awk 和 cut
ls -lrt | cut -f6
ls -lrt | awk '{print $6}'

# 读文件
while read line;
do
echo $line;
done < file.txt
# 改成子shell:
cat file.txt | (while read line;do echo $line;done)
```

## 系统管理

```shell
# 按文件大小排序目录下文件
ls | xargs du | sort -rn 
# 查看目录空间 s代表递归计算，也可以设置--max-depth
du -sh
du -h --max-depth=1
# 查进程 
ps -ef
ps aux

lsof -i:5064
lsof -u username
lsof -c python
lsof -p 43210
lsof +d mydir/
# 查看进程内存情况
pmap PID

# CPU
sar -u
# 每秒采样一次 输出5次
sar -q 1 5 
# 内存
free -h
sar -r 1 2
vmstat 1 5
# 网络 -t tcp -l listen状态的端口
netstat -at
netstat -antp | grep 6379
scp
sftp
rsync
```





















