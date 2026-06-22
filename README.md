# The Linux Programming Interface — with Projects

My journey through **_The Linux Programming Interface_ (TLPI)** by Michael Kerrisk —
the definitive guide to Linux and UNIX system programming.

This repo tracks my study of the book chapter by chapter. For every chapter I keep:

- **Theory** → notes, summaries, key syscalls, gotchas, and answers to exercises.
- **Projects** → small C programs that apply what the chapter teaches.

> Chapter 1 (*History and Standards*) is background reading only, so the hands-on
> work starts at **Chapter 2**.

> **Note:** The book itself is copyrighted and is **not** included in this repo
> (it is listed in `.gitignore`). Grab your own copy from
> [man7.org/tlpi](https://man7.org/tlpi/).

---

## Repository structure

```
.
├── theory/                         # Notes & summaries, one folder per chapter
│   ├── chapter-02-fundamental-concepts/
│   │   └── notes.md
│   └── ...
├── projects/                       # Hands-on C programs, one folder per chapter
│   ├── chapter-02-fundamental-concepts/
│   │   └── README.md
│   └── ...
├── .gitignore
├── LICENSE
└── README.md
```

- `theory/chapter-XX-*/` — what I learned, in my own words.
- `projects/chapter-XX-*/` — code that proves I learned it.

---

## How to build & run a project

Each project is plain C and builds with `gcc`:

```bash
cd projects/chapter-04-file-io-universal-io-model
gcc -Wall -Wextra -o demo demo.c
./demo
```

Some chapters link against the TLPI helper library (`libtlpi`). See
[the official source distribution](https://man7.org/tlpi/code/) if a project needs it.

---

## Progress & roadmap

Legend: ⬜ not started · 🟨 in progress · ✅ done

### Part 1 — Background and Concepts
- ⬜ **Ch 2** — Fundamental Concepts
- ⬜ **Ch 3** — System Programming Concepts

### Part 2 — Fundamentals of File I/O
- ⬜ **Ch 4** — File I/O: The Universal I/O Model
- ⬜ **Ch 5** — File I/O: Further Details

### Part 3 — Processes & Memory
- ⬜ **Ch 6** — Processes
- ⬜ **Ch 7** — Memory Allocation
- ⬜ **Ch 8** — Users and Groups
- ⬜ **Ch 9** — Process Credentials
- ⬜ **Ch 10** — Time
- ⬜ **Ch 11** — System Limits and Options
- ⬜ **Ch 12** — System and Process Information

### Part 4 — Files, Filesystems & Attributes
- ⬜ **Ch 13** — File I/O Buffering
- ⬜ **Ch 14** — File Systems
- ⬜ **Ch 15** — File Attributes
- ⬜ **Ch 16** — Extended Attributes
- ⬜ **Ch 17** — Access Control Lists
- ⬜ **Ch 18** — Directories and Links
- ⬜ **Ch 19** — Monitoring File Events

### Part 5 — Signals
- ⬜ **Ch 20** — Signals: Fundamental Concepts
- ⬜ **Ch 21** — Signals: Signal Handlers
- ⬜ **Ch 22** — Signals: Advanced Features
- ⬜ **Ch 23** — Timers and Sleeping

### Part 6 — Process & Program Control
- ⬜ **Ch 24** — Process Creation
- ⬜ **Ch 25** — Process Termination
- ⬜ **Ch 26** — Monitoring Child Processes
- ⬜ **Ch 27** — Program Execution
- ⬜ **Ch 28** — Process Creation and Program Execution in More Detail

### Part 7 — Threads (POSIX Threads)
- ⬜ **Ch 29** — Threads: Introduction
- ⬜ **Ch 30** — Threads: Thread Synchronization
- ⬜ **Ch 31** — Threads: Thread Safety and Per-Thread Storage
- ⬜ **Ch 32** — Threads: Thread Cancellation
- ⬜ **Ch 33** — Threads: Further Details

### Part 8 — Advanced Process Topics
- ⬜ **Ch 34** — Process Groups, Sessions, and Job Control
- ⬜ **Ch 35** — Process Priorities and Scheduling
- ⬜ **Ch 36** — Process Resources
- ⬜ **Ch 37** — Daemons
- ⬜ **Ch 38** — Writing Secure Privileged Programs
- ⬜ **Ch 39** — Capabilities
- ⬜ **Ch 40** — Login Accounting

### Part 9 — Shared Libraries
- ⬜ **Ch 41** — Fundamentals of Shared Libraries
- ⬜ **Ch 42** — Advanced Features of Shared Libraries

### Part 10 — Interprocess Communication (IPC)
- ⬜ **Ch 43** — Interprocess Communication Overview
- ⬜ **Ch 44** — Pipes and FIFOs
- ⬜ **Ch 45** — Introduction to System V IPC
- ⬜ **Ch 46** — System V Message Queues
- ⬜ **Ch 47** — System V Semaphores
- ⬜ **Ch 48** — System V Shared Memory
- ⬜ **Ch 49** — Memory Mappings
- ⬜ **Ch 50** — Virtual Memory Operations
- ⬜ **Ch 51** — Introduction to POSIX IPC
- ⬜ **Ch 52** — POSIX Message Queues
- ⬜ **Ch 53** — POSIX Semaphores
- ⬜ **Ch 54** — POSIX Shared Memory
- ⬜ **Ch 55** — File Locking

### Part 11 — Sockets & Network Programming
- ⬜ **Ch 56** — Sockets: Introduction
- ⬜ **Ch 57** — Sockets: UNIX Domain
- ⬜ **Ch 58** — Sockets: Fundamentals of TCP/IP Networks
- ⬜ **Ch 59** — Sockets: Internet Domain
- ⬜ **Ch 60** — Sockets: Server Design
- ⬜ **Ch 61** — Sockets: Advanced Topics

### Part 12 — Advanced I/O & Terminals
- ⬜ **Ch 62** — Terminals
- ⬜ **Ch 63** — Alternative I/O Models
- ⬜ **Ch 64** — Pseudoterminals

---

## License

Code in this repository is released under the [MIT License](LICENSE).
The book and its example code remain the property of their respective authors.
