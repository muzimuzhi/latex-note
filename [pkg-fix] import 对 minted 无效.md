# [pkg-fix] import 对 minted 无效

⟨⟩

## 前置知识

* `minted` 宏包（提供功能：使用 ` Pygments ` 格式化源码）
  * `\inputminted[⟨options⟩]{⟨language⟩}{⟨filename⟩}` 
    功能：根据指定的格式化选项 `⟨options⟩` 和编程语言 `⟨language⟩`，对文本文件 `⟨filename⟩` 对源码格式化（如语法高亮）后输出
* `import` 宏包（提供功能：引入子文件时只需提供相对于母文件的路径）
  * `\subimport{⟨path_extension⟩}{⟨file⟩}`
    功能：引入在路径 `⟨path_extension⟩` 下的文件 `⟨file⟩`，并使得在 `⟨file⟩` 的内部引入子文件时，只需提供子文件相对于  `⟨file⟩` 的路径
* `xpatch` 宏包（提供功能：给已定义命令打补丁）
  * `\xpatchcmd{⟨command⟩}{⟨search⟩}{⟨replace⟩}{⟨success⟩}{⟨failure⟩}`
    功能：在命令 `⟨command⟩` 的定义内部，将每一处内容 `⟨search⟩` 都替换为内容 `⟨replace⟩`，达到给命令 `⟨command⟩` 打补丁的效果。如果替换成功，则继续执行代码片段 `⟨success⟩`，否则执行代码片段 `⟨failure⟩`
* 其他，如宏展开等

## 问题复现

### 目录结构

```bash
.
├── main.tex
└── subdir
    ├── foo.tex
    ├── bar.tex
    └── demo.py
```

设置子目录 `subdir`，是为了让 `foo.tex` 和 `main.tex` 的绝对路径不同。

### 文件内容

* `main.tex` 引入子文件 `foo.tex` 和 `demo.py`
* `foo.tex` 引入子文件 `bar.tex` 和 `demo.py`

```latex
%% main.tex
\documentclass{article}
\usepackage{import}
\usepackage{minted}

\begin{document}
% 在 main 中，子文件使用相对于 main 的路径
\subimport{subdir/}{foo}
\inputminted{python}{subdir/demo.py}
\end{document}

%% subdir/foo.tex
\section{foo}
% 在 foo 中，子文件使用相对于 foo、而非 main 的路径
% （若不使用 import 宏包，此处的路径也必须是相对于 main 的）
\inputminted{python}{demo.py}
\input{bar}

%% subdir/bar.tex
\section{bar}

%% subdir/demo.py
import numpy as np
print(np.pi)
```

### 预期结果

预期的编译结果，应当与文件 `main-v2.tex` （见下方）的编译结果相同。注意 `main-v2.tex` 中没有子文件的嵌套引入，所有路径都是相对于（唯一的）主文件的。

```latex
%% main-v2.tex
\documentclass{article}
% \usepackage{import}
\usepackage{minted}

\begin{document}
\section{foo}
\inputminted{python}{subdir/demo.py}
\input{subdir/bar % equivalent to \section{bar}
\inputminted{python}{subdir/demo.py}
\end{document}
```

###遇到问题

```bash
(./subdir/foo.tex (./_minted-main/default-pyg-prefix.pygstyle)
(./_minted-main/default.pygstyle)Error: cannot read infile: [Errno 2] No such file or directory: 'demo.py'
system returned with code 256


! Package minted Error: Missing Pygments output; \inputminted was probably given a file that does not exist--otherwise, you may need the outputdir package option, or may be using an incompatible build tool, or may be using frozencache with a missing file.

See the minted package documentation for explanation.
Type  H <return>  for immediate help.
 ...                                              
```

简言，程序找不到文件 `foo.tex` 中要引入的子文件 `demo.py`。从输出的 PDF 来看，也是如此。如果把文件 `foo.tex` 中引入子文件 `demo.py` 时填写的路径，由 `demo.py`（相对于文件 `foo.tex` ）， 改成 `subdir/demo.py`（相对于文件 `main.tex` ），则能得到预期结果。

这说明，宏包`import` 提供的机制，对宏包 `minted` 提供的命令 `\inputminted` 是无效的。

## 解决方式

修改命令 `\inputminted` 的定义，加入「公共路径」。

```latex
%% main.tex (revised)
\documentclass{article}
\usepackage{import}
\usepackage{minted}
\usepackage{xpatch}

\makeatletter
\xpatchcmd
    {\inputminted}
    {#3}
    {\import@path#3}
    {}{}
\makeatother

\begin{document}    
\subimport{subdir/}{foo}
\inputminted{python}{subdir/demo.py}
\end{document}
```

## 解决思路

兵分两路。第一路，找到命令 `\inputminted` 的定义。

```latex
%% minted.sty (excerpted)

\ifthenelse{\boolean{minted@draft}}%
  {\newcommand{\inputminted}[3][]{%
    \begingroup
    \minted@configlang{#2}%
    \setkeys{minted@opt@cmd}{#1}%
    \minted@fvset
    \VerbatimInput{#3}%
    \endgroup}}%
  {\newcommand{\inputminted}[3][]{%
    \begingroup
    \minted@configlang{#2}%
    \setkeys{minted@opt@cmd}{#1}%
    \minted@fvset
    \minted@pygmentize[#3]{#2}%
    \endgroup}}
```

第二路，从命令 `\subimport` 出发，寻找储存「公共路径」的宏。宏包 `import` 不长，为方便讲解，我们将文件 `import.sty` 的代码部分全文抄录如下。

```latex
%% import.sty 
\ProvidesPackage{import}[2009/03/23 \space  v 5.1]
\ProcessOptions

\@ifundefined{import}{%
 \newcommand{\import}{\global\let\import@path\@empty \@doimport\input}
}{
 \PackageWarning{import}{\string\import\space command is already defined!^^J%
   Defining only its \string\inputfrom\space alias.}
}
\newcommand{\inputfrom}{\global\let\import@path\@empty \@doimport\input}
\newcommand{\subimport}{\@doimport\input}
\newcommand{\subinputfrom}{\@doimport\input}
\newcommand{\includefrom}{\global\let\import@path\@empty \@doimport\include}
\newcommand{\subincludefrom}{\@doimport\include}

\def\@doimport#1{\@ifstar
  {\@sub@import#1\@iffileonpath}{\@sub@import#1\IfFileExists}}

% #1 = import command,  #2 = switch for *,  #3 = import path extension
\def\@sub@import#1#2#3{%
  \begingroup
  \protected@edef\@tempa{\endgroup
    \let\noexpand\IfFileExists\noexpand#2%
    \noexpand\@import  \noexpand#1%  param 1
      {\@ifundefined{input@path}{}{\input@path}}% 2
      {\@ifundefined{Ginput@path}{}{\Ginput@path}}% 3
      {\import@path#3}{\import@path}% 4,5
      {\ifx\IfFileExists\im@@IfFileExists \noexpand\im@@IfFileExists 
       \else \noexpand\IfFileExists \fi}}% 6
  \@tempa}
%
% #1 = import command (\input or \include)
% #2 = previous input path list. #3 = previous graphics input path list.
% #4 = full path added to each.  #5 = previous import path.  
% #6 = previous \IfFileExists.   #7 = file name.
%
\def\@import#1#2#3#4#5#6#7{%
  \gdef\import@path{#4}%
  \protected@edef\input@path{{\import@path@fix{#4}}#2}%
  \protected@edef\Ginput@path{{\import@path@fix{#4}}#3}%
  #1{#7}%
  \let\IfFileExists#6% restore after \import*
  \gdef\import@path{#5}%
  \def\input@path{#2}\ifx\input@path\@empty \let\input@path\@undefined \fi
  \def\Ginput@path{#3}\ifx\Ginput@path\@empty \let\Ginput@path\@undefined \fi
}

\let\im@@IfFileExists\IfFileExists
\gdef\import@path{}

\let\import@path@fix\@firstofone % default

% Check for vms file names and set \import@path@fix appropriately
\gdef\@gtempa{[]}
\ifx\@gtempa\@currdir % VMS directory syntax
 \gdef\import@path@fix#1{\@gobbleVMSbrack#1][>}
 \gdef\@gobbleVMSbrack#1][#2{#1\ifx>#2\@empty
    \expandafter \strip@prefix \fi % Gobble up to >
    \@gobbleVMSbrack #2}
\fi
```

然后我们来人工对宏 `\subimport` 进行展开。

```latex
% 原始调用
\subimport{subdir/}{bar}

% 展开 \subimport 一次
\@doimport\input{subdir/}{bar}

% 展开 \@doimport 一次，此处
%   #1 = \input
\@sub@import\input\IfFileExists{subdir/}{bar}

% 展开 \@sub@import 一次，此处
%   #1 = \input
%   #2 = \IfFileExists
%   #3 = subdir/

% 先读取参数，注意前面都是 \@tempa 的定义，只有最后一行是调用
\begingroup
\protected@edef\@tempa{\endgroup
  \let\noexpand\IfFileExists\noexpand\IfFileExists%
  \noexpand\@import  \noexpand\input%  param 1
    {\@ifundefined{input@path}{}{\input@path}}% 2
    {\@ifundefined{Ginput@path}{}{\Ginput@path}}% 3
    {\import@path subdir/}{\import@path}% 4,5
    {\ifx\IfFileExists\im@@IfFileExists \noexpand\im@@IfFileExists 
     \else \noexpand\IfFileExists \fi}}% 6
\@tempa{bar}
  
% 然后展开 \@tempa 一次，
% 注意到 \import@path 在宏包载入时被定义为空命令
\@import
  \input
  {}
  {}
  {subdir/}
  {}
  {\IfFileExists}
  {bar}
  
% 观察到「公共路径」"subdir/" 是作为第四个参数传入 \@import 的，
% 这也与 \@import 定义前的注释一致。

% 由 \@import 的定义知，\import@path 即是存储当前「公共路径」的宏
```

最后，我们使用 `xpatch` 提供的功能，（粗暴地）将 `\import@path` 添加到 `\inputminted` 的第三个参数之前。

```latex
\usepackage{xpatch}

\makeatletter
\xpatchcmd
    {\inputminted}
    {#3}
    {\import@path#3}
    {}{}
\makeatother
```

## 后记

寻找特定源文件，可以使用命令 `kpsewhich`：

```bash
$ kpsewhich import.sty
/usr/local/texlive/2018/texmf-dist/tex/latex/import/import.sty
```

在打补丁之前，我们可以通过输出命令 `\import@path` 的内容，来验证想法。

```latex
% 添加在 \subimport 之后的任何位置
\makeatletter
\import@path
\makeatother
% 若使用文中的例子，则在 PDF 里可以得到 "subdir/"
```

宏包 `import` 的做法，对所有通过命令 `\input` 和 `\includegraphics` 进行嵌套引入的子文件和图片，都是有效的。

因为在调用外部程序（ `Pygments `）时，不涉及命令 `\intput`，文件的「公共路径」也没有手动传入，宏包 `minted` 无法自动享受到 `import` 提供的便利。类似的不兼容问题，在使用其他调用外部程序的宏包时，也可能遇到。