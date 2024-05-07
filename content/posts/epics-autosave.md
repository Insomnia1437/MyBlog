---
title: "epics autosave"
date:  2024-05-07T10:48:33+02:00
# weight: 1
# aliases: ["/first"]
tags: ["Chinese"]
author: "DW"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
# description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    # URL: "https://github.com/Insomnia1437/MyBlog/tree/master/content"
    URL: ""
    Text: "Source" # edit text
    appendFilePath: true # to append file path to Edit link
---
```python
# @Language: Markdown
# @Software: VS Code/MacDown/Typora/Vim
# @Author  : Di Wang
# @Email   : sdcswd@gmail.com
```

> See [epics examples](/posts/epics-examples)

## 功能
### save
- 把record的某个field值保存到某个文件中(`.sav`文件)
- 对于vxWorks, 涉及nfs的处理
- 要选择保存哪些field, 用`.req`文件来指定, 或者用record的info field. 例: `info(autosaveFields, VAL)`
- 处理`.req`文件的路径以及`.sav`文件的路径
- 对于`.sav`文件, 还要处理备份

### restore
- 在iocInit过程中, 从文件中恢复值.
- 涉及到iocInit过程中的哪个阶段(pass 0 or pass 1)恢复. autosave使用术语position和setting来区分pass0和pass1, 但显然不如直接使用pass0和pass1方便记忆.
- 处理不同情况的restore, 如waveform, record link.

## 编译

在`$(HOME)/epics/R7.0.8/modules`目录下编译autosave之后, 在`$(HOME)/epics/R7.0.8/ioc`目录下创建autosave ioc, 然后创建`RELEASE.local`文件.
```makefile
# inside Release.local
MODULES = $(TOP)/../../modules
AUTOSAVE = $(MODULES)/autosave-R5-11
```
然后在ioc app src目录下的 Makefile中:
```makefile
xxx_LIBS += autosave
xxx_DBD += asSupport.dbd
```

## 用法
以下为e3中的配置. 所有需要save and restore的record都在db文件中添加对应的info field, 也是比较推荐的用法, 可以避免手写`req`文件. 注意`afterInit`命令由`require` module提供.

```shell
#- Define autosave environment variables from shell environment, with suitable defaults

#- AUTOSAVE_INCOMPLETE      - Optional: Ok to save/restore save sets with missing values?
#-                            Default: 1
#- AUTOSAVE_CA_RECONNECT    - Optional: Retry connecting to PVs whose initial connection
#-                                      attempt failed?,  Default: 1
#- AUTOSAVE_DATED_BACKUP    - Optional: Save dated backup files?
#-                            Default: 1
#- AUTOSAVE_NUM_SEQ         - Optional: Number of sequenced backup files to write
#-                            Default: 3
#- AUTOSAVE_SEQ_PERIOD      - Optional: Time interval in seconds between sequenced backups
#-                            Default: 300
#- AUTOSAVE_VALUES_FILES_PASS0    - Optional: Filename, Default: values_pass0
#- AUTOSAVE_VALUES_FILES_PASS1    - Optional: Filename, Default: values_pass1
#- AUTOSAVE_SETTINGS_FILES        - Optional: Filename, Default : settings
#- AUTOSAVE_VALUES_PERIOD_PASS0   - Optional, Default 5
#- AUTOSAVE_VALUES_PERIOD_PASS1   - Optional, Default 10
#- AUTOSAVE_SETTINGS_PERIOD       - Optional, Default 5
#- AUTOSAVE_DEBUG_MODE            - Optional, Default 0

set_savefile_path    ("$(AS_TOP)/$(IOCDIR)", "/save")
set_requestfile_path ("$(AS_TOP)/$(IOCDIR)", "/req")

#- If there is no path, create it
system("install -d $(AS_TOP)/$(IOCDIR)/save")
system("install -d $(AS_TOP)/$(IOCDIR)/req")

save_restoreSet_status_prefix("$(IOCNAME):as-")

#- Debug-output level
save_restoreSet_Debug($(AUTOSAVE_DEBUG_MODE=0))

#- Ok to save/restore save sets with missing values (no CA connection to PV)?
save_restoreSet_IncompleteSetsOk($(AUTOSAVE_INCOMPLETE=1))

#- Tell save_restore to writed dated backup files.
save_restoreSet_DatedBackupFiles($(AUTOSAVE_DATED_BACKUP=1))

#- Tell save_restore to save sequence files.
#- save_restore to to maintain three copies of each *.sav file
save_restoreSet_NumSeqFiles($(AUTOSAVE_NUM_SEQ=3))

#- at 5 minute intervals
save_restoreSet_SeqPeriodInSeconds($(AUTOSAVE_SEQ_PERIOD=300))

#- should periodically retry connecting to PVs whose initial connection attempt failed
save_restoreSet_CAReconnect($(AUTOSAVE_CA_RECONNECT=1))

#- Time interval in seconds between forced save-file writes.  (-1 means forever).
#- This is intended to get save files written even if the normal trigger mechanism is broken.
save_restoreSet_CallbackTimeout(-1)
#- pass0 : save files are to be restored before record initialization
set_pass0_restoreFile("$(AUTOSAVE_SETTINGS_FILES=settings).sav")
set_pass0_restoreFile("$(AUTOSAVE_VALUES_FILES_PASS0=values_pass0).sav")
#- pass1 : save files are to be restored after record initialization
set_pass1_restoreFile("$(AUTOSAVE_SETTINGS_FILES=settings).sav")
set_pass1_restoreFile("$(AUTOSAVE_VALUES_FILES_PASS1=values_pass1).sav")

dbLoadRecords("save_restoreStatus.db", "P=$(IOCNAME):as-, DEAD_SECONDS=5")

#- Note afterInit supplied by require
afterInit("makeAutosaveFileFromDbInfo('$(AS_TOP)/$(IOCDIR)/req/$(AUTOSAVE_SETTINGS_FILES=settings).req','autosaveFields')")
afterInit("makeAutosaveFileFromDbInfo('$(AS_TOP)/$(IOCDIR)/req/$(AUTOSAVE_VALUES_FILES_PASS0=values_pass0).req','autosaveFields_pass0')")
afterInit("makeAutosaveFileFromDbInfo('$(AS_TOP)/$(IOCDIR)/req/$(AUTOSAVE_VALUES_FILES_PASS1=values_pass1).req','autosaveFields_pass1')")

#- Note afterInit supplied by require
#- We don't need PREFIX, because generated req files has the hard-code PV names from DBInfo : info tag
afterInit("create_monitor_set('$(AUTOSAVE_SETTINGS_FILES=settings).req','$(AUTOSAVE_SETTINGS_PERIOD=5)')")
afterInit("create_monitor_set('$(AUTOSAVE_VALUES_FILES_PASS0=values_pass0).req','$(AUTOSAVE_VALUES_PASS0_PERIOD=5)')")
afterInit("create_monitor_set('$(AUTOSAVE_VALUES_FILES_PASS1=values_pass1).req','$(AUTOSAVE_VALUES_PASS1_PERIOD=10)')")
```

## 问题

**Q**: `save_restoreStatus.db`中的record是做什么的?

**A**: 用一些record来表示autosave的运行状态与报错信息. 具体是在c程序中调用`ca_put()`来写入这些PV. 实现上非常不优雅, 不理解为什么非要用CA. 最大的优点也许就是可以用单独一个ioc来收集所有使用autosave的ioc的状态信息. 但真的有必要吗?
```shell
as-SR_heartbeat
as-SR_i_am_alive
as-SR_deadIfZero
as-SR_disable
as-SR_disableMaxSecs
as-SR_rebootStatus
as-SR_status
as-SR_recentlyStr
as-SR_rebootStatusStr
as-SR_rebootTime
as-SR_statusStr
as-SR_0_State * 8
as-SR_0_Status * 8
as-SR_0_StatusStr * 8
as-SR_0_Name * 8
as-SR_0_Time * 8
```

**Q**: `sav`文件, 默认生成5个(`setting.sav`, `setting.sav0`, `setting.sav1`, `setting.sav2`, `setting.savB`), 的创建逻辑与restore时的选择逻辑是什么?

**A**: **TODO**

## 其它
### autosaveBuild
比如有`savvy.db`, 里面定义了若干record, 又有`savvy_pass1.req`文件, 这样在`st.cmd`中就需要用两行命令:
```shell
dbLoadRecords("/path/to/savvy.db", "P=xxx")
create_monitor_set("savvy_pass1.req", 30, "P=xxx")
# 或者, 这样写
# create_monitor_set("auto_settings.req", 30, "P=xxx")
# 其中auto_settings.req文件中包含:
# file savvy_pass1.req P=xxx
```
这就导致如果多组(db, req)文件对, 要修改的话就很容易出错.

可以使用这条命令:
`autosaveBuild("built_settings.req", "_pass1.req", 1)`
这样`dbLoadRecords($(db_name).db)`命令调用后, autosave会自动搜索`$(db_name)_pass1.req`, 然后找到的话, 把`req`文件加入到`built_settings.req`文件中. 最后只需要处理`built_settings.req`文件就可以.

### asVerify
```shell
$ bin/linux-x86_64/asVerify -h
usage: asVerify [-vr] <autosave_file>
         -v (verbose) causes all PV's to be printed out
             Otherwise, only PV's whose values differ are printed.
         -r (restore_file) causes restore files named
            '<autosave_file>.asVerify' and '...B'to be written.
         -d (debug) increment debug level by one.
         -rv (or -vr) does both
examples:
    asVerify auto_settings.sav
        (reports only PVs whose values differ from saved values)
    asVerify -v auto_settings.sav
        (reports all PVs, marking differences with '***'.)
    asVerify -vr auto_settings.sav
        (reports all PVs, and writes a restore file.)
    asVerify auto_settings.sav
    caput <myStatusPV> $?
        (writes number of differences found to a PV.)

NOTE: For the purpose of writing a restore file, you can specify a .req
file (or any file that contains PV names, one per line) instead of a
.sav file.  However, this program will misunderstand any 'file' commands
that occur in a .req file.  (It will look for a PV named 'file'.)
```