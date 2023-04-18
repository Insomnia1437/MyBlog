---
title: A Short Survery on IDE and VCS
categories: Software
tags:
  - Essay
  - KEK
  - English
abbrlink: 46c470de
date: 2019-11-19 01:51:40
---

<!-- # A Short Introduction To VCS and IDE -->

```python
# @Time    : 2019-11-19
# @Language: Markdown
# @Software: Typora
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

------

## Overview

This essay will briefly introduce and compare several kinds of VCS software, IDE and popular text editors. I hope this article may help people in KEK who need to program (**Using Java, C/C++, Python**) and collaborate with others could select efficient and suitable tools easily.
<!-- more -->
The introduction would mainly based on three stages: 

- **Editor**: how to read the program source code with a more user friendly experience 

- **IDE**: how to improve programming efficiency rather than spending extra time on the library configuration or build tools when modifying the source code as your requirement 

- **VCS**: how to cooperate with others to maintain the project consistency and keep a detailed historical record of the different version of project.

**Term explanation**


> **I**ntegrated **D**evelopment **E**nvironment (IDE):
>
> An integrated development environment (IDE) is a software application that provides comprehensive facilities to computer programmers for software development. An IDE normally consists of at least a source code editor, build automation tools, and a debugger. [IDE](https://en.wikipedia.org/wiki/Integrated_development_environment)




> **V**ersion **C**ontrol **S**ystem (**VCS**):
>
> A component of software configuration management, version control, also known as revision control or source control, is the **<u>management of changes to documents, computer programs</u>**, large web sites, and other collections of information.  [Version control on Wikipedia](https://en.wikipedia.org/wiki/Version_control)


## Comparison of text editors

**Note:  Recent years, the boundary between Editor and IDE becomes obscure, many editors can also integrate with build tools, debugger, compiler or interpreter with some plugin configuration to become a lightweight IDE (though some configuration might cost time and be somewhat geek), thus, I only select several "traditional" text editors <u>in my mind</u>.**

------

**Table explanation**

- Platform: Include Windows, macOS and Linux
- Cost: Since the price varies between different plans, only personal license price is listed
- Repository stars on GitHub: This item may reflect the software's popularity

| Name               | Platform     | Cost   | Open source | Repository star on GitHub | Japanese support | Learning curve |
| ------------------ | ------------ | ------ | ----------- | ------------------------- | ---------------- | -------------- |
| Vim                | All          | Free   | Yes         | 18.4k                     | Yes              | Hard           |
| Visual Studio Code | All          | Free   | Yes         | 86.9k                     | Yes              | Easy           |
| Notepad++          | Windows only | Free   | Yes         | 9.9k                      | Yes              | Easy           |
| Sublime Text       | All          | $80*   | No          | N/A                       | plug-in          | Medium         |
| Atom               | All          | Free   | Yes         | 50.4k                     | No               | Easy           |
| Ultraedit          | All          | $79.95 | No          | N/A                       | Yes              | Easy           |

> *: Sublime Text may be downloaded and evaluated for free, however a license must be purchased for continued use.

## Comparison of IDE

**Table explanation**

| Name                    | Platform | Cost                       | Japanese support | Programming Language              |
| ----------------------- | -------- | -------------------------- | ---------------- | --------------------------------- |
| Microsoft Visual Studio | All      | Free for Community edition | plug-in          | C/C++, Python                     |
| IntelliJ IDEA           | All      | Free for Community edition | No               | Java                              |
| Eclipse                 | All      | Free                       | No               | Mainly Java, also PHP, Python etc |
| PyCharm                 | All      | Free for Community edition | No               | Python                            |

> Both IntelliJ IDEA and PyCharm belong to the JetBrains Company, the business edition can be purchased together with other IDEs (for different languages like PHP, .NET, Ruby, Go and so on) for **74,700 Yen** per year. See here: [JetBrains products](https://www.jetbrains.com/products.html)
>

## Comparison of VCS

**Functionality of VCS:**

- **Backup and Restore**:  Files are saved as they are edited, and you can jump to any moment in time.  
- **Synchronization**:  Lets people share files and stay up-to-date with the latest version. 
- **Short-term undo**: Go back to the “last known good” version in the database. 
- **Long-term undo**:  Jump back to the old version, and see what change was made that day. 
- **Track Changes**:  As files are updated, you can leave messages explaining why the change happened (stored in the VCS, not the file). This makes it easy to see how a file is evolving over time, and why. 
- **Track Ownership**:  A VCS tags every change with the name of the person who made it.  
- **Branching and merging**:  You can **branch** a copy of your code into a separate area and modify it in isolation (tracking changes separately). Later, you can **merge** your work back into the common area. 

**Basic concepts:**

- **Repository (repo)**: The database storing the files. 
- **Server**: The computer storing the repo.
- **Client**: The computer connecting to the repo.
- **Working Set/Working Copy**: Your local directory of files, where you make changes.
- **Trunk/Main**: The primary location for code in the repo. Think of code as a family tree — the trunk is the main line.
- **Add**: Put a file into the repo for the first time, i.e. begin tracking it with Version Control.
- **Revision**: What version a file is on (v1, v2, v3, etc.).
- **Head**: The latest revision in the repo.
- **Check out**: Download a file from the repo.
- **Check in**: Upload a file to the repository (if it has changed). The file gets a new revision number, and people can “check out” the latest one.
- **Check in Message**: A short message describing what was changed.
- **Changelog/History**: A list of changes made to a file since it was created.
- **Update/Sync**: Synchronize your files with the latest from the repository. This lets you grab the latest revisions of all files.
- **Revert**: Throw away your local changes and reload the latest version from the repository.

Here is some comparison of some common VCS software. 

**Table explanation** 

| Name             | Repository model | Learning curve | Storage method | Atomic operation | Partial checkout | Web Interface                                                |
| ---------------- | ---------------- | -------------- | -------------- | ---------------- | ---------------- | ------------------------------------------------------------ |
| Git              | Distributed      | Hard           | Snapshot       | Yes              | No               | GitLab, GitHub, Trac, Kallithea, Bitbucket etc |
| Subversion (SVN) | Client-server    | Easy           | Changeset      | Yes              | Yes              | Apache 2 module included, WebSVN, ViewSVN, ViewVC, Trac etc |
| Mercurial        | Distributed      | Medium         | Changeset      | Yes              | Yes              | Bitbucket, Trac, Kallithea                                   |
| CVS              | Client-server    | Easy           | Changeset      | No               | Yes              | cvsweb, ViewVC, others                                       |

> It might be confusing when user transfer from SVN to Git owing to the different meaning of some basic commands, like " update ".
>
> Noted that Mercurial shares some features with SVN as well as being a distributed system, and because of the similarities, the learning curve for those already familiar with SVN will be less steep.

