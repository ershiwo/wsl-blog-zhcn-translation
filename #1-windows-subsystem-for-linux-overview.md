#Windows Subsystem for Linux Overview

>在 4 月 7 号推送的 build 14328 fast ring 中，我们迎来了 Bash on Ubuntu on Windows 这个功能。自 build 2016 微软向广大开发者宣布 Windows 将支持 Linux Bash 之后，我们终于得见这个功能的真容。与此同时也有不少人对其实现方式有所疑问。这篇文章翻译自 MSDN blog，Jack Hammons 将向大家介绍 Windows Subsystem for Linux 的一些基本情况。

我们刚刚公布了借助 Windows Subsystem for Linux（Windows 上运行的 Linux 子系统，WSL）在 Windows 上原生运行 Linux ELF64 二进制文件的 Bash on Ubuntu on Windows 项目。这个令人激动的子系统是由微软视窗内核团队打造的。最近这几天我们被问的最多的问题是：“你们搞得这个玩意儿和传统的虚拟机有什么区别？”在这个系列文章的第一篇中，我们将概述 WSL 的基本情况并回答前面提到的那些个问题。在未来的文章里我们会再深入介绍每一个模块区域。

*另外这篇文章是替 Deepu Thomas 发的。*

##Windows 子系统的历史

从一开始 Windows NT 就被设计为允许像 Win32 这样的环境子系统直接向应用程序提供可编程接口，并不需要由内核绑定执行细节。这就赋予 NT 内核天生支持 POSIX，OS/2 以及 Win32 子系统的能力。
早期的子系统仅能运行在基于其提供给应用程序的接口生成对应的 NT 系统调用的用户态下。所有应用程序都是 PE/COFF 可执行文件（通过子系统提供的 API 和 NTDLL 来运行 NT 系统调用的运行库和服务集合）。当某个用户态程序启动时，就会根据可执行文件头判断要加载哪个正确的子系统环境以供使用。
近几个版本的子系统为支持基于 Unix 的应用程序子系统（Subsystem for Unix-based Applications，SUA）而替换了 POSIX 层。这些用户态组件的集合要满足如下条件：

1. 进程和信号管理；
2. 终端管理；
3. 系统服务请求及内部进程通信。

SUA 的主要任务是帮助应用程序在不进行重大改动的情况下移植至 Windows 平台运行。它通过使用 NT 架构下的 POSIX 用户态 API 实现。鉴于这些组件都是运行在用户态基础上的，所以很难和类似 `fork()` 这样的内核态系统调用做到功能和性能上的相似。由于这种模式依赖重编译程序来满足不断增加的功能移植需求，对长期维护是不利的。
随着时间的推移，这些早期的子系统们退出了历史舞台。然而因为 Windows NT 内核被设计为允许加入新的子系统环境，我们才能使用这一部分的早期投入并拓展它们来开发 Windows Subsystem for Linux。

##Windows Subsystem for Linux

WSL 是一个可以让原生 Linux ELF64 二进制文件运行在 Windows 中的组件集合。它包含用户态和内核态组件，主要有以下部分：

1. 用户态会话管理服务：处理 Linux 实例生命周期；
2. 微型供应商驱动（Pico provider drivers）（lxss.sys，lxcore.sys）：翻译 Linux 系统调用以模拟 Linux 内核；
3. 微进程（Pico processes）：托管原生的用户态 Linux（例如 `/bin/bash`）.

这是在用户态 Linux 二进制文件和 Windows 内核组件中间诞生魔法的地方。通过向微进程中置入原生 Linux 二进制文件，我们使 Linux 系统调用被重定向至 Windows 内核中。`lxss.sys` 和 `lxcore.sys` 驱动负责将 Linux 系统调用翻译为 NT API，并模拟 Linux 内核。

![Figure 1: WSL Components](https://xxx.com/xx.png)

##LXSS 管理服务

LXSS 管理服务是 Linux 子系统的驱动代理，同时也是 `Bash.exe` 调起 Linux 二进制文件的途径。该服务也用于同步控制安装卸载过程，确保同时间只有一个进程进行此类操作，并且锁定 Linux 二进制文件直到操作挂起前都不被唤起。
由特定用户启动的所有 Linux 进程会进入同一个 Linux 实例里。该实例是一个数据结构，跟踪所有 Linux 进程，线程和运行时状态。它会在某个 NT 进程需要启动 Linux 二进制文件时自动创建。
一旦最后一个 NT 客户端关闭，相应的 Linux 实例也会随之终止。任何在该实例中启动的进程以及其守护进程（例如 `git` 的凭据缓存）都会被终止。

##微进程

作为 **Project Drawbridge** 的一部分，Windows 内核使用了“微进程”和“微驱动”的概念。“微进程”指不触发与子系统有关系的 OS 服务的 OS 进程，比如 Win32 进程环境块（Win32 Process Environment Block，PEB）。此外，对于某个微进程，系统调用和用户态的异常都会分配到对应的驱动中。
微进程和驱动提供了 WSL 的运行基础，通过向微进程地址空间内载入可执行 ELF 二进制文件并在系统调用的 Linux 兼容层上执行它们来运行原生未经修改的 Linux 二进制文件。

##系统调用

WSL 通过在 Windows NT 内核上虚拟出一个 Linux 内核界面以执行原生 Linux ELF64 二进制文件。其中公开对外的内核界面就是系统调用（syscalls）。每个调用都是是由内核提供的服务，它们可以在用户态中被调用。Linux 内核和 Windows NT 内核都向用户态公开了数百个系统调用，但是它们的含义不同，通常也不能直接兼容。例如，Linux 内核包含像 `fork`，`open` 和 `kill` 这样的调用，而在 Windows NT 中对应的调用是 `NtCreateProcess`，`NtOpenFile` 和 `NtTerminateProcess`。
WSL 包换内核态驱动（lsxx.sys 和 lxcore.sys），
