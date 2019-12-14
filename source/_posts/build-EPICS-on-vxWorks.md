---
title: How to build EPICS on vxWorks
date: 2019-11-29 19:52:46
categories: 
  - "MEMO"
  - "EPICS"
tags:
  - "EPICS"
  - "vxWorks"
  - "KEK"
  - "English"
---

```python
# @Time    : 2019-11-29
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

## EPICS Base Installation Instruction

Check this site for the detail. [README](https://epics.anl.gov/base/R3-14/12-docs/README.html)

This MEMO will introduce how to build EPICS Base 3.14.12.5 on linux and run IOC on vxWorks, VME5500 .

If you have any problem of the basic structure of EPICS (e.g. confused about the Host ARCH and Target ARCH), please refer to the [**EPICS Application Developer's Guide**](https://epics.anl.gov/base/R3-14/12-docs/AppDevGuide/). (Though it may take several days to read :)

> Noted: the introduction of Wind River software installation will not be included.

### EPICS Base Release 3.14.12.5

Download EPICS Base file from the ANL site here [https://epics.anl.gov/base/R3-14/12.php](https://epics.anl.gov/base/R3-14/12.php)

```shell
mkdir base
cd base
tar xzf /yourdirectory/baseR3.14.12.5.tar.gz
cd base-3.14.12.5/
```
### Software requirements

- make (version 3.81 or later)
- Perl (version 5.8.1 or later)
- vxWorks (You must have vxWorks installed if any of your target systems are vxWorks systems.)
- gcc (version larger than 6.1 might have some problems. A discussion: [EPICS tech talk: Compiling EPICS 3.14.12.5 with GCC 6](https://epics.anl.gov/tech-talk/2016/msg01120.php))

### Set environment variables
- EPICS_BASE
```shell
setenv EPICS_BASE `env PERL5LIB=src/tools perl src/tools/fullPathName.pl .`
echo $EPICS_BASE
```
- EPICS_HOST_ARCH
```shell
setenv EPICS_HOST_ARCH `startup/EpicsHostArch`
echo $EPICS_HOST_ARCH
```
- readline 
If readline is not installed on your linux:
```shell
vi /configure/os/CONFIG_SITE.Common.linux-x86_64
# comment the following line
# COMMANDLINE_LIBRARY = READLINE
```
- Set vxWorks environment varibles
```shell
eval `/cont/VxWorks/vw68/wrenv.sh -p vxworks-6.8 -o print_env -f csh`
```
> Note: wrenv.sh might change the reference of `make` command, be careful to make sure the make version is larger than 3.81

### Do site-specific build configuration
Since there is no "vxWorks-ppc604" target architecture in base 3.14.12.5, we need to create it on our own.
```shell
cp -p configure/os/CONFIG_SITE.linux-x86_64.vxWorks-ppc60{3,4}
cp -p configure/os/CONFIG_SITE.linux-x86_64.vxWorks-ppc60{3,4}_long
```
Then modify the CONFIG_SITE file, set the target architecture.
```shell
vi configure/CONFIG_SITE
```
set `CROSS_COMPILER_TARGET_ARCHS=vxWorks-ppc604_long`

Set vxWorks
```shell
vi configure/os/CONFIG_SITE.Common.vxWorksCommon
```
Set these values
```
...
VXWORKS_VERSION = 6.8
# WIND_BASE is where you installed the Wind River software. In our case is:
WIND_BASE = /cont/VxWorks/vw683
...
WORKBENCH_VERSION = 3.2
UTILITIES_VERSION = 1.0
```
### Build EPICS base

It is recommended that using a log file to track your make process to help you find the error or warning.

In my case:
```shell
mkdir log
make | & tee log/base-3.14.12.5-make-linux64.txt
```
check the warnings and run tests
```shell
grep -i warning log/base-3.14.12.5-make-linux64.txt | wc
make runtests | & tee /log/base-3.14.12.5-test-linux64.txt
```

## Create IOC
Next we can create an IOC and run on vxWorks
```shell
cd ../base-3.14.12.5/..
mkdir -p test/vx
cd test/vx/
$EPICS_BASE/bin/$EPICS_HOST_ARCH/makeBaseApp.pl -t example myapp
$EPICS_BASE/bin/$EPICS_HOST_ARCH/makeBaseApp.pl -i -t example -p myapp myapp
```

```
# here you should input vxworks as your host_arch according to the prompt information. In my case:
The following target architectures are available in base:
    linux-x86_64
    vxWorks-ppc604_long
What architecture do you want to use? 
```

```shell
mkdir log
make | & tee log/base-3.14.12.5-app-linux64.txt
```

check the EPICS environment variables in this file:
`less iocBoot/iocmyapp/cdCommands`

Then telnet to your vxWroks, and go to this IOC's `iocBoot/iocmyapp/` directory, then load this IOC by `< st.cmd`
Using `dbl` to check whether IOC runs successfully or not.
