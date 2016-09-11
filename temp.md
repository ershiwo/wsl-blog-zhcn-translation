The Windows Subsystem for Linux includes kernel mode drivers (lxss.sys and lxcore.sys) that are responsible for handling Linux system call requests in coordination with the Windows NT kernel. The drivers do not contain code from the Linux kernel but are instead a clean room implementation of Linux-compatible kernel interfaces. On native Linux, when a syscall is made from a user mode executable it is handled by the Linux kernel. On WSL, when a syscall is made from the same executable the Windows NT kernel forwards the request to lxcore.sys.  Where possible, lxcore.sys translates the Linux syscall to the equivalent Windows NT call which in turn does the heavy lifting.  Where there is no reasonable mapping the Windows kernel mode driver must service the request directly.

As an example, the Linux fork() syscall has no direct equivalent call documented for Windows. When a fork system call is made to the Windows Subsystem for Linux, lxcore.sys does some of the initial work to prepare for copying the process. It then calls internal Windows NT kernel APIs to create the process with the correct semantics, and completes copying additional data for the new process.
File system

File system support in WSL was designed to meet two goals.

    Provide an environment that supports the full fidelity of Linux file systems
    Allow interoperability with drives and files in Windows

The Windows Subsystem for Linux provides virtual file system support similar to the real Linux kernel. Two file systems are used to provide access to files on the users system: VolFs and DriveFs.
VolFs

VolFs is a file system that provides full support for Linux file system features, including:

    Linux permissions that can be modified through operations such as chmod and chroot
    Symbolic links to other files
    File names with characters that are not normally legal in Windows file names
    Case sensitivity

Directories containing the Linux system, application files (/etc, /bin, /usr, etc.), and users Linux home folder, all use VolFs.

Interoperability between Windows applications and files in VolFs is not supported.
DriveFs

DriveFs is the file system used for interoperability with Windows. It requires all files names to be legal Windows file names, uses Windows security, and does not support all the features of Linux file systems. Files are case sensitive and users cannot create files whose names differ only by case.

All fixed Windows volumes are mounted under /mnt/c, /mnt/d, etc., using DriveFs. This is where users can access all Windows files. This allows users to edit files with their favorite Windows editors such as Visual Studio Code, and manipulate them with open source tools in Bash using WSL at the same time.