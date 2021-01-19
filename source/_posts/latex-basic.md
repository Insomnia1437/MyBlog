---
title: LaTeX 基础语法
date: 2020-12-28 15:38:28
categories:
  - MEMO
  - LaTex
tags:
  - Chinese
---

```python
# @Time    : 2020-12-28
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```
> 因为要准备写大论文了，之前写 [JACoW](https://www.jacow.org/) 论文都是直接拿template往里填充，对TeX语法也并不熟悉，决定稍微学习下，文章基于[overleaf](https://www.overleaf.com/learn)的教程和`<latex入门-简版>-刘海洋`。

<!-- more -->

## 基础概念

类似于HTML 和 CSS , `tex`文件可以有两种格式定义：`class`和`style`，前者是针对一个特定文档类型，后者则主要是添加某种feature。

## BibTeX

参考文献管理是个麻烦事。
BibTeX涉及两种格式：`bst`和`bib`。

`bst`代表`Bibliography Style`，和`tex`文件的`style`类似，定义了参考文献的格式，比如期刊名怎么缩写，作者名字是名在前还是姓在前等等。

`bib`文件就是一个*参考文献数据库文件*，在一个`bib`文件里放一堆reference就构成了一个数据库，然后需要在文字哪个部分用哪个就在`tex`正文中用`\cite{label}`命令去引用，其中label是自定义的某个文献的别名。不引用的话默认不会出现在PDF文件中，除非用了`\nocite{label}`，表示“我没有在正文中引用这个文献，但依然要在文末的reference section里显示它”。

大致了解LaTeX的都知道，需要多次编译，过程如下。
```
(xe/pdf)latex foo.tex   # 表示使用 latex, pdflatex 或 xelatex 编译
bibtex foo.aux
(xe/pdf)latex foo.tex
(xe/pdf)latex foo.tex
```

为什么需要四次编译？[具体细节参考这篇](https://liam.page/2016/01/23/using-bibtex-to-generate-reference/)

1. LaTeX读取`tex`文件，找里面出现的`cite`命令，然后创建一个`aux`后缀的辅助文件，表示要引用某个label的文献，但此时还不知道是哪个文献。
2. 由BibTex读取`aux`文件，去参考文献数据库里找名为label的文献，然后生成一个`bbl`文件，bbl文件里就是latex格式的`bibitem`列表。
3. LaTeX这时候去读`bbl`文件，把文献列表在PDF中显示，然后更新`aux`文件，建立label和文献编号的关系。
4. 最后一步，再读取`aux`文件，把正文中引用部分的label也替换成编号。

`bib`文件大概长这样：
```latex
@book{adams1995hitchhiker,
  title={The Hitchhiker's Guide to the Galaxy},
  author={Adams, D.},
  isbn={9781417642595},
  url={http://books.google.com/books?id=W-xMPgAACAAJ},
  year={1995},
  publisher={San Val}
}
```
而通过bibtex编译生成的`bbl`文件长这样：
```latex
\begin{thebibliography}{9} % Use for 1-9 references
	\bibitem{SuperKEKB}
	    Y. Ohnishi et al., 
	    \textquotedblleft{Accelerator design at SuperKEKB}\textquotedblright,
        \emph{Prog. Theor. Exp. Phys.}, 2013, 03A011.
    ...
	\end{thebibliography}
```
当然也可以直接在`tex`文件里自己写`bibitem`，不过写的多了估计就意识到为什么要发明BibTeX这个东西了。

## 基础语法

在前言（preamble）中添加一些设置值，然后`\maketitle`，正文中的一些基础语法，斜体，加黑，下划线，插入图片表格，标签以及如何引用。

数学公式可以是`inline mode`或者`displayed mode`，后者可以添加公式编号。

```latex
\documentclass[12pt, letterpaper, twoside]{article}
\usepackage[utf8]{inputenc}
\usepackage{graphicx}
\graphicspath{ {images/} }

\usepackage{amsmath}

\title{First document}
\author{Hubert Farnsworth \thanks{funded by the Overleaf team}}
\date{February 2017}

\begin{document}

\maketitle
\tableofcontents

\begin{abstract}
This is a simple paragraph at the beginning of the 
document. A brief introduction about the main subject.
\end{abstract}

We have now added a title, author and date to our first \LaTeX{} document!

Some of the \textbf{greatest}
discoveries in \underline{science} 
were made by \textbf{\textit{accident}}.

\begin{figure}[h]
    \centering
    \includegraphics[width=0.25\textwidth]{mesh}
    \caption{a nice plot}
    \label{fig:mesh1}
\end{figure}

As you can see in the figure \ref{fig:mesh1}, the 
function grows near 0. Also, in the page \pageref{fig:mesh1} 
is the same example.

\begin{itemize}
  \item The individual entries are indicated with a black dot, a so-called bullet.
  \item The text in the entries may be of any length.
\end{itemize}

\begin{enumerate}
  \item This is the first entry in our list
  \item The list numbers increase with each entry we add
\end{enumerate}

The mass-energy equivalence is described by the famous equation
\[ E=mc^2 \]
discovered in 1905 by Albert Einstein. 
In natural units ($c = 1$), the formula expresses the identity
\begin{equation}
E=m
\end{equation}

\begin{center}
 \begin{tabular}{||c c c c||} 
 \hline
 Col1 & Col2 & Col2 & Col3 \\ [0.5ex] 
 \hline\hline
 1 & 6 & 87837 & 787 \\ 
 \hline
 2 & 7 & 78 & 5415 \\
 \hline
 3 & 545 & 778 & 7507 \\
 \hline
 4 & 545 & 18744 & 7560 \\
 \hline
 5 & 88 & 788 & 6344 \\ [1ex] 
 \hline
\end{tabular}
\end{center}

\end{document}
```

### 对于分段：

```latex
0	\chapter{chapter}
1	\section{section}
2	\subsection{subsection}
3	\subsubsection{subsubsection}
4	\paragraph{paragraph}
5	\subparagraph{subparagraph}
```

### 对于特殊字符：

| 字符 | 功能             | 打印字符本身                 |
| ---- | ---------------- | ---------------------------- |
| #    | 宏参数           | `\#`                         |
| $    | 数学公式         | `$`                          |
| %    | 注释             | `\%`                         |
| ^    | 公式上标         | `\^{}` or `\textasciicircum` |
| &    | 表格中分列       | `\&`                         |
| _    | 公式下标         | `\_`                         |
| { }  | processing block | `\{ \}`                      |
| ~    | 不可             | `\textasciitilde` or `\~{}`  |
| \    | 符号后接命令     | `\textbackslash`             |

### 自定义命令：

对于`\newcommand{\plusbinomial}[3][2]{(#2 + #3)^#1}`，`[3]`代表总共三个参数，`[2]`代表第一个参数的默认值。

```latex
\newcommand{\plusbinomial}[3][2]{(#2 + #3)^#1}

To save some time when writing too many expressions 
with exponents is by defining a new command to make simpler:

\[ \plusbinomial{x}{y} \]

And even the exponent can be changed

\[ \plusbinomial[4]{y}{y} \]
```

### 自定义environment：

以下示例用了两种方法定义了带编号的environment

```latex
%In the preamble
---------------------------------
%Numbered environment
\newcounter{example}[section]
\newenvironment{example}[1][]{\refstepcounter{example}\par\medskip
   \noindent \textbf{Example~\theexample. #1} \rmfamily}{\medskip}


%Numbered environment defined with Newtheorem
\usepackage{amsmath}
\newtheorem{SampleEnv}{Sample Environment}[section]
--------------------------------------------------------------------


\begin{example}
User-defined numbered environment
\end{example}

\begin{SampleEnv}
User-defined environment created with the \texttt{newtheorem} command.
\end{SampleEnv}
```

## 自定义class

一个class文件的通常分为四个部分：

### identification

定义了版本，class的名称为`Thesis`，时间。

```latex
\NeedsTeXFormat{LaTeX2e}[1996/12/01]
\ProvidesClass{Thesis}
              [2007/22/02 v1.0
   LaTeX document class]
```

### preliminary declarations

在这个部分定义新命令，引入继承的基础class，引入别的package，以及一些其他声明。

### options

`\DeclareOption`命令接受两个参数，第一个代表option，第二个代表如果接受这个option需要执行的代码。

```latex
\DeclareOption{onecolumn}{\OptionNotUsed}
\DeclareOption{green}{\renewcommand{\headlinecolor}{\color{green}}}
\DeclareOption{red}{\renewcommand{\headlinecolor}{\color{slcolor}}}
\DeclareOption*{\PassOptionsToClass{\CurrentOption}{article}}
\ProcessOptions\relax
\LoadClass[twocolumn]{article}
```

### more declarations

其他的一些命令定义等。