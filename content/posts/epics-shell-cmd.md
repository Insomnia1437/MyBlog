---
title: "epics shell commands"
date:  2024-05-25T12:34:12+02:00
# weight: 1
# aliases: ["/first"]
tags: ["English", "EPICS"]
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

provided shell commands in `7.0.8`:
```shell
epics> help
#               ClockTime_Report                asDumpHash      asInit
asSetFilename   asSetSubstitutions              ascar           asdbdump
asphag          aspmem          asprules        aspuag          astac
callbackParallelThreads         callbackQueueShow
callbackSetQueueSize            casr            cd              coreRelease
date            dbCreateAlias   dbDumpBreaktable                dbDumpDevice
dbDumpDriver    dbDumpField     dbDumpFunction  dbDumpLink      dbDumpMenu
dbDumpPath      dbDumpRecord    dbDumpRecordType
dbDumpRegistrar dbDumpVariable  dbLoadDatabase  dbLoadGroup     dbLoadRecords
dbLoadTemplate  dbLockShowLocked                dbNotifyDump    dbPutAttribute
dbPvdDump       dbPvdTableSize  dbReportDeviceConfig            dbStateClear
dbStateCreate   dbStateSet      dbStateShow     dbStateShowAll  dba
dbap            dbb             dbc             dbcar           dbd
dbel            dbgf            dbgl            dbgrep          dbhcr
dbior           dbjlr           dbl             dbla            dbli
dblsr           dbnr            dbp             dbpf            dbpr
dbpvar          dbs             dbsr            dbstat          dbtgf
dbtpf           dbtpn           dbtr            dlload          echo
eltc            epicsEnvSet     epicsEnvShow    epicsEnvUnset
epicsMutexShowAll               epicsParamShow  epicsPrtEnvParams
epicsThreadResume               epicsThreadShow epicsThreadShowAll
epicsThreadSleep                errlog          errlogInit      errlogInit2
exit            generalTimeReport               gft             help
installLastResortEventProvider  iocBuild        iocInit         iocLogInit
iocLogPrefix    iocLogShow      iocPause        iocRun          iocshCmd
iocshLoad       iocshRun        on              pft             postEvent
pval            pvasr           pwd             refdiff         refmon
refsave         refshow         registerAllRecordDeviceDrivers
registryDeviceSupportFind       registryDriverSupportFind       registryDump
registryFunctionFind            registryRecordTypeFind
scanOnceQueueShow               scanOnceSetQueueSize            scanpel
scanpiol        scanppl         setIocLogDisable
softIocPVA_registerRecordDeviceDriver           startPVAServer  stopPVAServer
system          taskwdShow      tpn             var

Type 'help <command>' to see the arguments of <command>.  eg. 'help db*'
```

#### \#
Note that, trailing comment does not work as you thought inside iocsh. if you run command like `outputWhenHigherThan(12, 10)  # output 10 when value > 12`, do not be too surprised with the result.
```shell
epics> help #

# newline-terminated comment
epics> echo #123
#123
epics> echo # 123
#
epics> echo # echo 123 > test.log
epics> system "cat test.log"
#
```

#### access security related
```shell
epics> as
asDumpHash          asSetFilename       ascar               asphag              asprules
astac               asInit              asSetSubstitutions  asdbdump            aspmem
aspuag
```

#### db related
- dba, db address
- dbla, db list alias
- dbl, db list
- **dbgrep**, db search
- **dbgf**
- **dbpf**
- **dbpr**
- **dbtr**, process record and print record
- dbnr, print number of records
- dbb, set breakpoint
- dbd, delete breakpoint
- dbs, single step
- dbc, continue
- dbp, print suspended record
- dbap, auto print record that has a breakpoint when processed
- dbstat, print list of suspended records, and breakpoints set in locksets
```shell
# db address
epics> dba calc
Record Address: 0x5598f77c66f0 Field Address: 0x5598f77c69a0 Field Description: 0x5598f7752290
   No Elements: 1
   Record Type: calc
    Field Type: 10 = DBF_DOUBLE
    Field Size: 8
       Special: 0
DBR Field Type: 10 = DBR_DOUBLE
epics> dbCreateAlias pdbbase calc clac:alias
epics> dbla
clac:alias -> calc
epics> dbtr ai
ACKS: MINOR         ACKT: YES           ADEL: 0             AFTC: 0
AFVL: 0             ALST: 13            AMSG:               AOFF: 0
ASG : private       ASLO: 1             BKPT: 00            DESC:
DISA: 0             DISP: 0             DISS: NO_ALARM      DISV: 1
DTYP: Soft Channel  EGU :               EGUF: 0             EGUL: 0
EOFF: 0             ESLO: 1             EVNT:               FLNK: CONSTANT
HHSV: MAJOR         HIGH: 80            HIHI: 90            HOPR: 0
HSV : MINOR         HYST: 1             INIT: 0             INP : CONSTANT
LALM: 20            LBRK: 0             LCNT: 0             LINR: NO CONVERSION
LLSV: MAJOR         LOLO: 10            LOPR: 100           LOW : 20
LSV : MINOR         MDEL: 0             MLST: 13            NAME: ai
NAMSG:              NSEV: NO_ALARM      NSTA: NO_ALARM      ORAW: 0
PACT: 0             PHAS: 0             PINI: RUNNING       PREC: 3
PRIO: LOW           PROC: 0             PUTF: 0             ROFF: 0
RPRO: 0             RVAL: 0             SCAN: Passive       SDIS: CONSTANT
SDLY: -1            SEVR: MINOR         SIML: CONSTANT      SIMM: NO
SIMS: NO_ALARM      SIOL: JSON_LINK {const: 3.14159}        SMOO: 0
SSCN: 65535         STAT: LOW           SVAL: 3.14159
TIME: 2024-05-25 13:20:57.409527694     TPRO: 0             TSE : 0
TSEL: CONSTANT      UDF : 0             UDFS: INVALID       UTAG: 0
VAL : 13
```

#### error log
- eltc, Control display of error log messages on console
- errlog, Send a message to the log server
- errlogInit, errlogInit2

#### hardware related
- **dbior**
- dbhcr

#### scan related
- scanOnceQueueShow
- scanOnceSetQueueSize
- scanpel, Print Event Lists
- scanpiol, Print I/O Event Lists
- scanppl, Print Periodic Lists

#### Time related
- generalTimeReport
- ClockTime_Report

```shell
epics> help installLastResortEventProvider

installLastResortEventProvider

Installs the optional Last Resort event provider at priority 999,
which returns the current time for every event number
epics> help ClockTime_Report

ClockTime_Report interest_level

Reports the IOC's OS clock synchronization status:
  - On VxWorks and RTEMS when synchronized:
      * Synchronization time provider priority
      * Initial and last synchronization times
      * Synchronization interval
  - Otherwise:
      * Program start time
```
#### CA related
- casr, Channel Access Server Report
- dbel, prints the Channel Access event list for the speciﬁed record
- dbcar, Shows status of Channel Access links (CA_LINK)
```shell
epics> casr 4
Channel Access Server V4.13
No clients connected.
CAS-TCP server on 0.0.0.0:5064 with
    CAS-UDP name server on 0.0.0.0:5064
        Last name requested by 172.29.165.14:43519:
        User '', V4.13, Priority = 0, 0 Channels
Sending CAS-beacons to 1 address:
    172.29.175.255:5065
Free-lists total 350120 bytes, comprising
    7 client(s), 512 channel(s), 512 monitor event(s), 0 putNotify(s)
    16 small (16384 byte) buffers, 4294967295 jumbo (10000024 byte) buffers
Server resource id table:
    Bucket entries in use = 0 bytes in use = 32800
    Bucket entries/hash id - mean = 0.000000 std dev = 0.000000 max = 0
epics> dbel ai 2
1 PV Event Subscriptions ( monitors ).
 VAL { VALUE ALARM }, thread=0x7f190c000f60, queue empty
```
#### PVA related
- pval
- pvasr
- refdiff, reference counter
- refmon
- refsave
- refshow

#### Test related
- dbtgf, test get field; Old: gft
- dbtpf, test put field; Old: pft
- dbtpn, test Process notify; Old: tpn
- dblsr, lock set report
- dbLockShowLocked

#### Miscellaneous
- epicsParamShow
- epicsPrtEnvParams
- epicsEnvShow
- **on**, ??????? how to use?
- iocshCmd, run ioc shell command
- iocshRun, run ioc shell command, with macro
- iocshLoad, load a file with macro

```shell
epics> help on

on 'error' 'continue' | 'break' | 'wait' [value] | 'halt'

Change IOC shell error handling.
  continue (default) - Ignores error and continue with next commands.
  break - Return to caller without executing futher commands.
  halt - Suspend process.
  wait - stall process for [value] seconds, then continue.
epics> on error halt
Interactive shell ignores  on error ...
```