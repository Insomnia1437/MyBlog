---
title: MVME5500-vxworks
categories:
  - Accelerator Control
tags:
  - Control
  - VxWorks
  - VME
  - EPICS
  - Instrument
  - Chinese
abbrlink: '980566e0'
date: 2020-12-02 20:25:14
---

```python
# @Time    : 2020-11-28
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

> 最近折腾了一星期的VME和VxWorks，遇到了不少问题，怀疑了EPICS ioc，base版本，bootROM，kernel image文件权限，甚至开始怀疑VME机箱了，最后在东日研究所公司飯塚san的帮助下解决了，记录下踩的坑。最想吐槽的是因为VME5500使用RJ45作为串口，因此需要使用RJ45转Dsub9的线，但是我MacBook的拓展坞又不支持DB9，试了下中间加一层DB9转usb，失败，最后还是找了台8年前的HP老台式搞定，但运气不好的我又遇到了台式机电源挂掉，只能又从废弃机器里拆了一个还能用的电源给它换上，得出的教训就是以后再买Windows的办公笔记本，一定要买自带串口的。

<!-- more -->

## 基本介绍

因为我使用的是是MVME5500，CPU是MPC7457，PowerPC架构。要在VME5500上运行VxWorks，需要先通过串口在板卡的flash里写入bootROM。这个bootROM会初始化一些基本的服务，比如串口，然后通过串口去修改boot parameter，更改IP地址掩码，要使用的kernel文件地址等等。以下为boot parameter示例（p命令只能在boot环境使用，在VxWorks shell内还是要通过`bootChange`命令查看。）：

```shell
[VxWorks Boot]: p

boot device          : gei
unit number          : 0
processor number     : 0
host name            : lcba10
file name            : /mnt/misc/vxWorks/wind221/Sep2009/vxWorks
inet on ethernet (e) : 172.19.74.28:ffffe000
host inet (h)        : 172.19.64.190
gateway inet (g)     : 172.19.64.1
user (u)             : epics
flags (f)            : 0x0
target name (tn)     : IOCEVTEST5
startup script (s)   : /usr/users/control/epics/MRF_EV/develop/Xapp_091201/iocBoot/iocXappApp/st-pmagtest-Main-114MHz.cmd
```

## vme地址与cpu地址

板卡使用的是VME地址空间，而cpu只认识自己的地址空间，因此板卡的驱动程序需要map一下，cpu是32位，地址空间2G，具体映射在BSP的`config.h`文件中定义。我使用的BSP定义了VME A32地址空间中的128M，如下所示，VME地址`0x08000000`被映射到cpu地址空间的`0xe7f00000`，一个板卡可以只使用部分空间，比如我的板卡只用了1M的VME地址，但是要注意不同板卡之间同类地址空间不要有重叠（不同地址空间，比如有些板卡支持同时使用A16和A24，没试过）。

```
 * VME_A32_MSTR_LOCAL =             -----------------
 *              (0xe7f00000)       |                 } Translated to VME bus
 *       LSI1.bs =     (0xe7f00000)|                 } (0x08000000) LSI1.bs
 *                                 | VME A32 space   }
 *                                 |     128MB       }
 *       LSI1.bd - 1 = (0xe7f3ffff)| (0x08000000)    } (0x0803ffff) LSI1.bd -1
 *       LSI0.bs =     (0xe7f40000)| LSI0 and LSI1   } (0xfb000000) LSI0.bs
 *                                 | outbound maps   }
 *       LSI0.bd - 1 = (0xe7f4ffff)| through here    } (0xfb00ffff) LSI0.bd - 1
 *                     (0xe7f50000)|                 } (0xfb010000)
 *                                  .................
 *                                 :   Unused A32    :

```

## VxWorks常用命令
通过telnet进入VxWorks shell之后可以输入`help`查看可用命令，除了`ls` `pwd` `cd`这些之外，查看内存命令很常用。

比如我要看内存`0xe7f00000`的内容，可以使用`d`命令：

```
-> d 0xe7f00000
NOTE: memory values are displayed in hexadecimal.
0xe7f00000:  10e3 0000 0007 2200 0002 0680 0000 0000  *......".........*
0xe7f00010:  c000 fff7 f001 ffff 0000 0000 0000 0000  *................*
0xe7f00020:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f00030:  0000 0000 0000 0000 0000 0000 0100 0003  *................*
0xe7f00040:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f00050:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f00060:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f00070:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f00080:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f00090:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f000a0:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f000b0:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f000c0:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f000d0:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f000e0:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
0xe7f000f0:  0000 0000 0000 0000 0000 0000 0000 0000  *................*
value = 0 = 0x0
```

如果要查询一些VxWorks函数，可以：
```
-> lkup "Reset"
taskReset                 0x00258d98 text
sysUniverseReset          0x0010cb64 text
pciAutoDevReset           0x001035e4 text
sysSerialReset            0x00111210 text
pciConfigReset            0x0010244c text
value = 0 = 0x0
```

## 在VxWorks上编译和运行c程序

因为我要在VxWorks上运行epics，虽然也可以通过修改epics device support来调试，但是每次都要重新编译挺麻烦，还是单独写个c程序方便。

VxWorks编程我也不熟悉，就粗暴的引用了一堆头文件，然后调用一些函数，下面的程序调用了`sysBusToLocalAdrs`函数，把VME地址映射到cpu地址，然后通过VxWorks提供的函数`vxMemProbe`去测试cpu地址是否可读。

为什么写这个程序是因为我在测试reflective memory 5565模块时，地址映射成功，但是提示地址不可读。

```c
#include <stdlib.h>
#include <stdio.h>
#include <vxWorks.h>
#include <types.h>
#include <iv.h>
#include <vme.h>
#include <sysLib.h>
#include <memLib.h>
#include <intLib.h>
#include <logLib.h>
#include <vxLib.h>

int main()
{
    void **pp;
    long status;
    char pValue;
    unsigned int probe_addr;
    unsigned int step = 0x00010000;

    status = sysBusToLocalAdrs(0x0d, (char *)0x08000000, (char **)pp);
    // sysBusToLocalAdrs(0x0d, 0x08000000, &adr)
    if (status != OK)
    {
        printf("map 0x08000000 error\n");
        return -1;
    }
    printf("map: 0x08000000 to addr: %p, map status is %d\n", *pp, status);
    probe_addr = 0xe7f00000;
    while ( probe_addr <= 0xe8000000)
    {
        status = vxMemProbe((char *)probe_addr, VX_READ, 1, &pValue);
        if (status != OK)
        {
            printf("Cannot read addr %x\n", probe_addr);
        }
        else
        {
            printf("probe addr %x, status is %d, return read value: %d\n", probe_addr, status, pValue);
        }
        probe_addr += step;
    }
    return 0;
}
```

要编译这个程序需要设置VxWorks环境变量来指定头文件位置，由于编译epics过程中已经设置，这里就不多讲。编译器用的是`ccppc`。
编译之后会生成可执行文件，比如默认文件名a.c编译生成`a.out`，然后在VxWorks中执行：
`ld < a.out`
`main`
然后就会输出了。

## 其它命令

一些别的常用的VxWorks命令，`nfs`会很常用
```shell
-> nfsHelp
nfsHelp                       Print this list
netHelp                       Print general network help list
nfsMount "host","filesystem"[,"devname"]  Create device with
                                file system/directory from host
nfsUnmount "devname"          Remove an NFS device
nfsAuthUnixShow               Print current UNIX authentication
nfsAuthUnixPrompt             Prompt for UNIX authentication
nfsIdSet id                   Set user ID for UNIX authentication
nfsDevShow                    Print list of NFS devices
nfsExportShow "host"          Print a list of NFS file systems which
                                are exported on the specified host
mkdir "dirname"               Create directory
rm "file"                     Remove file

EXAMPLE:  -> hostAdd "wrs", "90.0.0.2"
          -> nfsMount "wrs","/disk0/path/mydir","/mydir/"
          -> cd "/mydir/"
          -> nfsAuthUnixPrompt     /* fill in user ID, etc. *
          -> ls                    /* list /disk0/path/mydir *
          -> copy < foo            /* copy foo to standard out *
          -> ld < foo.o            /* load object module foo.o *
          -> nfsUnmount "/mydir/"  /* remove NFS device /mydir/ *

```

对于VxWorks的时间，通过`sysClkRateSet()`和`sysClkRateGet()`获取的是system clock rate，默认值可以在BSP中修改，默认为60，即每秒钟时钟发生60次中断。

记住这个对于分析task的delay很重要。查看task的TCB命令为`taskShow()`，在shell中可以简单的使用`i`命令显示所有task：

```shell
-> i

  NAME         ENTRY       TID    PRI   STATUS      PC       SP     ERRNO  DELAY
----------  ------------ -------- --- ---------- -------- -------- ------- -----
tJobTask    1d08cc         3fd960   0 PEND         250fd4   3fd880       0     0
tExcTask    1cfc6c         3050f0   0 PEND         250fd4   304ff0       0     0
tLogTask    logTask        400d00   0 PEND         24f190   400bc0       0     0
tNbioLog    1d15a0         404620   0 PEND         250fd4   404500       0     0
tShell0     shellTask      51c340   1 PEND         250fd4   51c010       0     0
tShellRem5> shellTask      714ee0   1 READY        2596a0   7130e0       0     0
ipcom_tick> 263e48         4de3a0  20 DELAY        2584c4   4de320       0     2
tNet0       ipcomNetTask   409740  50 READY        250fd4   409630  3d0001     0
ipcom_sysl> 1786b4         4c55b0  50 PEND         2518f0   4c5400       0     0
ipcom_teln> ipcom_telnet   4ffb30  50 PEND         250fd4   4ff930       0     0
ipsntps     1bcee8         502cb0  50 PEND+T       250fd4   502b30      33    69
ipcom_teln> ipcom_telnet   51f600  50 PEND         250fd4   51f3e0       0     0
tStdioProx> 183bd4         521860  50 READY        250dcc   521360       0     0
tLogin51f6> 183e58         538ea0  50 PEND         2518f0   538dd0       0     0
tPortmapd   portmapd       509920  54 PEND         250fd4   5096d0      16     0
NTPTimeSyn> 10e0d7c        7aea20 109 PEND+T       250fd4   7ae8b0  3d0004   942
ClockTimeS> 10e0d7c        7b1dd0 109 PEND+T       250fd4   7b1c40  3d0004   522
cbHigh      10e0d7c        5a8580 128 PEND         250fd4   5a8460       0     0
timerQueue  10e0d7c        7d1fb0 129 PEND         250fd4   7d1e30       0     0
scanOnce    10e0d7c        5bfb60 132 PEND         250fd4   5bfa30       0     0
scan-0.1    10e0d7c        67b4f0 133 PEND+T       250fd4   67b330  3d0004     5
scan-0.2    10e0d7c        674c30 134 PEND+T       250fd4   674a70  3d0004     5
...
```

| Field  | Meaning                               |
| ------ | ------------------------------------- |
| NAME   | task 名称                             |
| ENTRY  | symbol name 或 task开始地址           |
| TID    | task ID                               |
| PRI    | 优先级                                |
| STATUS | task状态                              |
| PC     | 程序计数器                            |
| SP     | Stack pointer                         |
| ERRNO  | error code                            |
| DELAY  | 0的话代表没有delay，否则为clock ticks |

比如上面`scan-0.1`这个task，是epics record scan 为 0.1 second任务，delay ticks为5，换算一下也就是delay了大约0.1秒，对于`scan-0.1`来说，已经是影响record process的delay值了。

task的状态有这么几种：

| String   | Meaning                                                      |
| -------- | ------------------------------------------------------------ |
| READY    | 等待CPU                                                      |
| PEND     | 等待资源                                                     |
| DELAY    | Task is asleep for some duration.                            |
| SUSPEND  | Task is unavailable for execution (but not suspended, delayed, or pended). |
| DELAY+S  | delay+suspend                                                |
| PEND+S   | pend+suspend                                                 |
| PEND+T   | Task is pended with a timeout.                               |
| PEND+S+T | Task is pended with a timeout, and also suspended.           |
| ...+I    | Task has inherited priority (+I may be appended to any string above). |
| DEAD     | Task no longer exists.                                       |

