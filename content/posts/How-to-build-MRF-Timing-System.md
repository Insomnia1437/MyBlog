---
title: How to build MRF Timing System
categories:
  - MEMO
  - Linux
tags:
  - EPICS
  - vxWorks
  - KEK
  - MRF
  - Timing System
  - EVR
  - English
abbrlink: '59e84085'
date: 2019-11-30 18:04:35
---

```python
# @Time    : 2019-11-30
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

## Build MRF EVR230 on vxWorks
> This file is a guide to build your own MRF Timing IOC on vxworks as well as write your own device support to meet your own demand of EPICS record.
<!-- more -->
Some links might be helpful:
- [MRF Company](http://mrf.fi/)
- [EPICS](http://www.aps.anl.gov/epics/)
- [mrfioc2](http://epics.sourceforge.net/mrfioc2/)
- [MSI](https://epics.anl.gov/extensions/msi/index.php)
- [devlib2](http://epics.sourceforge.net/devlib2/)
  
### Requirements
- EPICS Base >= 3.14.8.2
- devLib2 (2.9)
- MSI (Macro expansion tool) Required with Base < 3.15.1

### Build mrfioc2
Download mrfioc2-2.2.0.tar.gz and extract to directory, name as "mrfioc2-2.2.0"

Download devlib2-2.9.tar.gz and extract to directory name as "devlib2-2.9"

devlib2:

> edit file: configure/RELEASE

> Add EPICS base full path

> run make

mrfioc2

> edit file: configure/RELEASE

> Add EPICS base full path and DEVLIB2 path
> e.g. DEVLIB2=$(EPICS_BASE)/modules/devlib2-2.9

> run make

### Create EVR application

```
# use these command to create the IOC
$EPICS_BASE/bin/$EPICS_HOST_ARCH/makeBaseApp.pl -t ioc temp
$EPICS_BASE/bin/$EPICS_HOST_ARCH/makeBaseApp.pl -i -t ioc -p temp temp
# The following target architectures are available in base:
    linux-x86_64
    vxWorks-ppc604_long
What architecture do you want to use? vxWorks-ppc604_long
# Then you should get below files in the directory
configure  iocBoot  Makefile  tempApp
```

> edit file: configure/RELEASE

> Add DEVLIB2, MRFIOC2 and EPICS_BASE full path

Then write your own database file and device support file if needed. (Talked later)

Modify following files with the instruction of the simple example of Device Support.

> ***App/src/Makefile

> ***App/Db/Makefile

> edit file:**App/src/Makefile
```
for EVR (mrfioc2)
***_DBD += mrfCommon.dbd drvemSupport.dbd
***_DBD += evrSupport.dbd mrmShared.dbd

for devlib2
***_DBD += epicsvme.dbd epicspci.dbd

***_LIBS += evrMrm mrmShared evr mrfCommon
***_LIBS += epicspci epicsvme
```

> edit file:iocBoot/ioc**/st.cmd

```
# id, slot(1 org), base Adr, level, vector
mrmEvrSetupVME("tstEVR", 3, 0x08000000, 5, 0xc0)
```
> run make

> run this IOC on vxWorks

## How to write your EPICS own device support

You need to Add these files:

```
***App/src/devYourdevsup.c
***App/src/devYourdevsup..h
***App/src/Yourdbd.dbd
***App/Db/Yourdb.db
```
And edit 
```
***App/src/Makefile
***App/Db/Makefile
iocBoot/ioc***/st.cmd
```

For example, I changed the `static long read_wf(waveformRecord *prec)` function from the original 

`epics-base-master/src/std/dev/devWfSoft.c` file

and add my feature (make it possible to storage String type array in a waveform record).

[My EventWf-IOC on GitHub](https://github.com/Insomnia1437/EventWf-IOC)
> Note this IOC runs on Windows 10

```
***App/src/devEventWf.c
***App/src/devEventWf.h
***App/src/eventWf.dbd
***App/Db/eventWf.db
```

In the `***App/src/Makefile` file, add:

```
LIBRARY_IOC += eventWf
DBD += eventWf.dbd
eventWf_SRCS += devEventWf.c
eventWf_LIBS += $(EPICS_BASE_IOC_LIBS)
***_DBD += eventWf.dbd
myapp_LIBS += eventWf
```

In the `***App/Db/Makefile` file, add:

```
DB += eventWf.db
```

In the `iocBoot/ioc***/st.cmd` file, add your db.

run make

