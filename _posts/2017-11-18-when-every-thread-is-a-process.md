---
layout: post
title: ["When Every Thread is a Process"]
date: 2017-11-18
---

I've tried to collect here the userspace quirks that arise when the POSIX notion of processes & threads clashes with the Linux notion of processes.

### Background: Threads Are a Figment of Your Imagination
In Linux, the distinction between processes and threads does not actually exist:[\[1\]](http://www.informit.com/articles/article.aspx?p=370047&seqNum=3)

> To the Linux kernel, there is no concept of a thread. Linux implements all threads as standard processes. The Linux kernel does not provide any special scheduling semantics or data structures to represent threads. Instead, a thread is merely a process that shares certain resources with other processes. Each thread has a unique task_struct and appears to the kernel as a normal process (which just happens to share resources, such as an address space, with other processes).

Normally this is a nonissue when working in userspace. There are some nice wrappers for raw system calls provided by your favorite libc (e.g. [glibc](https://www.gnu.org/software/libc/)). Plus with all those nice POSIX wrappers, like the entire [pthreads API](http://man7.org/linux/man-pages/man7/pthreads.7.html), it's easy to forget there is a distinction between the Linux API and the glibc wrappers. I'll be honest, before making this post I didn't realize there was a difference between [Section 2 (syscalls)](http://man7.org/linux/man-pages/man2/intro.2.html) and [Section 3 (library functions)](http://man7.org/linux/man-pages/man3/intro.3.html) of the Linux man pages. Especially considering that the pages in Section 2 are [very](http://man7.org/linux/man-pages/man2/futex.2.html) [diligent](http://man7.org/linux/man-pages/man2/clone.2.html) when noting the differences between a raw system call and its glibc wrapper.

The constant switch between userland terminology and kernel terminology can get pretty confusing. That's how one ends up having [`getpid(2)`](http://man7.org/linux/man-pages/man2/getpid.2.html) and [`kill(2)`](http://man7.org/linux/man-pages/man2/getpid.2.html), while the thread-equivalents are called [`gettid(2)`](http://man7.org/linux/man-pages/man2/gettid.2.html) and [`tgkill(2)`](http://man7.org/linux/man-pages/man2/tgkill.2.html). I'll try my best in this post to be clear, but to make life easy here's a handy table for remembering the differences, put in terms of the corresponding system calls:

| Kernel Term            | Userspace Term   | Definition                    |
|------------------------|------------------|-------------------------------|
| Thread Group ID (TGID) | Process ID (PID) | The ID returned by `getpid()` |
| Process ID (PID)       | Thread ID (TID)  | The ID returned by `gettid()` |

Remember: **TID = PID for the thread that spawns a userspace process (i.e. main)**

### procfs: How the Kernel Exposes Processes Information
The proc filesystem, or procfs, is a "pseudo-filesystem which provides an interface to kernel data structures."[\[2\]](http://man7.org/linux/man-pages/man5/proc.5.html) Generally it is mounted at the path `/proc`, so that's what I will assume going forward. As far as threads and processes are concerned, there are two relavent directories:

    /proc/[pid]
           There is a numerical subdirectory for each running process;
           the subdirectory is named by the process ID.


    /proc/[pid]/task (since Linux 2.6.0-test6)
           This is a directory that contains one subdirectory for each
           thread in the process.  The name of each subdirectory is the
           numerical thread ID ([tid]) of the thread (see gettid(2)).
           Within each of these subdirectories, there is a set of files
           with the same names and contents as under the /proc/[pid]
           directories...  In a multithreaded process, the contents
           of the /proc/[pid]/task directory are not available if the
           main thread has already terminated (typically by calling
           pthread_exit(3)).

The `/proc/[pid]/task` "is a directory that contains one subdirectory for each thread in the process." Because the main thread of a process is merely the thread where TID=PID, there should be a `/proc/[pid]/task/[pid]`. Running the command `ls /proc/[pid]/task` confirms this.

My initial assumption was that the `/proc/[pid]` and `/proc/[pid]/task/[pid]` directories *should* be identical. As the man page says in its description of `/proc/[pid]/task` (emphasis mine):
> Within each of these subdirectories, there is a set of files with *the same names and contents* as under the /proc/[pid] directories.

However those directories, although similar, are not identical. Running a diff of their contents for a process with PID=3 we see

```
ryan@ryan:~$ diff <(ls /proc/3) <(ls /proc/3/task/3)
2d1
< autogroup
4a4
> children
8d7
< coredump_filter
19d17
< map_files
24d21
< mountstats
45,46d41
< task
< timers
```

The unique files in `/proc/[pid]` are
```
autogroup
coredump_filer
map_files
mountstats
task
timers
```

The unique files in `/proc/[pid]/task/[pid]` are
```
children
```

### Acessing `/proc/[tid]`: Abusing the Lack of True Threads
The procfs is generated on the fly by the kernel when a system call is made involving `/proc`. Since `/proc` contains *process IDs*, if we run the command `ls /proc | grep -o '[9-0]*'`, then as expected we will get a list of PIDs, but not any child TIDs.

As stated earlier, a process is merely when TID = PID.

Remember, "there is a numerical subdirectory for **each** running process." To the kernel, a process and thread are treated identically. They are all tasks. So can we access threads from the root-level `/proc`? The answer is **yes**. Although threads are not listed at the root-level `/proc`, *they can be accessed there*.

For example, suppose there is a process PID=3 with 2 threads. We can see the following (other output omitted for clarity):
```
$ ls /proc
3

$ ls /proc/3/task
3 4

$ ls /proc/4
# This gives the output for thread TID 4 of PID 3!
```

On its face it appears that there is no `/proc/[tid]` directory, but if we know the desired thread ID, the kernel has no issue giving us access to that path. The thread is in fact a running process, and we have proper privileges to access it, so the kernel has no issue displaying `/proc/[tid]` directory for us.

### Terminology Quirks, or "Does this API apply to threads or processes?"
Sometimes the terminology gets confused between the kernel space notion of a process and the user space notion of a process. For example, the documentation for Linux namespaces mixes the terms freely at times. For example the manual page [namespaces(7)](http://man7.org/linux/man-pages/man7/namespaces.7.html) says

> The setns(2) system call allows the calling process to join an existing namespace.
>
> The unshare(2) system call moves the calling process to a new namespace.

However the [setns page](http://man7.org/linux/man-pages/man2/setns.2.html) says

> setns - reassociate thread with a namespace

In fact, the entire namespace API works on the level of threads, not processes and thread groups. It is entirely possible to create a process with two threads, each of which is in a separate namespace. Whether you *should* do that is another question, but either way, it is possible. I learned about this difference in the documentation the hard way the first time I used network namespaces. I had a process with two threads communicating over a socket on localhost. I ended up adding a `setns` call in one thread to switch network namespaces *before the local socket was open* and the threads were no longer able to communicate! This may be an obvious mistake to those familiar with namespaces and network namespaces in particular, but it was quite the surprise for me. Until then I had thought that the namespace was a process-level setting, not a thread-level setting.

Below is an example process that has two threads, each in a separate network namespace. The output of the `ip a` command will indeed demonstrate that the threads are associate with completely separate network namespaces:

```c
// netns_test.c
// Create a single process with 2 threads, each in a separate network
// namespace
// gcc -pthread netns_test.c

#ifndef _GNU_SOURCE
#define _GNU_SOURCE
#endif

#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <sched.h>  // setns()
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <pthread.h>

void* child(void* args) {
	int fd = open("/var/run/netns/test", O_RDONLY);
	if(fd == -1) {
		perror("open");
		return args;
	}

	if(setns(fd, CLONE_NEWNET) == -1) {
		perror("setns");
		return args;
	}

	printf("\n\nCHILD thread\n");
	system("ip a");

	return args;
}

int main() {
	pthread_t child_thread;
	pthread_create(&child_thread, NULL, child, NULL);

	sleep(1);

	printf("\n\nMAIN thread\n");
	system("ip a");

	return 0;
}
```

### TID APIs and the Absence of a gettid Wrapper
There are [a few](https://sourceware.org/bugzilla/show_bug.cgi?id=6399#c24) other Linux APIs that operate on the thread-level, not the process-level (from the user's perspective). As seen in that link as well, glibc does not expose a wrapped `gettid` function for the corresponding system call. If you really need a thread ID, you need to use the raw system call or make your own syscall wrapper:

```c
/* Wrapper to implement gettid(2) */
#ifndef _GNU_SOURCE
#define _GNU_SOURCE        /* needed for syscall() */
#endif
#include <unistd.h>
#include <sys/syscall.h>   /* For SYS_xxx definitions */
pid_t gettid() {
	return syscall(SYS_gettid);
}
```

I've seen a few arguments that working with thread IDs is a bad idea, as threads are spawned and terminated all the time, so it is very likely that a given TID will very end up as garbage. Either way, I think this is good info to know, as the "threads aren't really threads" detail of Linux occaisionally rears its head in userspace.

## Conclusions
Although threads and processes are often discussed as if they are different entities, even in the Linux man pages themselves. On Linux this is not the case. Tread lightly when you are dealing with any kind of Linux API that operates on a process or PID.

