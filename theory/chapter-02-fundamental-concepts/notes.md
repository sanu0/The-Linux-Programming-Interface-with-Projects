# Chapter 02 - Fundamental Concepts

## Summary
A fast tour of the core ideas behind Linux system programming: the kernel, the
shell, users/groups, processes, files, I/O, and the system-call interface. Each
idea here is expanded in later chapters.

---

## 2.1 The Core Operating System: The Kernel
- "Operating system" has two meanings:
  - **Broad:** the whole package = kernel + bundled tools (shell, GUI, editors, file utilities).
  - **Narrow:** just the central software that manages CPU, RAM, and devices. This narrow thing is the **kernel**, and it is what this book is about.
- The kernel executable usually lives at `/boot/vmlinuz`. The `z` means it is a compressed executable. (History: early UNIX kernel = `unix`, later `vmunix` once virtual memory existed; Linux = `vmlinuz`.)
- Main jobs of the kernel:
  - **Process scheduling** - decides which process runs on the CPU and for how long. Linux is *preemptive* multitasking: the kernel decides, not the process.
  - **Memory management** - shares limited RAM using *virtual memory*; isolates processes; keeps only the needed parts of a program in RAM.
  - **File systems** - create/read/update/delete files on disk.
  - **Process creation & termination** - loads programs, gives them resources, frees those resources afterwards.
  - **Device access** - one standard interface to devices; arbitrates between processes wanting the same device.
  - **Networking** - sends/receives and routes network packets for processes.
  - **System call API** - official entry points a process uses to ask the kernel to do privileged work. This API is the main subject of the book.
- The kernel gives each user the illusion of a **virtual private computer**.
- **User mode vs kernel mode** (enforced by the CPU hardware):
  - *User mode*: restricted; code can only touch user-space memory.
  - *Kernel mode* (supervisor): full access; can run privileged instructions (halt CPU, configure memory hardware, start device I/O).
  - Memory is split into *user space* and *kernel space*. This hardware fence stops a buggy/malicious program from corrupting the kernel or other processes.
- **Process view vs kernel view:**
  - A process sees the world as asynchronous and hidden: it doesn't know when it will be scheduled, where it sits in RAM (or if it's swapped to disk), or where its files physically live.
  - There is one kernel that knows and controls everything: scheduling, tracking every process, mapping filenames to disk and virtual memory to physical RAM/swap, and doing all device I/O.
  - Key idea: "a process does X" (creates a file, creates a process, etc.) is shorthand for "a process **asks the kernel** to do X." The kernel mediates everything.

---

## 2.2 The Shell
- A **shell** is a program that reads commands you type and runs the matching programs in response. Also called a **command interpreter**.
- A **login shell** is the shell process started for you when you first log in.
- On UNIX/Linux the shell is an ordinary **user process**, not part of the kernel. You can run many shells, switch between them, and run different ones at the same time.
- Major shells through history:
  - **`sh` (Bourne shell)** - oldest widely used shell (Steve Bourne). Introduced I/O redirection, pipelines, filename globbing, variables, environment variables, command substitution, background jobs, and functions.
  - **`csh` (C shell)** - Bill Joy (Berkeley). C-like syntax; added command history, command-line editing, job control, and aliases. Not backward compatible with `sh`.
  - **`ksh` (Korn shell)** - David Korn (AT&T). Successor to `sh`, backward compatible, plus interactive features like `csh`.
  - **`bash` (Bourne Again shell)** - GNU's rewrite of `sh`; the most common shell on Linux. On Linux, `sh` is usually `bash` emulating the Bourne shell.
- The POSIX.2 shell standard was based on the Korn shell; both `bash` and `ksh` conform to POSIX but add their own (differing) extensions.
- Shells are used two ways:
  - **Interactively** - you type commands one at a time.
  - **Scripting** - a *shell script* is a text file of shell commands. Shells are mini programming languages: variables, loops, conditionals, functions, and I/O.

---

## 2.3 Users and Groups
- Every user has a unique **login name (username)** and a numeric **user ID (UID)**.
- User details live in the **password file `/etc/passwd`** (one line per user):
  - login name
  - UID
  - **GID** - the user's primary group
  - **home directory** - where you land right after logging in
  - **login shell** - the program run to interpret your commands
- The encrypted password can be in `/etc/passwd`, but for security it usually lives in the separate **shadow password file (`/etc/shadow`)**, readable only by privileged users.
- **Groups** organize users for administration, especially to control access to files/resources (e.g. a project team sharing files).
  - Early UNIX: a user could belong to only one group. BSD allowed membership in multiple groups at once; adopted by POSIX.1-1990.
  - Group details live in the **group file `/etc/group`** (one line per group): group name, **GID**, and a comma-separated list of member usernames.
- **Superuser (root)**: special account with **UID 0**, login name usually `root`. Bypasses all permission checks - can read/modify any file and send signals to any process. Used by the sysadmin for administrative tasks.

---

## 2.4 Single Directory Hierarchy, Directories, Links, and Files
- Linux puts **all** files in ONE single hierarchical tree (unlike Windows, where each disk has its own tree: `C:`, `D:`). The top is the **root directory `/`**; everything descends from it. Other disks are attached into this one tree (mounting, Ch 14).
- **File types:** every file is tagged with a type. **Regular (plain) files** = ordinary data. Other types: **directories, symbolic links, devices, pipes, sockets**. "File" usually means a file of *any* type.
- **Directories and (hard) links:**
  - A **directory** is a special file holding a table that maps **filenames -> references to the actual file objects**. Each filename+reference pair is a **link**.
  - A single file can have **multiple links** (multiple names) in the same or different directories.
  - Every directory has at least two entries: **`.`** (link to itself) and **`..`** (link to its parent). Root's `..` points to itself (`/..` == `/`).
- **Symbolic (soft) links:**
  - A **symbolic link** is a small special file that just contains the **pathname of another file** (its **target**); it "points" to the target.
  - When a pathname is used in a syscall, the kernel usually **dereferences (follows)** symlinks automatically, recursively if needed (with a limit to stop loops).
  - A symlink whose target doesn't exist = a **dangling link**.
  - Terms: **hard link** = normal link; **soft link** = symbolic link (details in Ch 18).
- **Filenames:**
  - Up to **255 chars** on most Linux filesystems. May contain anything except **`/`** and the **null byte (`\0`)**.
  - Best practice: stick to the **portable filename character set** `[-._a-zA-Z0-9]`. Other characters can have special meaning in the shell/regex and need **escaping** (backslash).
  - Avoid names starting with **`-`** (can be mistaken for command options).
- **Pathnames:**
  - A pathname = optional leading `/` then filenames separated by `/`. Every component except the last must be a directory; the last can be any file type.
  - **directory part** = everything before the final slash; **base part** = the final name.
  - **Absolute** pathname: starts with `/` (resolved from root), e.g. `/home/mtk/.bashrc`.
  - **Relative** pathname: no leading `/`; resolved from the process's current working directory. `..` = parent.
- **Current working directory (CWD):**
  - Each **process** has its own CWD = its "current location" in the tree; relative pathnames resolve from it.
  - A process **inherits** its CWD from its parent. A login shell's initial CWD = the **home directory** from `/etc/passwd`. Change it with `cd`.
- **File ownership and permissions:**
  - Each file has an owner **UID** and a group **GID**.
  - Users split into three categories: **owner** (user), **group** (members of the file's group), **other** (everyone else).
  - Three permission bits per category (**9 total**): **read, write, execute**.
    - On **files**: read = read contents; write = modify contents; execute = run it (program/script).
    - On **directories**: read = list the filenames; write = add/remove/rename entries; execute ("search") = enter/traverse into it to access files inside.

---

## 2.5 The File I/O Model
- **Universality of I/O:** the *same* system calls - `open()`, `read()`, `write()`, `close()` (and `lseek()`) - work on **all** file types, including devices. The kernel translates each request into the right filesystem or device-driver operation, so one program works on any file type.
- The kernel models a file as essentially **one type: a sequential stream of bytes**. For disk files (and disks/tapes) you can also jump to any position with **`lseek()`** (random access).
- **No end-of-file character:** end of file is detected when a `read()` returns **no data (0 bytes)**. The newline character (`\n`, ASCII 10, "linefeed") is treated as the line terminator **by convention** (apps/libraries), not by the kernel.
- **File descriptors (fds):**
  - I/O system calls refer to an open file by a **file descriptor** = a small **non-negative integer**.
  - You normally get one from `open()` (passing a pathname).
  - A process started by the shell inherits **three** open fds:
    - **0 = standard input (stdin)** - where it reads input
    - **1 = standard output (stdout)** - where it writes normal output
    - **2 = standard error (stderr)** - where it writes error/diagnostic messages
  - In an interactive shell these three are normally connected to the **terminal**.
  - In the C **stdio** library they correspond to the streams `stdin`, `stdout`, `stderr`.
- **The stdio library:**
  - C programs usually do I/O via standard C library **stdio** functions: `fopen()`, `fclose()`, `scanf()`, `printf()`, `fgets()`, `fputs()`, etc.
  - These are **layered on top of** the lower-level I/O system calls (`open`, `close`, `read`, `write`). They add convenience and buffering.

---

## 2.6 Programs
- A program normally exists in **two forms**:
  - **Source code** - human-readable text written in a language like C.
  - **Binary machine code** - the instructions the CPU actually executes.
  - Source must be **compiled + linked** into a binary to run; the two senses of "program" are treated as the same because compiling/linking produces semantically equivalent machine code.
  - Contrast: a **script** is a text file of commands run directly by an interpreter (e.g. a shell), not compiled to machine code.
- **Filters:** a program that reads from **stdin**, transforms the data, and writes to **stdout**. Examples: `cat`, `grep`, `tr`, `sort`, `wc`, `sed`, `awk`.
- **Command-line arguments:** a C program reads them via `int main(int argc, char *argv[])`:
  - `argc` = count of arguments.
  - `argv` = array of argument strings; **`argv[0]` is the program's own name**.

---

## 2.7 Processes (Part 1: lifecycle)
**What a process is**
- A **process is an instance of an executing program.** To run a program, the kernel loads its code into virtual memory, allocates space for its variables, and sets up bookkeeping structures (PID, termination status, UIDs/GIDs, etc.).
- Kernel's view: processes are the entities it **shares the computer's resources among.** Limited resources (e.g. memory) are allocated/adjusted over a process's life and released at termination; renewable ones (CPU, network bandwidth) are shared fairly.

**Process memory layout (segments)**
- **Text** - the program's machine instructions (read-only, shareable between processes running the same program).
- **Data** - static/global variables.
- **Heap** - memory allocated dynamically at runtime (grows "up").
- **Stack** - grows/shrinks with function calls; holds local variables + call linkage (return address, saved registers).

**Process creation & program execution**
- **`fork()`** - a process creates a new process. Caller = **parent**, new one = **child**. The kernel makes the child a **duplicate** of the parent: the child gets its **own copies** of the data, heap, and stack (modifiable independently); the **text is shared read-only**.
- After `fork()`, the child either runs different code paths in the same program, or commonly calls **`execve()`** to load and run a **brand-new program** - which discards the current text/data/heap/stack and replaces them with the new program's.
- **`exec()`** = shorthand for the family of C library functions layered on `execve()` (there is no real function literally named `exec()`). "to exec" = perform this replacement.
- Classic pattern: **`fork()` then `exec()`** is exactly how a shell runs a command (fork a child, child execs the program, parent waits).

**Process ID (PID) and parent PID (PPID)**
- Every process has a unique integer **PID**.
- Every process has a **PPID** = the PID of the process that created it.

**Process termination & termination status**
- A process ends either by requesting it itself via **`_exit()`** (or the **`exit()`** library wrapper), or by being **killed by a signal**.
- It yields a **termination status**: a small non-negative integer the parent reads with **`wait()`**.
  - With `_exit()`, the process chooses its own status.
  - If killed by a signal, the status reflects the signal.
- Convention: **0 = success, nonzero = error.** Shells expose the last program's status in the **`$?`** variable.

---

## 2.7 Processes (Part 2: attributes & special processes)
**Process credentials (user/group IDs)**
- **Real UID/GID** - who the process actually belongs to. Inherited from the parent; a login shell gets them from `/etc/passwd`.
- **Effective UID/GID** - used (with supplementary groups) to decide **permissions** when accessing protected resources (files, IPC). Usually equal to the real IDs. Changing the effective IDs lets a process temporarily take on another user/group's privileges.
- **Supplementary group IDs** - extra groups the process belongs to. Inherited from the parent; a login shell gets them from `/etc/group`.

**Privileged processes**
- Traditionally, a process with **effective UID 0** (superuser) is **privileged**: it bypasses the kernel's normal permission checks. Processes with a nonzero effective UID are **unprivileged** and must obey the rules.
- A process can be privileged because its parent was (e.g. a login shell started by root), or via the **set-user-ID** mechanism: running a set-UID program makes the process's effective UID equal to the **owner of the program file**.

**Capabilities**
- Since Linux 2.2, the superuser's all-or-nothing power is split into distinct units called **capabilities** (names start with `CAP_`, e.g. `CAP_KILL`).
- Each privileged operation needs a specific capability; a process can do it only if it holds that capability. An effective-UID-0 process has **all** capabilities. Granting a subset enables some privileged actions but not others. (Details: Ch 39.)

**The `init` process**
- At boot the kernel creates **`init`** (from `/sbin/init`), the **"parent of all processes."** Every process is created (via `fork()`) by `init` or one of its descendants.
- `init` always has **PID 1**, runs with superuser privileges, **can't be killed** (even by root), and exits only at shutdown. Its job: create and monitor the processes a running system needs. (Modern systems often use `systemd` as init.)

**Daemon processes**
- A **daemon** is a normal process that is (a) **long-lived** (often started at boot, runs until shutdown) and (b) runs in the **background with no controlling terminal**. Examples: `syslogd` (system logging), `httpd` (web server). Names often end in `d`.

**Environment list (environment variables)**
- Each process has an **environment list**: `name=value` pairs kept in its user-space memory.
- A child from `fork()` **inherits a copy** of the parent's environment -> a way for a parent to pass info to a child. `exec()` either keeps the existing environment or installs a new one.
- Shell: create/export with `export NAME=value` (`setenv` in csh). In C: access via `extern char **environ` and library functions (`getenv()`, `setenv()`).
- Examples: `HOME` (login directory), `PATH` (directories the shell searches for commands).

**Resource limits**
- Each process consumes resources (open files, memory, CPU time). With **`setrlimit()`** a process caps its own consumption.
- Each limit has a **soft limit** (the actually enforced cap) and a **hard limit** (ceiling the soft limit may be raised to). An unprivileged process can set its soft limit anywhere from 0 up to the hard limit, but can only **lower** its hard limit.
- Limits are **inherited** by children via `fork()`. The shell adjusts them with the **`ulimit`** command (`limit` in csh).

---

## 2.8 Memory Mappings
- **`mmap()`** creates a new **memory mapping** in the calling process's virtual address space - a region of memory "backed by" either a file or zero-filled pages.
- Two categories:
  - **File mapping** - maps a region of a file into memory. You then access the file's contents directly as bytes in that memory region; the kernel loads pages from the file **on demand**. (a.k.a. **memory-mapped I/O**)
  - **Anonymous mapping** - no backing file; pages are initialized to **0**. (Used e.g. for fresh memory allocation.)
- Mappings can be **shared** between processes - either two processes map the same file region, or a child inherits the parent's mapping via `fork()`.
- **Private vs shared:**
  - **Private** mapping: your modifications are **not** visible to other processes and are **not** written back to the file (copy-on-write).
  - **Shared** mapping: your modifications **are** visible to other processes sharing it and **are** carried through to the underlying file.
- Common uses: initializing a process's **text segment** from the executable file, allocating new **zero-filled memory**, fast **file I/O** (memory-mapped I/O), and **interprocess communication** (via a shared mapping).

---

## 2.9 Static and Shared Libraries
- An **object library** is a file holding the compiled object code of a set of (usually related) functions that programs can call. Bundling them eases program creation and maintenance. Two types: **static** and **shared**.
- **Static libraries** (archives, `.a`):
  - A structured bundle of compiled object modules; you name the library in the link command.
  - The linker resolves the needed functions and **copies those object modules into the executable** -> the program is **statically linked**.
  - Downsides: every program carries its **own copy** -> wastes **disk** space; wastes **memory** when several such programs run at once (each keeps its own copy of the function in RAM); updating a function means recompiling it into the library **and relinking** every program that uses it.
- **Shared libraries** (`.so`; `.dll` on Windows, `.dylib` on macOS):
  - When linked, the linker does **not** copy modules in; it just records that the executable **needs** that shared library at run time.
  - At run time, the **dynamic linker** finds and loads the required shared libraries and resolves the executable's function calls to their definitions in those libraries.
  - Only **one copy** of the shared library code needs to be in memory; **all** running programs use that single copy.
  - Benefits: saves **disk** (one compiled copy), saves **memory** (shared in RAM), and makes **updates easy** - rebuild the shared library once and existing programs pick up the new version next time they run (e.g. a security fix reaches all programs at once).
- Ties to earlier: the dynamic linker uses memory mappings (2.8) to map the library file into each process; the single physical copy is shared (like the text segment in 2.7). The C library (`libc`, where `printf` lives) is the classic shared library.

---

## 2.10 Interprocess Communication (IPC) and Synchronization
- A running system has many processes; some are independent, but some must **cooperate**, which requires **communicating** (sharing data) and **synchronizing** (coordinating their actions/timing).
- Processes *could* communicate by reading/writing disk files, but that's often **too slow and inflexible**.
- Linux provides a rich set of IPC mechanisms:
  - **Signals** - notify that an event/condition has occurred.
  - **Pipes** (the shell `|` operator) and **FIFOs** - transfer data between processes.
  - **Sockets** - transfer data between processes on the **same host or different hosts over a network**.
  - **File locking** - lock regions of a file so other processes can't read/update those regions.
  - **Message queues** - exchange discrete messages (packets of data).
  - **Semaphores** - **synchronize** the actions of processes.
  - **Shared memory** - two+ processes share a piece of memory; one process's change is **immediately** visible to the others (the fastest method).
- Why so many overlapping mechanisms? Historical evolution under different UNIX variants + standards. E.g. **FIFOs (from System V)** and **UNIX domain sockets (from BSD)** do essentially the same job; both survive for compatibility.

---

## 2.11 Signals
- A **signal** is a "**software interrupt**": its arrival tells a process that some **event or exceptional condition** has occurred. Each signal type is a different integer with a symbolic name of the form **`SIGxxxx`**.
- Signals can be sent by: the **kernel**, **another process** (with suitable permissions), or the **process itself**. The kernel sends one when, e.g.:
  - the user typed the interrupt character (usually **Ctrl-C**);
  - one of the process's **children terminated**;
  - a **timer** (alarm) set by the process expired;
  - the process accessed an **invalid memory address**.
- Sending: the **`kill`** command (shell) or the **`kill()`** system call (in programs).
- On receipt, depending on the signal, a process either:
  - **ignores** it, or
  - is **killed** by it, or
  - is **suspended** (then later resumed by a special signal).
- For most signals, instead of the default action a program can choose to **ignore** it or install a **signal handler** - a programmer-defined function automatically called when the signal is delivered.
- Lifecycle: between being **generated** and being **delivered**, a signal is **pending**. It's normally delivered as soon as the process next runs. A process can **block** a signal by adding it to its **signal mask**; a signal generated while blocked stays pending until unblocked.

---

## 2.14 Sessions, Controlling Terminals, and Controlling Processes
- Hierarchy: a **session** is a collection of **process groups (jobs)**, and each process group is a set of processes. (**session > process groups > processes**.)
- All processes in a session share a **session ID**. The **session leader** is the process that created the session; its PID becomes the session ID. Usually the **login shell** is the session leader.
- A session usually has a **controlling terminal** - established when the session leader first opens a terminal device (for an interactive shell, the terminal you logged in at). A terminal is the controlling terminal of **at most one** session.
- The session leader that opens the terminal becomes the **controlling process**. It receives **`SIGHUP`** if the terminal disconnects (e.g. the terminal window is closed) - this is why terminal-bound programs die when you close the window.
- **Foreground vs background:**
  - At any time, exactly **one** process group is the **foreground job** - it may read input from and write output to the terminal.
  - Typing **Ctrl-C** (interrupt) or **Ctrl-Z** (suspend) makes the terminal driver send a signal that **kills or suspends the whole foreground process group**.
  - A session can have any number of **background jobs**, started by ending a command with **`&`**.
- Job-control shells provide commands to **list jobs, signal jobs, and move jobs between foreground and background** (`jobs`, `fg`, `bg`).

---

## 2.15 Pseudoterminals
- A **pseudoterminal (PTY)** is a pair of connected virtual devices: a **master** and a **slave**. The pair is an **IPC channel** carrying data **both ways** between them.
- Key point: the **slave behaves exactly like a real terminal.** So a terminal-oriented program can be connected to the slave, while another program (the **driver**) connected to the master **drives** it - playing the role normally played by the **human user at a real terminal**.
- Data flow: what the driver writes to the master goes through the **usual terminal input processing** (e.g. carriage-return -> newline) and arrives as input to the program on the slave; what the slave program writes goes through terminal **output processing** and arrives as input to the driver.
- Uses: **terminal emulator windows** (xterm, GUI/IDE terminals) and **network login services** like **`telnet`** and **`ssh`**.
- Tie-in: the `/dev/pts/N` device behind a controlling terminal (2.14) is literally a **p**seudo**t**erminal **s**lave; real consoles are `/dev/ttyN`.

---

## Important system calls / functions
| Call | Purpose | Notes |
| ---- | ------- | ----- |
| `open()` | Open (or create) a file; returns a file descriptor | Takes a pathname; fd is a small non-negative int |
| `read()` | Read bytes from an fd | Returns **0** at end of file |
| `write()` | Write bytes to an fd | |
| `close()` | Close an fd, freeing it for reuse | |
| `lseek()` | Move the file offset (random access) | For files/devices that support seeking |
| `printf()`, `fopen()`, `scanf()`, `fgets()`, `fputs()`, `fclose()` (stdio) | Higher-level, **buffered** I/O | Layered on top of the syscalls above |
| `fork()` | Create a new process (a child duplicate of the caller) | Child copies data/heap/stack, shares text read-only |
| `execve()` | Replace the current process's program with a new one | `exec()` = the C-library family layered on it |
| `_exit()` / `exit()` | Terminate the process (choosing its status) | `exit()` is the library wrapper |
| `wait()` | Parent waits for a child and reads its termination status | 0 = success; shell sees it as `$?` |
| `getpid()` / `getppid()` | Get the process's own PID / its parent's PID | |
| `setrlimit()` / `getrlimit()` | Set / get resource limits (soft & hard) | Inherited via `fork()`; shell uses `ulimit` |
| `getenv()` / `setenv()` | Read / set environment variables | C access via `extern char **environ` |
| `mmap()` | Create a memory mapping (file-backed or anonymous) | Private (copy-on-write) vs shared; used for I/O, allocation, IPC |
| `kill()` | Send a signal to a process | Shell equivalent: the `kill` command |
| `signal()` / `sigaction()` | Install a signal handler (or set ignore/default) | `sigaction()` is the modern, preferred one |

## Gotchas & things to remember
- "A process does X" really means "a process asks the kernel to do X."
- The user/kernel mode boundary is enforced by hardware, not by trust.
- **`SIGKILL` (9) and `SIGSTOP` cannot be caught, blocked, or ignored** - they always take effect (the two "unstoppable" signals).

## Exercises
### 2.2 Self-check
1. **Why does it matter that the shell is a user process, not part of the kernel?** Because it can be freely replaced/swapped, run in many instances at once, has no special privileges (it uses the same syscalls as any program), and if it crashes the kernel keeps running.
2. **Interactive vs script?** Interactive = type commands one at a time with immediate feedback and comforts like history (`Ctrl+R`), tab-completion, and line editing. Script = a file of commands run as a repeatable batch using programming features (variables, loops, conditionals, functions).
3. **How does the shell find `git`?** It searches the directories in `$PATH` in order (first match wins) for an executable named `git`, then `fork()`s a child and `execve()`s `git` while the shell waits. (`which`/`type` show which one wins.)
4. **Why is `cd` a built-in?** Each process has its own current working directory; a child changing it wouldn't affect the shell and would vanish on exit, so `cd` must run inside the shell process itself.

## Links
- [TLPI book site](https://man7.org/tlpi/)
