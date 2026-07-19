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

## 3.2 Library Functions
- A **library function** is one of the many functions in the **standard C library**. (The book often just says "function".) They do diverse jobs: opening a file, formatting a time, comparing strings, allocating memory, etc.
- Two kinds:
  - **Pure user-space** functions that make **no system calls** at all - e.g. string functions (`strlen`, `strcmp`), math (`sqrt`). They just compute in user mode.
  - Functions **layered on top of system calls** - e.g. `fopen()` uses the `open()` syscall to actually open the file.
- Library functions often give a **friendlier interface** than the raw syscall beneath:
  - **`printf()`** adds **formatting** (`%d`, `%s`) and **buffering**, whereas the underlying **`write()`** syscall just outputs a block of bytes.
  - **`malloc()` / `free()`** do the bookkeeping that makes allocating/freeing memory far easier than the underlying **`brk()`** syscall (which just moves the heap boundary).
- Key distinction: a **system call** enters the **kernel** (a mode switch); a **library function** is ordinary **user-space** code that *may or may not* call a syscall underneath. From the programmer's view, both look like normal C function calls.
- Practical: **man-page sections** distinguish them - `man 2 xxx` = system calls, `man 3 xxx` = library functions (e.g. `man 2 write`, `man 3 printf`).

---

## 3.3 The Standard C Library; The GNU C Library (glibc)
- The "**standard C library**" is a **specification**; each UNIX system ships an **implementation** of it. On Linux the most common implementation is **glibc** (the GNU C library) - the `libc.so.6` shared library that nearly every program links against. It provides `printf`, `malloc`, the **syscall wrappers**, etc.
- Historically maintained by Roland McGrath, then Ulrich Drepper (the book is ~2010; today it's community/steering-committee maintained).
- **Other C libraries** exist, especially smaller ones for **embedded**/constrained use: **uClibc**, **diet libc**. (Modern extra beyond the book: **musl** - small, clean; popular in **Alpine Linux / Docker** containers.) The book sticks to glibc.
- **Finding the glibc version:**
  - From the shell, **run the library file itself** like a program: `/lib/libc.so.6` prints a version banner. (Modern shortcut: `ldd --version`.)
  - Find *where* glibc lives: run **`ldd`** (list dynamic dependencies) on a dynamically-linked program and look for `libc`: `ldd myprog | grep libc`.
  - **From a program**, two ways:
    - **Compile time:** test the constants **`__GLIBC__`** and **`__GLIBC_MINOR__`** in `#ifdef` (e.g. 2 and 12 for glibc 2.12). Limitation: reflects only the build system.
    - **Run time:** call **`gnu_get_libc_version()`** (returns e.g. `"2.12"`), or `confstr(_CS_GNU_LIBC_VERSION, ...)` (returns e.g. `"glibc 2.12"`). Run-time matters because, with **dynamic linking** (2.9), the glibc used at run time can differ from the one at compile time.

---

## 3.4 Handling Errors from System Calls and Library Functions
- **Golden rule:** almost every system call / library function returns a status showing success or failure - **always check it**. Skipping checks is a "false economy" that wastes debugging hours when something that "couldn't fail" does.
- A **few** calls never fail and need no check: **`getpid()`** (always returns the PID), **`_exit()`** (always terminates).
- **System call error convention:**
  - The man page documents return values; usually **failure = a return of `-1`**.
  - On failure the call sets the global **`errno`** to a **positive** value identifying the error. `#include <errno.h>` declares `errno` and the **`E...` constants** (e.g. `EINTR`, `ENOENT`, `EACCES`). The **ERRORS** section of each man page lists the possible values.
  - **Check order matters:** a successful call does **not** reset `errno` to 0 (and SUSv3 even permits a successful call to set it). So **first check the return value for an error, THEN read `errno`** for the cause.
  - Special case: a few calls (e.g. `getpriority()`) can legitimately return `-1` on success. For these, set `errno = 0` **before** the call, then treat "returned -1 **and** `errno != 0`" as the error.
- **Reporting errors** from `errno`:
  - **`perror(msg)`** - prints your `msg`, then `": "` and the text for the current `errno`, to **stderr**.
  - **`strerror(errnum)`** - **returns** the error string for a given error number (use it with `fprintf`, etc.). The returned string may be **statically allocated** (can be overwritten by later calls) and is **locale-sensitive**.
- **Library-function errors** fall into three groups:
  1. Report **exactly like syscalls**: return `-1` and set `errno` (e.g. `remove()`). Diagnose the same way.
  2. Return a **different failure value** but still set `errno` (e.g. `fopen()` returns `NULL`). Still usable with `perror()`/`strerror()`.
  3. **Don't use `errno` at all** - check per the function's man page. For these, using `errno`/`perror()`/`strerror()` is a **mistake**.
- (Modern note beyond the book: `errno` is actually **per-thread** (thread-safe), and **`strerror_r()`** is the thread-safe version of `strerror()`.)

---

## 3.5 Notes on the Example Programs
> The helper code shown below comes from TLPI's **freely-licensed (LGPL/AGPL)** source
> distribution; the explanations/comments are my own. These helpers appear in nearly
> every example program, so it pays to understand them deeply.

### 3.5.1 Command-Line Options and Arguments
- Two option styles:
  - **Traditional UNIX**: a hyphen + a single letter, optionally with an argument - `-l`, `-n 5`, `-o out.txt`.
  - **GNU long options**: two hyphens + a word - `--help`, `--verbose`.
- Options are parsed with the standard **`getopt()`** library function (Appendix B).
- Programs with nontrivial command lines also support **`--help`** (print a usage message).

**`getopt()` usage, explained line by line:**
```c
#include "tlpi_hdr.h"

int main(int argc, char *argv[]) {
    int opt, count = 1, verbose = 0;

    /* optstring "n:v": valid options are -n and -v.
       A ':' AFTER a letter means "this option requires an argument". */
    while ((opt = getopt(argc, argv, "n:v")) != -1) {
        switch (opt) {
        case 'n':                     /* -n seen; its argument is in the global optarg */
            count = getInt(optarg, GN_GT_0, "-n");
            break;
        case 'v':                     /* -v is a boolean flag (takes no argument) */
            verbose = 1;
            break;
        case '?':                     /* getopt returns '?' for an unknown option / missing arg */
            usageErr("%s [-n count] [-v] file\n", argv[0]);
        }
    }
    /* After the loop, the global `optind` = index of the first NON-option (positional) arg. */
    if (optind >= argc)
        usageErr("%s [-n count] [-v] file\n", argv[0]);
    char *file = argv[optind];
    /* ... */
}
```
- `getopt(argc, argv, optstring)` - returns the next option letter each call, or **`-1`** when options run out.
- **`optstring`** - the valid option letters; a trailing **`:`** on a letter means that option **takes an argument**.
- **`optarg`** (global) - points to the current option's argument (for options that take one).
- **`optind`** (global) - index of the next `argv` element; after the loop it marks where the **positional arguments** begin.
- **`'?'`** - `getopt` returns this for an unrecognized option or a missing required argument.

### 3.5.2 Common Functions and Header Files

**Common header `tlpi_hdr.h`** - included by almost every example:
```c
#ifndef TLPI_HDR_H              /* include guard: avoid processing this file twice */
#define TLPI_HDR_H

#include <sys/types.h>          /* common system type definitions (pid_t, etc.) */
#include <stdio.h>              /* printf(), fprintf(), ... */
#include <stdlib.h>            /* exit(), malloc(), EXIT_SUCCESS/EXIT_FAILURE */
#include <unistd.h>            /* prototypes for many system calls (read, write, fork...) */
#include <errno.h>             /* errno + the E... constants */
#include <string.h>           /* strerror(), strlen(), ... */

#include "get_num.h"           /* the book's getInt()/getLong() */
#include "error_functions.h"   /* the book's errExit(), errMsg(), ... */

typedef enum { FALSE, TRUE } Boolean;          /* C89 had no bool type */
#define min(m,n) ((m) < (n) ? (m) : (n))
#define max(m,n) ((m) > (n) ? (m) : (n))
#endif
```
- **Include guard** (`#ifndef/#define/#endif`) prevents double inclusion.
- Bundles the ~6 headers nearly every program needs, so examples don't repeat them.
- Pulls in the book's own helpers and defines a `Boolean` type + `min`/`max` macros.
- Net effect: one `#include "tlpi_hdr.h"` replaces a dozen boilerplate lines.

**Error-diagnostic functions** (declared in `error_functions.h`) - what each does:

| Function | Prints | Exits? | Exit method |
| --- | --- | --- | --- |
| `errMsg(fmt, ...)` | your msg + `errno` name & text -> stderr | No | - |
| `errExit(fmt, ...)` | same as `errMsg` | Yes | `exit()` (or `abort()`->core if `EF_DUMPCORE` set) |
| `err_exit(fmt, ...)` | same | Yes | `_exit()`, and does **not** flush stdout |
| `errExitEN(en, fmt, ...)` | uses the **given** error number `en` | Yes | `exit()` |
| `fatal(fmt, ...)` | just your msg (no `errno`) | Yes | `exit()` |
| `usageErr(fmt, ...)` | `Usage: ` + your msg | Yes | `exit()` |
| `cmdLineErr(fmt, ...)` | `Command-line usage error: ` + your msg | Yes | `exit()` |

- All take **`printf`-style** args and auto-append a newline.
- A `NORETURN` macro marks the exiting ones as non-returning so `gcc -Wall` won't warn.
- **When to use which:** `errExit` = everyday (a call failed and set `errno`); `errMsg` = same info but keep running; `fatal` = error not tied to `errno`; `usageErr`/`cmdLineErr` = user invoked the program wrong; `errExitEN` = POSIX threads (they return the error number instead of setting `errno`); `err_exit` = a child after `fork()` that must die without flushing inherited stdio buffers.

Typical use (replaces the verbose `perror`+`exit` pattern from 3.4):
```c
fd = open(pathname, O_RDONLY);
if (fd == -1)
    errExit("open %s", pathname);   /* -> ERROR [ENOENT No such file...] open data.txt ; then exits */
```

**Implementation - `terminate()` (decides HOW to exit):**
```c
static void
terminate(Boolean useExit3)               /* useExit3: TRUE -> exit(3); FALSE -> _exit(2) */
{
    char *s = getenv("EF_DUMPCORE");       /* read an env var (see 2.7) */
    if (s != NULL && *s != '\0')           /* if it exists AND is non-empty... */
        abort();                           /* ...raise SIGABRT -> core dump for the debugger */
    else if (useExit3)
        exit(EXIT_FAILURE);                /* tidy exit: flush stdio + run atexit handlers */
    else
        _exit(EXIT_FAILURE);               /* raw kernel exit: no flush, no handlers */
}
```
- Clever: exporting `EF_DUMPCORE=1` makes any `errExit` dump core - debug without recompiling.
- `exit()` vs `_exit()`: tidy vs raw (detail in Ch 25).

**Implementation - `outputError()` (builds & prints the message):**
```c
static void
outputError(Boolean useErr, int err, Boolean flushStdout,
            const char *format, va_list ap)
{
#define BUF_SIZE 500
    char buf[BUF_SIZE], userMsg[BUF_SIZE], errText[BUF_SIZE];

    vsnprintf(userMsg, BUF_SIZE, format, ap);      /* format caller's msg safely (bounded) */

    if (useErr)                                    /* include errno info? */
        snprintf(errText, BUF_SIZE, " [%s %s]",
            (err > 0 && err <= MAX_ENAME) ?
                ename[err] : "?UNKNOWN?",          /* symbolic name, e.g. "ENOENT" */
            strerror(err));                        /* human description */
    else
        snprintf(errText, BUF_SIZE, ":");

    snprintf(buf, BUF_SIZE, "ERROR%s %s\n", errText, userMsg);  /* assemble final line */

    if (flushStdout)
        fflush(stdout);        /* push pending normal output FIRST, so ordering is right */
    fputs(buf, stderr);        /* write error to stderr */
    fflush(stderr);            /* ensure it shows even if stderr is buffered */
}
```
- `vsnprintf` = bounded `printf` into a buffer using a `va_list` (won't overflow).
- Prints **both** the symbolic name (`ename[err]`) and the description (`strerror`) - see `ename` below.
- `fflush(stdout)` before writing stderr keeps normal output and errors in the correct order (2.5 buffering).

**Implementation - `errMsg()` (note the errno save/restore):**
```c
void
errMsg(const char *format, ...)
{
    va_list argList;
    int savedErrno = errno;                        /* save: our printing might change errno */
    va_start(argList, format);                     /* start varargs after 'format' */
    outputError(TRUE, errno, TRUE, format, argList);  /* TRUE -> include errno text */
    va_end(argList);
    errno = savedErrno;                            /* restore so caller still sees it */
}
```
- `va_list`/`va_start`/`va_end` = C's variadic-argument mechanism (same as `printf` uses).
- **Saves and restores `errno`** - a courtesy so `errMsg()` doesn't clobber the caller's error code.

**`errExit()` vs `err_exit()`:**
```c
void errExit(const char *format, ...) {
    va_list a; va_start(a, format);
    outputError(TRUE, errno, TRUE, format, a);     /* include errno, flush stdout */
    va_end(a);
    terminate(TRUE);                               /* -> exit() */
}
void err_exit(const char *format, ...) {
    va_list a; va_start(a, format);
    outputError(TRUE, errno, FALSE, format, a);    /* do NOT flush stdout */
    va_end(a);
    terminate(FALSE);                              /* -> _exit() */
}
```

**`ename.c.inc` - error number -> symbolic name:**
```c
static char *ename[] = {
    /*  0 */ "",
    /*  1 */ "EPERM", "ENOENT", "ESRCH", "EINTR", "EIO", /* ... */
    /* 11 */ "EAGAIN/EWOULDBLOCK", "ENOMEM",
    /* 13 */ "EACCES", /* ... */
};
#define MAX_ENAME 132
```
- Why it exists: `strerror()` gives the *description* but **not** the *symbolic name*, and man pages refer to errors by symbolic name. Printing `ename[errno]` lets you look up the error easily.
- **Architecture-specific** (errno numbers vary by platform; generated by `Build_ename.sh`).
- Empty strings = unused numbers; two-name entries = constants sharing a value, e.g. **`EAGAIN`/`EWOULDBLOCK`** (returned when a call would block but the caller asked for non-blocking; `EAGAIN` from System V, `EWOULDBLOCK` from BSD).

**Numeric argument parsing - `getInt()` / `getLong()`** (safer than `atoi()`, which silently returns 0 on bad input):
```c
/* flags (from get_num.h): */
#define GN_NONNEG   01     /* value must be >= 0 */
#define GN_GT_0     02     /* value must be > 0 */
#define GN_ANY_BASE 0100   /* accept any base, like strtol()'s base 0 */
#define GN_BASE_8   0200   /* octal */
#define GN_BASE_16  0400   /* hex */

static long
getNum(const char *fname, const char *arg, int flags, const char *name)
{
    long res; char *endptr; int base;

    if (arg == NULL || *arg == '\0')          /* reject NULL / empty string */
        gnFail(fname, "null or empty string", arg, name);

    base = (flags & GN_ANY_BASE) ? 0 :        /* choose the numeric base from flags */
           (flags & GN_BASE_8)  ? 8 :
           (flags & GN_BASE_16) ? 16 : 10;    /* default: decimal */

    errno = 0;                                /* 3.4 rule: clear errno before strtol */
    res = strtol(arg, &endptr, base);
    if (errno != 0)                           /* overflow/underflow: strtol set errno */
        gnFail(fname, "strtol() failed", arg, name);
    if (*endptr != '\0')                      /* leftover chars => not a clean number ("12ab") */
        gnFail(fname, "nonnumeric characters", arg, name);
    if ((flags & GN_NONNEG) && res < 0)       /* enforce >= 0 if requested */
        gnFail(fname, "negative value not allowed", arg, name);
    if ((flags & GN_GT_0) && res <= 0)        /* enforce > 0 if requested */
        gnFail(fname, "value must be > 0", arg, name);
    return res;
}
/* getInt() calls getNum() then also range-checks against INT_MIN..INT_MAX. */
```
- **`endptr`**: `strtol` points it at the first character it *couldn't* convert; if that isn't `'\0'`, the string had trailing junk.
- The `errno = 0` before / check after is exactly the 3.4 pattern for a call (`strtol`) that can legitimately return values that look like errors.
- `name` labels the argument in any error message (e.g. `"-n"`), so the user knows which input was bad.
- Usage: `int n = getInt(argv[1], GN_GT_0, "num-threads");  /* must be > 0, else error+exit */`

---

## 3.6 Portability Issues
Techniques for writing system programs that run on any standards-conformant system:
**feature test macros**, **standard system data types**, and a few miscellaneous issues.

### 3.6.1 Feature Test Macros
- Different standards govern the API (POSIX/SUS from The Open Group; BSD and System V/SVID from the historic implementations). **Feature test macros** let us tell the headers **which standard's definitions to expose**.
- Two ways to define one (must be **before** including headers):
  ```c
  #define _BSD_SOURCE 1        /* in the source, before any #include */
  ```
  ```bash
  cc -D_BSD_SOURCE prog.c      # or via the compiler -D option
  ```
- Name logic: the *implementation* decides which features to reveal by **testing** (`#if`) which macros the app defined.
- **Portable (standard) macros:**
  - **`_POSIX_SOURCE`** - expose POSIX.1-1990 + ISO C (1990). Superseded by the next.
  - **`_POSIX_C_SOURCE`** - `1` = same as above; `>=199309` adds POSIX.1b (realtime); `>=199506` adds POSIX.1c (threads); `200112` = POSIX.1-2001 base; `200809` = POSIX.1-2008 base.
  - **`_XOPEN_SOURCE`** - any value = POSIX.1/POSIX.2/X-Open; `>=500` = SUSv2; `>=600` = **SUSv3** + C99; `>=700` = SUSv4. (500/600/700 = SUS v2/v3/v4.)
- **glibc-specific macros:** `_BSD_SOURCE` (BSD defs; favors BSD on conflicts), `_SVID_SOURCE` (System V defs), **`_GNU_SOURCE`** (everything above + GNU extensions).
- **Defaults** (plain `gcc`, no special options): `_POSIX_SOURCE`, `_POSIX_C_SOURCE=200809` (value varies by glibc), `_BSD_SOURCE`, `_SVID_SOURCE`. Using `-ansi`/`-std=c99` or defining a macro yourself restricts to just those definitions. Defining macros is **additive**.
- **SUSv3 conformance:** SUSv3 specifies only `_POSIX_C_SOURCE` (=200112) and `_XOPEN_SOURCE` (=600); defining just **`_XOPEN_SOURCE=600`** gives full SUSv3. (SUSv4: 200809 / 700.)
- The book's examples compile under default `gcc` options **or**: `cc -std=c99 -D_XOPEN_SOURCE=600`. Each man page lists the macro(s) needed to expose a given function.

### 3.6.2 System Data Types
- Don't use native C types (`int`, `long`) directly for things like PIDs/UIDs/offsets, because:
  - Native type **sizes vary** across implementations/compilation environments (a `long` may be 4 or 8 bytes), and different systems may use different types for the same info.
  - Types can even **change between releases** (Linux user/group IDs were 16-bit on <=2.2, 32-bit on >=2.4).
- So SUSv3 defines **standard system data types** via `typedef`, e.g. `typedef int pid_t;`. Most end in **`_t`**; many live in **`<sys/types.h>`**. Use them for portable declarations:
  ```c
  pid_t mypid;   /* correct on any SUSv3-conformant system */
  ```
- Common ones (Table 3-1): `pid_t` (PID, signed int), `uid_t`/`gid_t` (user/group ID, int), `off_t` (file offset/size, signed int), `size_t` (object size, unsigned), `ssize_t` (byte count or -1), `mode_t` (perms+type), `time_t` (seconds since Epoch), `ino_t` (i-node), `dev_t` (device), `sig_atomic_t`, etc. "is an integer type" means SUSv3 requires an integer but **not which** native one - portable code shouldn't care.
- **Printing them safely:** `printf()` can't know the underlying type, and C's promotion rules make it an `int` or `long` depending on the typedef. Hardcoding `%d` vs `%ld` is an implementation dependency. **Solution: cast to `long` and use `%ld`:**
  ```c
  pid_t mypid = getpid();
  printf("My PID is %ld\n", (long) mypid);   /* portable */
  ```
  - Exception: **`off_t`** is `long long` in some environments -> cast to `long long`, use `%lld` (Section 5.10).
  - C99's `%zd` (for `size_t`/`ssize_t`) and `%jd` (for `intmax_t`) exist but the book avoids them (not on all UNIX).

### 3.6.3 Miscellaneous Portability Issues
- **Initializing structures:** for standard structs (e.g. `struct sembuf`), the **field order is generally unspecified** and there may be **extra implementation-specific fields**. So **positional initializers aren't portable**:
  ```c
  struct sembuf s = { 3, -1, SEM_UNDO };          /* NOT portable (assumes field order) */
  ```
  Use **explicit assignments** or **C99 designated initializers**:
  ```c
  struct sembuf s;
  s.sem_num = 3; s.sem_op = -1; s.sem_flg = SEM_UNDO;              /* portable */
  struct sembuf s = { .sem_num = 3, .sem_op = -1, .sem_flg = SEM_UNDO };  /* C99, portable */
  ```
  Same reason: don't **binary-write** a struct to a file; write fields individually (usually as text) in a defined order.
- **Optional macros:** a macro may be absent on some systems (e.g. `WCOREDUMP()` isn't in SUSv3). Guard with `#ifdef`:
  ```c
  #ifdef WCOREDUMP
      /* use WCOREDUMP() */
  #endif
  ```
- **Header-file variation:** required headers for a function can differ across systems. The book shows Linux's needs and marks extras `/* For portability */`. Including **`<sys/types.h>` early** is a wise portable habit (though not required on Linux).

_(Chapter 3 ends here: 3.7 is a summary; 3.8 is a single exercise about the `reboot()` magic numbers.)_

---

## Important system calls / functions
| Call | Purpose | Notes |
| ---- | ------- | ----- |
| `getppid()` | Return the parent process's PID | Used in the book to illustrate syscall overhead |
| `fopen()` (library) | Open a file as a buffered stdio stream | Layered on the `open()` syscall |
| `printf()` (library) | Formatted, buffered output | Layered on `write()`; adds formatting + buffering |
| `malloc()` / `free()` (library) | Allocate / free heap memory | Do bookkeeping over the `brk()`/`sbrk()` syscall |
| `brk()` / `sbrk()` | Adjust the program break (heap boundary) | Low-level; `malloc()` is built on top |
| `gnu_get_libc_version()` | Get the glibc version at **run time** | Returns e.g. `"2.12"`; vs compile-time `__GLIBC__` / `__GLIBC_MINOR__` |
| `errno` (variable) | Set to a positive `E...` code when a call fails | Check the return value **first**, then read `errno` |
| `perror()` | Print `msg: <error text>` for current `errno` to stderr | `#include <stdio.h>` |
| `strerror()` | Return the error string for an error number | `#include <string.h>`; not thread-safe (use `strerror_r()`) |
| `getopt()` | Parse command-line options (`-x`, `-o arg`) | Standard library; used throughout the book (Appendix B) |
| `getInt()` / `getLong()` (book helpers) | Parse a numeric CLI arg with validation | `flags`: `GN_NONNEG`, `GN_GT_0`, `GN_ANY_BASE`, `GN_BASE_8/16` |
| `errExit()`, `errMsg()`, `fatal()` (book helpers) | Print error (+`errno`) and optionally exit | From `error_functions.c`; reused by every example |

## Gotchas & things to remember
- You never call the kernel directly - you call a **glibc wrapper** that traps into the kernel.
- A failed syscall wrapper returns **-1** and sets **`errno`**; always check it.
- Check the **return value** for failure **before** reading `errno` (`errno` is *not* cleared on success, so it may hold a stale value).

## Exercises
-

## Links
- [TLPI book site](https://man7.org/tlpi/)
