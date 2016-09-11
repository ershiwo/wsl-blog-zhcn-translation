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
WSL 包含内核态驱动（lsxx.sys 和 lxcore.sys），他们负责在 Windows NT 内核中协调处理Linux 的系统调用请求。该驱动不包含任何来自 Linux 内核的代码，而是完全重写的一个干净的兼容 Linux 内核的界面。在原生 Linux 上，一个由用户界面产生并执行的系统调用会被 Linux 内核处理；而在 WSL 中，同样的工作由 Windows NT 内核转发给 `lxcore.sys` 执行。某些情况下 `lxcore.sys` 把 Linux 系统调用翻译为对应的 Windows NT 调用反而带来了沉重的负担，在没有对应映射的地方 Windows 内核态驱动不得不直接处理那些请求。
举个例子，Linux 中的 `fork()` 就没有对应的 Windows 调用方法。当向 WSL 发起该调用时，`lxcore.sys` 会为复制进程进行初始化准备工作。然后它会调用内部的 Windows NT 内核 API 中的正确语法创建进程，并完成为新进程复制额外数据的任务。

##文件系统

WSL 中的文件系统支持被设计为实现下列两个目标：

1. 提供一个完整还原的 Linux 文件系统；
2. 允许与 Windows 间进行驱动器和文件的互通。

WSL 提供了一个可以模拟真实 Linux 内核的虚拟文件系统支持。可以通过以下两种方式访问用户系统上的文件：`VolFs` 和 `DriveFs`。

###VolFs

VolFs 可以提供对 Linux 文件系统的完整支持，包括：

- 可通过如 `chmod` 和 `chroot` 等程序进行修改的 Linux 权限控制；
- 文件符号链接；
- 支持被 Windows 认为不合法的文件名字符；
- 大小写敏感。

Linux 系统，应用程序文件（`/etc`，`/bin`，`/usr`，etc.）以及用户的 Linux 家目录都使用了 VolFs。
在 VolFs 中不支持与 Windows 程序及文件的互通。

###DriveFs

DriveFs 是设计用来实现与 Windows 互通的文件系统。它要求所有文件名必须是合法的 Windows 文件名，使用 Windows 安全设置，并且不支持所有 Linux 文件系统的特点。文件名大小写敏感，但用户不能创建名称中只有大小写不同的文件。
当前所有 Windows 卷均以 `/mnt/c`，`/mnt/d`…… 的形式挂载，它们使用 DriveFs。用户可以在这里访问全部 Windows 下的文件。这里允许用户同时使用他们喜爱的 Windows 编辑器（如 Visual Studio Code） 和 Bash 下的开源工具操作文件。

#EOF

![video](https://sec.ch9.ms/ch9/ad03/33a90710-0d66-4c48-8f7f-db974771ad03/WSFLArchitectureDeepuThomas_mid.mp4)
视频是 Deepu Thomas 和 Seth Juarez 讨论关于 WSL 的底层结构。
