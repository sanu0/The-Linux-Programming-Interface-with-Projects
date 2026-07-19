# Chapter 04 - File I/O: The Universal I/O Model

## Summary
The first real look at the system-call API: **file descriptors** and the four
system calls of the **universal I/O model** - `open()`, `read()`, `write()`, `close()` -
which work on *every* kind of file (regular files, pipes, terminals, devices...).

---

## 4.1 Overview
- All I/O system calls refer to an open file by a **file descriptor (fd)** - a (usually small) **non-negative integer**. Fds refer to **all** file types: regular files, pipes, FIFOs, sockets, terminals, devices. **Each process has its own set of fds.**
- **Three standard file descriptors** - opened by the shell *before* the program starts; the program **inherits copies** of them:

  | fd | Purpose | POSIX name (`<unistd.h>`) | stdio stream |
  | -- | ------- | ------------------------- | ------------ |
  | 0 | standard input | `STDIN_FILENO` | `stdin` |
  | 1 | standard output | `STDOUT_FILENO` | `stdout` |
  | 2 | standard error | `STDERR_FILENO` | `stderr` |

  - In an interactive shell these normally refer to the **terminal**. **I/O redirection** (`>`, `<`) makes the shell repoint them *before* starting the program (the program is unaware).
  - Prefer the **POSIX names** (`STDIN_FILENO`...) over the bare numbers `0/1/2`.
  - The stdio variables `stdin`/`stdout`/`stderr` can be reassigned with **`freopen()`**, which may change the underlying fd (so don't assume `stdout` is still fd 1 after an `freopen()`).
- **The four key I/O system calls:**
  - `fd = open(pathname, flags, mode)` - open (optionally **create**) a file; returns an fd. `flags` = access mode (read/write/both) + creation options; `mode` = permissions **if creating** (omit otherwise).
  - `numread = read(fd, buffer, count)` - read up to `count` bytes into `buffer`; returns the number read, or **0 at end-of-file**.
  - `numwritten = write(fd, buffer, count)` - write up to `count` bytes from `buffer`; returns the number actually written (**may be less than `count`**).
  - `status = close(fd)` - release the fd and its kernel resources once I/O is finished.

### Example: a minimal `cp` (Listing 4-1, `copy.c`)
```c
#include <sys/stat.h>          /* S_I* permission-bit constants */
#include <fcntl.h>             /* O_* open() flag constants */
#include "tlpi_hdr.h"          /* book helpers + common headers */

#ifndef BUF_SIZE               /* allow "cc -DBUF_SIZE=..." to override */
#define BUF_SIZE 1024
#endif

int
main(int argc, char *argv[])
{
    int inputFd, outputFd, openFlags;
    mode_t filePerms;
    ssize_t numRead;
    char buf[BUF_SIZE];

    if (argc != 3 || strcmp(argv[1], "--help") == 0)
        usageErr("%s old-file new-file\n", argv[0]);   /* need exactly 2 args */

    /* Open the INPUT file read-only */
    inputFd = open(argv[1], O_RDONLY);
    if (inputFd == -1)
        errExit("opening file %s", argv[1]);

    /* Open/create the OUTPUT file: create if absent, write-only, truncate if it exists */
    openFlags = O_CREAT | O_WRONLY | O_TRUNC;
    filePerms = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP |
                S_IROTH | S_IWOTH;                     /* rw-rw-rw- if created */
    outputFd = open(argv[2], openFlags, filePerms);
    if (outputFd == -1)
        errExit("opening file %s", argv[2]);

    /* Copy loop: read a chunk, write exactly that many bytes, repeat */
    while ((numRead = read(inputFd, buf, BUF_SIZE)) > 0)
        if (write(outputFd, buf, numRead) != numRead)
            fatal("couldn't write whole buffer");
    if (numRead == -1)
        errExit("read");

    if (close(inputFd) == -1)  errExit("close input");
    if (close(outputFd) == -1) errExit("close output");
    exit(EXIT_SUCCESS);
}
```
Line-by-line:
- `#include <fcntl.h>` provides the **`O_*`** flags for `open()`; `<sys/stat.h>` provides the **`S_I*`** permission bits.
- `BUF_SIZE` (default 1024) is the chunk size; the `#ifndef` lets you override it at compile time with `-DBUF_SIZE=N`.
- `if (argc != 3 ...)` - require exactly two arguments (source + destination), else print usage and exit.
- `open(argv[1], O_RDONLY)` - open the source **read-only**; no `mode` needed (not creating). Check `-1`.
- `openFlags = O_CREAT | O_WRONLY | O_TRUNC` - **bit flags OR'd together**: create if missing, open write-only, and truncate to length 0 if it already exists.
- `filePerms = S_IRUSR | ... | S_IWOTH` - permission bits used **only if** the file is created (`rw-rw-rw-`); the real result is also filtered by the process **umask** (Ch 15).
- `open(argv[2], openFlags, filePerms)` - open/create the destination; check `-1`.
- The `while` loop: `read()` returns the number of bytes read (`> 0`); for each chunk, `write()` exactly `numRead` bytes. If `write` returns something other than `numRead`, it's a partial/failed write -> `fatal`.
- After the loop, `numRead == -1` means the final `read()` failed (not EOF) -> `errExit`.
- `close()` both fds (checked), then `exit(EXIT_SUCCESS)`.

### Beyond the book: robustly handling partial writes
- The book does a **single** `write()` per chunk and treats a short write as **fatal** (detect + give up):
  ```c
  if (write(outputFd, buf, numRead) != numRead)
      fatal("couldn't write whole buffer");
  ```
- But `write()` can legitimately transfer **fewer** bytes than requested - common on **pipes / sockets / terminals**, or due to a signal or a full disk. Robust ("real world") code **loops to write the remainder**:
  ```c
  ssize_t totalWritten = 0;                          /* progress for THIS chunk; reset per chunk */
  while (totalWritten < numRead) {
      ssize_t numWritten = write(outputFd,
                                 buf + totalWritten,        /* where the remaining data starts */
                                 numRead - totalWritten);   /* how many bytes are still left   */
      if (numWritten == -1) { /* report error, clean up */ break; }
      totalWritten += numWritten;                    /* advance progress */
  }
  ```
  - `totalWritten` is **reset to 0 per chunk** (in the outer `read` loop) but **incremented per `write()`** (in the inner loop) - two different jobs at two loop levels.
  - `buf + totalWritten` (pointer arithmetic) = start of the not-yet-written bytes; `numRead - totalWritten` = how many remain.
- Rule of thumb: for **regular disk files** `write()` almost always writes everything (so the book's single-write is usually fine), but for **pipes/sockets/terminals** you must loop. Default to the loop in serious code.

---

## 4.2 Universality of I/O
- **Universality of I/O**: the same four calls - `open()`, `read()`, `write()`, `close()` - work on **every** type of file (regular files, pipes, FIFOs, sockets, terminals, devices). A program written with just these works on **any** file type, unchanged.
- The `copy` program from 4.1 works on all of these without modification:
  - `./copy test test.old` - regular file -> regular file
  - `./copy a.txt /dev/tty` - regular file -> the terminal (behaves like `cat`)
  - `./copy /dev/tty b.txt` - terminal input -> regular file (type text; end with **Ctrl-D** = EOF)
  - `./copy /dev/pts/16 /dev/tty` - input from another (pseudo)terminal -> this terminal
- **How it's achieved:** every filesystem and device driver **implements the same set of I/O system calls**. The kernel handles the filesystem/device-specific details internally (via the **Virtual File System (VFS)** layer), so application code can ignore device-specific factors. (`/dev/*` are device files; `/dev/tty` and `/dev/pts/N` are terminals - ties back to 2.15 pseudoterminals.)
- **Escape hatch:** for operations that **don't** fit the read/write model (e.g. set a terminal's line speed, get the terminal window size, eject a disc), use the catch-all **`ioctl()`** system call (Section 4.8).

---

## 4.3 Opening a File: `open()`
```c
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *pathname, int flags, ... /* mode_t mode */);
/* Returns a file descriptor on success, or -1 on error (errno set) */
```
- Opens an existing file, or **creates + opens** a new one. `pathname` identifies the file; if it's a **symbolic link it is dereferenced** (followed).
- Returns the new **file descriptor** on success, or **-1** on error.
- **`flags`** is a bit mask: an **access mode** plus optional **creation/status flags** OR'd (`|`) together.
- **Access modes** (choose exactly one): `O_RDONLY` (read), `O_WRONLY` (write), `O_RDWR` (both).
  - Gotcha: `O_RDONLY` is `0`, so **`O_RDONLY | O_WRONLY` is NOT `O_RDWR`** - it's a logical error.
- **`mode`** = permission bits for a **newly created** file (only needed/used when `O_CREAT` is present; omit otherwise). Given as octal or `S_I*` constants OR'd. Actual perms = `mode & ~umask` (Ch 15).
- **Lowest-fd guarantee (SUSv3):** a successful `open()` uses the **lowest-numbered unused fd**. Trick to open a file *as* a specific fd:
  ```c
  if (close(STDIN_FILENO) == -1) errExit("close");   /* free fd 0 */
  fd = open(pathname, O_RDONLY);                      /* now guaranteed to be fd 0 */
  ```
- Examples (Listing 4-2):
  ```c
  fd = open("startup", O_RDONLY);                                   /* existing, read-only */
  fd = open("myfile", O_RDWR | O_CREAT | O_TRUNC, S_IRUSR | S_IWUSR); /* create/truncate, rw for owner */
  fd = open("w.log", O_WRONLY | O_CREAT | O_TRUNC | O_APPEND, S_IRUSR | S_IWUSR); /* append log */
  ```

### 4.3.1 The `open()` flags Argument
Flags fall into **three groups**:
- **Access mode flags** - `O_RDONLY`/`O_WRONLY`/`O_RDWR`. Retrievable via `fcntl(F_GETFL)`.
- **File creation flags** (affect the `open()` call / creation semantics; **cannot** be retrieved or changed later).
- **Open file status flags** (affect later I/O; **can** be retrieved/changed via `fcntl(F_GETFL/F_SETFL)`).

| Flag | Group | Purpose | SUS |
| ---- | ----- | ------- | --- |
| `O_RDONLY`/`O_WRONLY`/`O_RDWR` | access | read / write / read+write | v3 |
| `O_CREAT` | creation | create the file if it doesn't exist (needs `mode`) | v3 |
| `O_EXCL` | creation | with `O_CREAT`: **fail** if file already exists (atomic "create exclusively") | v3 |
| `O_TRUNC` | creation | truncate an existing regular file to length 0 | v3 |
| `O_DIRECTORY` | creation | fail (`ENOTDIR`) if pathname isn't a directory | v4 |
| `O_NOFOLLOW` | creation | fail (`ELOOP`) if pathname is a symbolic link (don't follow) | v4 |
| `O_CLOEXEC` | creation | set close-on-exec on the new fd (avoids fork/exec races; since 2.6.23) | v4 |
| `O_NOCTTY` | creation | don't let a terminal become the controlling terminal | v3 |
| `O_NOATIME` | creation | don't update last-access time on `read()` (backup/indexers; since 2.6.8) | - |
| `O_DIRECT` | creation | bypass the buffer cache (direct I/O) | - |
| `O_LARGEFILE` | creation | large-file support on 32-bit systems | - |
| `O_APPEND` | status | every write is atomically appended to end of file | v3 |
| `O_NONBLOCK` | status | open in non-blocking mode | v3 |
| `O_SYNC` | status | writes are synchronous (data+metadata to disk before return) | v3 |
| `O_DSYNC` | status | synchronized I/O *data* integrity (since 2.6.33) | v3 |
| `O_ASYNC` | status | signal-driven I/O (set via `fcntl` on Linux, not `open`) | - |

Key flags to remember:
- **`O_CREAT`** - create if absent; **must** supply `mode` (else the file gets random permission bits from the stack). Works even when opening only for reading.
- **`O_EXCL`** (+ `O_CREAT`) - `open()` fails with **`EEXIST`** if the file exists. The existence-check + creation is **atomic** -> the caller is guaranteed to be the file's creator (used for lock files / avoiding races). Also fails on a symlink target (security).
- **`O_TRUNC`** - wipe an existing regular file to 0 bytes (need write permission).
- **`O_APPEND`** - each `write()` **atomically** seeks to end then writes -> safe for multiple writers appending to one log.
- **`O_CLOEXEC`** - mark the fd close-on-exec so it isn't leaked into programs started by `exec()`; the flag on `open()` avoids a fork/exec race that a separate `fcntl()` would have.
- Some Linux-specific flags (`O_DIRECT`, `O_DIRECTORY`, `O_NOATIME`, `O_NOFOLLOW`) require defining **`_GNU_SOURCE`**.

### 4.3.2 Errors from `open()`
On failure `open()` returns -1 and sets `errno`, e.g.:
- **`EACCES`** - permission denied (file perms, or a directory in the path, or can't create).
- **`EISDIR`** - it's a directory and you tried to open it for writing.
- **`EMFILE`** - process hit its open-fd limit (`RLIMIT_NOFILE`).
- **`ENFILE`** - system-wide open-file limit reached.
- **`ENOENT`** - file doesn't exist (and no `O_CREAT`), or a directory in the path is missing / a dangling symlink.
- **`EROFS`** - read-only filesystem, opened for writing.
- **`ETXTBSY`** - the file is a currently-running executable (can't open it for writing).

### 4.3.3 The `creat()` System Call
```c
#include <fcntl.h>
int creat(const char *pathname, mode_t mode);   /* returns fd, or -1 */
```
- Historical (early UNIX `open()` couldn't create files). Creates+opens a file, or truncates it if it exists.
- Exactly equivalent to: `open(pathname, O_WRONLY | O_CREAT | O_TRUNC, mode)`.
- **Obsolete** now (open()'s flags give more control, e.g. `O_RDWR`); you may still see it in old code.

---

## Important system calls / functions
| Call | Purpose | Notes |
| ---- | ------- | ----- |
| `open()` | Open (or create) a file; returns an fd | `flags` (O_* bitmask) + `mode` (perms if creating) |
| `read()` | Read up to `count` bytes into a buffer | Returns bytes read; **0 = EOF**; `-1` = error |
| `write()` | Write up to `count` bytes from a buffer | Returns bytes written (**may be < count**) |
| `close()` | Release an fd and its kernel resources | Check its result too |
| `ioctl()` | Device/file-specific operations outside the universal model | The catch-all "escape hatch" (Section 4.8) |
| `creat()` | Create+open (or truncate) a file | Obsolete = `open(path, O_WRONLY\|O_CREAT\|O_TRUNC, mode)` |

## Gotchas & things to remember
- `read()` returning **0 means EOF**; `-1` means error.
- `write()` may write **fewer** bytes than requested - always check the return value (loop if needed).
- `mode` (permissions) matters **only** when `O_CREAT` is used.
- Prefer `STDIN_FILENO`/`STDOUT_FILENO`/`STDERR_FILENO` over `0`/`1`/`2`.

## Exercises
-

## Links
- [TLPI book site](https://man7.org/tlpi/)
