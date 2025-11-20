# Linux Kernel and Userspace Architecture

Understanding the separation between the kernel and userspace is fundamental to Linux security. This architecture enforces privilege separation, isolates processes, and protects the system's core from user-level applications. This security boundary is managed by the CPU and the kernel itself.

## Kernel Space vs. User Space

The Linux operating system divides its memory and execution environment into two distinct domains:

- **Kernel Space**: The heart of the OS, where the kernel code runs. It has unrestricted access to all hardware, memory, and system resources. The kernel is responsible for process scheduling, memory management, and handling hardware interrupts. Code running in kernel space operates with the highest level of privilege (Ring 0 on x86 architectures).

- **User Space**: The environment where user applications, utilities, and services run. Processes in user space have limited privileges and cannot directly access hardware or other processes' memory. To perform a privileged operation, a user space process must request it from the kernel. This is done via **system calls**.

This separation ensures that a misbehaving or compromised user application cannot bring down the entire system or access sensitive data from other processes.

## The System Call (Syscall) Interface

The **system call interface** is the mandatory gateway between user space and the kernel. When a user process needs to perform a privileged action—such as opening a file, sending data over the network, or creating a new process—it must execute a `syscall`.

1.  A user process prepares the necessary parameters (e.g., file path, network socket details) in CPU registers.
2.  It triggers a software interrupt (e.g., `int 0x80` on x86-32, or the `SYSCALL` instruction on x86-64).
3.  The CPU switches from a lower-privilege mode (e.g., Ring 3 for user space) to the highest-privilege mode (Ring 0 for kernel space).
4.  The kernel executes a specific syscall handler based on the requested syscall number.
5.  The kernel validates the parameters, performs the requested action, and checks for errors.
6.  The result is passed back to the user process, and the CPU switches back to the lower-privilege user mode.

### Security Implications of System Calls

Because syscalls are the only entry point into the kernel, the interface is a primary focus for security hardening:

- **Parameter Validation**: The kernel must rigorously validate all parameters passed from user space. A failure to do so can lead to vulnerabilities like buffer overflows or directory traversals within the kernel itself, resulting in a **kernel exploit** and full system compromise.
- **Syscall Filtering**: Security mechanisms like **seccomp** (Secure Computing Mode) and **SELinux** can restrict the set of syscalls that a process is allowed to make. This is a powerful sandboxing technique. For example, a simple network service might be restricted from making any file-related syscalls.
- **Kernel Auditing**: The Linux Audit framework (`auditd`) can log every syscall made by a process, providing a detailed trail for forensic analysis.

## Protection Rings and Privilege Levels

Modern CPUs enforce privilege separation through a mechanism called **protection rings**. These are hierarchical levels of privilege, with Ring 0 being the most privileged and Ring 3 being the least.

- **Ring 0**: Reserved for the OS kernel. Code in Ring 0 can execute any CPU instruction and access any memory address.
- **Rings 1 and 2**: Not typically used by Linux. They were designed for OS components like drivers but are less relevant in modern monolithic kernel architectures.
- **Ring 3**: Used for all user space processes. Code in Ring 3 is restricted and cannot directly access hardware or protected memory.

When a process makes a system call, the CPU transitions from Ring 3 to Ring 0. This hardware-enforced transition is what makes the kernel/userspace boundary secure and efficient.

## Memory and Process Isolation

The kernel leverages the CPU's **Memory Management Unit (MMU)** to enforce memory isolation. Each process gets its own virtual address space, which is a private map of memory addresses. The MMU translates these virtual addresses into physical RAM addresses.

- A process can only access memory within its own virtual address space.
- The kernel ensures that one process cannot read or write to the memory of another process or the kernel itself.
- Any attempt to access a forbidden memory address results in a **segmentation fault**, and the kernel terminates the offending process.

This isolation is a cornerstone of Linux security, preventing a compromised application from stealing data from other running processes.

By strictly enforcing these boundaries, the Linux kernel ensures a stable and secure multi-user environment where applications can run safely without impacting the core system or each other.
