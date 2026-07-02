# Chapter 03 - System Programming Concepts

## Summary
Prerequisites for system programming: what **system calls** are and how they work,
how **library functions** differ from them, the **(GNU) C library**, checking return
statuses and handling errors via **`errno`**, and writing **portable** code (feature
test macros and standard system data types defined by SUSv3).

---

## Standards & terms
- **SUSv3 = Single UNIX Specification, version 3** (published 2001). A big standard that
  defines the API a UNIX system must provide so that programs are **portable** across
  UNIX implementations.
  - Developed by The Open Group / the Austin Group; it was **merged with POSIX**, so
    **SUSv3 is essentially the same document as POSIX.1-2001**.
  - The book uses "SUSv3" as its yardstick: it tells you whether a feature is
    **standardized (portable)** or **Linux-specific**.
  - Later revision: **SUSv4 = POSIX.1-2008**.

---

## 3.1 System Calls
- A **system call** is a **controlled entry point into the kernel**: it lets a process ask the
  kernel to perform an action on its behalf (e.g. create a process, do I/O, create a pipe).
  The full list is in the `syscalls(2)` man page.
- Key general points:
  - A system call **switches the CPU from user mode to kernel mode** so it can access
    protected kernel memory (ties to 2.1).
  - The set of system calls is **fixed**, and each has a **unique number** (programs use
    **names**, not numbers).
  - A system call may take **arguments** that move data between user space and kernel space.
- Calling a syscall *looks* like a normal C function call, but under the hood you actually
  call a **wrapper function in the C library** (glibc), which does the low-level work.
- Simplified steps (x86-32 example):
  1. The program calls the **glibc wrapper** function.
  2. The wrapper copies the syscall **arguments** into the specific **CPU registers** the kernel expects.
  3. The wrapper copies the **system call number** into a register (`%eax`).
  4. The wrapper runs a **trap instruction** (`int 0x80`, or the faster `sysenter`/`syscall`), switching to **kernel mode**.
  5. The kernel's `system_call()` handler runs: saves registers, checks the syscall number, then uses it to index **`sys_call_table`** and call the right **service routine** (`sys_xyz()`), which validates arguments, does the work, and returns a status. Then it restores registers and switches back to **user mode**.
  6. If the result indicates an error, the wrapper sets the global **`errno`** and returns **-1**.
- Linux convention: service routines return a **nonnegative value on success**, and a **negative** (negated `errno`) on error; the wrapper turns that into `errno` + a `-1` return.
- **System calls have small but real overhead** (mode switch + all these steps). Example: `getppid()` is ~20x slower than a plain C function call. Implication: avoid unnecessary syscalls in hot paths (this is *why* stdio buffers I/O - 2.5).
- Terminology: "invoking the system call `xyz()`" means "calling the wrapper that invokes it."
- Tooling: **`strace`** traces the system calls a program makes (great for debugging/learning).

---

## Important system calls / functions
| Call | Purpose | Notes |
| ---- | ------- | ----- |
| `getppid()` | Return the parent process's PID | Used in the book to illustrate syscall overhead |

## Gotchas & things to remember
- You never call the kernel directly - you call a **glibc wrapper** that traps into the kernel.
- A failed syscall wrapper returns **-1** and sets **`errno`**; always check it.

## Exercises
-

## Links
- [TLPI book site](https://man7.org/tlpi/)
