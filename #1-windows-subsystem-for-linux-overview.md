#Windows Subsystem for Linux Overview

>在 4 月 7 号推送的 build 14328 fast ring 中，我们迎来了 Bash on Ubuntu on Windows 这个功能。自 build 2016 微软向广大开发者宣布 Windows 将支持 Linux Bash 之后，我们终于得见这个功能的真容。与此同时也有不少人对其实现方式有所疑问。这篇文章翻译自 MSDN blog，Jack Hammons 将向大家介绍 Windows Subsystem for Linux 的一些基本情况。

我们刚刚公布了借助 Windows Subsystem for Linux（Windows 上运行的 Linux 子系统，WSL）在 Windows 上原生运行 Linux ELF64 二进制文件的 Bash on Ubuntu on Windows 项目。这个令人激动的子系统是由微软视窗内核团队打造的。最近这几天我们被问的最多的问题是：“你们搞得这个玩意儿和传统的虚拟机有什么区别？”在这个系列文章的第一篇中，我们将概述 WSL 的基本情况并回答前面提到的那些个问题。在未来的文章里我们会再深入介绍每一个模块区域。

另外这篇文章是替 Deepu Thomas 发的。

##Windows 子系统的历史
从一开始 Windows NT 就被设计为允许像 Win32 这样的环境子系统直接向应用程序提供可编程接口，并不需要由内核绑定执行细节。这就赋予 NT 内核天生支持 POSIX，OS/2 以及 Win32 子系统的能力。
早期的子系统仅能运行在基于其提供给应用程序的接口生成对应的 NT 系统调用的用户模式下。所有应用程序都是 PE/COFF 可执行文件（通过子系统提供的 API 和 NTDLL 来运行 NT 系统调用的运行库和服务集合）。当某个用户模式程序启动时，就会根据可执行文件头判断要加载哪个正确的子系统环境以供使用。
近几个版本的子系统为支持基于 Unix 的应用程序子系统（Subsystem for Unix-based Applications，SUA）而替换了 POSIX 层。这些用户模式元素的集合要满足如下条件：
	1. 进程和信号管理；
	2. 终端管理；
	3. 系统服务请求及内部进程通信。
SUA 的主要任务是帮助应用程序在不进行重大改动的情况下移植至 Windows 平台运行。
