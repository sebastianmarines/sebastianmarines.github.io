---
title: "Understanding Linux's File Descriptors: A Deep Dive Into '2>&1' and Redirection"
date: 2024-01-22T11:50:00-06:00
summary: "In this post, we will learn what file descriptors are, how they work, and how to implement redirections from scratch."
tags: ["linux", "c", "bash"]
author: "Sebastian Marines"
categories: [" "]
---

You have probably seen this syntax before:

```bash
$ command > file 2>&1
```

This redirects the standard output and standard error of `command` to `file`. But what does it mean? What are file descriptors? What is standard output? Let's find out.

## File descriptors

> A file descriptor is the Unix abstraction for an open input/output stream: a file, a network connection, a pipe (a communication channel between processes), a terminal, etc.[^1]

[^1]: [File Descriptors - Harvard CS61](https://cs61.seas.harvard.edu/site/ref/file-descriptors/)

Every process normally has 3 file descriptors that are open by default and are inherited from the parent process (usually the shell)

| Integer value | Name            | `<unistd.h>` symbolic constant | `<stdio.h>` file stream |
|---------------|-----------------|-------------------------------|-------------------------|
| 0             | Standard input  | `STDIN_FILENO`                | `stdin`                 |
| 1             | Standard output | `STDOUT_FILENO`               | `stdout`                |
| 2             | Standard error  | `STDERR_FILENO`               | `stderr`                |

Source: [File Descriptor - Wikipedia](https://en.wikipedia.org/wiki/File_descriptor)

When a process opens a file (remember that everything in Unix is a file, including devices like the terminal, sockets, pipes, etc.), the kernel assigns a file descriptor to it. This file descriptor is an integer that uniquely identifies the file for the process.

Internally, the kernel keeps a table of file descriptors for each process. This table is called the file descriptor table. Each entry in the table contains information about the file, such as the file offset, the file status flags, etc.

When a process opens a file, the kernel returns the lowest available file descriptor. This means that the first file opened by a process will have file descriptor 3, the second file will have file descriptor 4, and so on.

Take a look at the following C program:

```c
#include <stdio.h>
int main()
{
    fprintf(stdout, "I'm writing to stdout\n");
    fprintf(stderr, "I'm writing to stderr\n");
}
```

The program prints two lines, one to standard output and one to standard error. Let's compile and run it:

```bash
$ gcc -o print-fd print-fd.c
$ ./print-fd
I'm writing to stdout
I'm writing to stderr
```

Both lines are printed to the terminal. But why?

## Inspecting file descriptors for a process

To inspect the file descriptors for a process, we can use the [`/proc` filesystem](https://docs.kernel.org/filesystems/proc.html). This filesystem is a virtual filesystem that provides information about the system and processes running on it. It is usually mounted at `/proc`.

It contains a directory for each process running on the system. The name of the directory is the process ID. For example, the directory for the current process is `/proc/self`.

Each directory contains a lot of information about the process, but we are interested in the `fd` directory. This directory contains a symlink for each file descriptor open by the process. The name of the symlink is the file descriptor number and the target is the file the descriptor is pointing to.

```bash
$ ls -l /proc/self/fd
total 0
lrwx------. 1 sebastian sebastian 64 Jan 22 19:28 0 -> /dev/pts/0
lrwx------. 1 sebastian sebastian 64 Jan 22 19:28 1 -> /dev/pts/0
lrwx------. 1 sebastian sebastian 64 Jan 22 19:28 2 -> /dev/pts/0
lr-x------. 1 sebastian sebastian 64 Jan 22 19:28 3 -> /proc/2645/fd
```

As you can see, bash has 3 file descriptors open by default: 0, 1, and 2. All of them are pointing to the same file: `/dev/pts/0`. This is the terminal where the process is running.

## What does 2>&1 mean?

Now that we know what file descriptors are, we can understand what the syntax `2>&1` means.

```bash
$ ./print-fd > file.txt 2>&1
```

This redirects the standard output of `./print-fd` to `file.txt` and redirects the standard error (file descriptor 2) of `./print-fd` to the same place as standard output (file descriptor 1).

Let's see other examples:

```bash
$ # Redirect standard error to /dev/null
$ ./print-fd 2> /dev/null
I'm writing to stdout

$ # Redirect standard output to /dev/null and standard error to the current terminal
$ ./print-fd > /dev/null 2> $(tty)
I'm writing to stderr
```


## How redirections work

With this information, we can understand how redirections work. Let's take a look at the following command bash script:

```bash
#!/bin/bash
echo "hello" > /tmp/1234
```

This command redirects the standard output of `echo` to the file `/tmp/1234`. Let's see what happens when we run it and inspect the syscalls using `strace`[^2]:

[^2]: [strace](https://strace.io/)

```bash
strace -f -e trace=write,dup2,read,openat ./test.sh
```

> **Note:** `strace` is a tool that allows us to inspect the syscalls a process is making. It is very useful for debugging and understanding how programs work. We are limiting the output to only show the syscalls we are interested in.

```
...
read(3, "# test.sh\n\n#!/bin/bash\necho \"hel"..., 80) = 48
dup2(3, 255)                            = 255
read(255, "# test.sh\n\n#!/bin/bash\necho \"hel"..., 48) = 48
openat(AT_FDCWD, "/tmp/1234", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 1)                              = 1
write(1, "hello\n", 6)                  = 6
...
+++ exited with 0 +++
```

1. The first line of the output is the `read` syscall. It is using file descriptor 3 to read the script and trying to read 80 characters, but only reads 48. This is because the script is only 48 characters long.
2. The next 2 lines are the `dup2` syscall. It is duplicating file descriptor 3 to file descriptor 255. This is because the next syscall is also using file descriptor 3, so we need to duplicate it to another file descriptor to avoid closing it.
3. Next, is the `openat` syscall. It is opening the file `/tmp/1234` with the flags `O_WRONLY|O_CREAT|O_TRUNC`. This means that the file will be opened in write-only mode, it will be created if it doesn't exist, and it will be truncated to 0 length if it exists. It returns file descriptor 3, that was the file descriptor that was freed by the previous `dup2` syscall.
4. The next `dup2` syscall is duplicating file descriptor 3 to file descriptor 1. This is because we want to redirect the standard output of `echo` to the file `/tmp/1234` (Remember that standard output is file descriptor 1).
5. The next syscall is the `write` syscall. It is writing the string `hello\n` to file descriptor 1, which is the file `/tmp/1234`.
 
## Implementing redirections from scratch

Understanding the process bash follows to implement redirections is very useful, but what if we want to implement redirections from scratch? How can we do it?

To do this, we can follow this process:

1. Fork the process
2. In the child process, open the files to redirect to using `open`
3. Using the [dup2](https://man7.org/linux/man-pages/man2/dup.2.html) syscall, redirect standard output and standard error to the files
4. Use the [execvp](https://man7.org/linux/man-pages/man3/exec.3p.html) syscall to replace the current process with the underlying program

Let's see how this works in practice. The following C program redirects the standard output and standard error of the underlying program to the files `/out.log` and `/error.log` respectively.

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(int argc, char * argv[]) {
  pid_t pid;
  int status;

  // Fork the process
  pid = fork();
  if (pid == -1) {
    perror("fork");
    return 1;
  }

  if (pid == 0) {
    // We are in the child process

    // Open the error file, with write-only permissions and create it if it doesn't exist
    int newerr = open("/error.log", O_WRONLY | O_CREAT, 0666);
    if (newerr == -1) {
      perror("open");
      return 1;
    }

    // Open the output file, with write-only permissions and create it if it doesn't exist
    int newout = open("/out.log", O_WRONLY | O_CREAT, 0666);
    if (newout == -1) {
      perror("open");
      return 1;
    }

    // Make stderr file descriptor point to the error file
    if (dup2(newerr, STDERR_FILENO) == -1) {
      perror("dup2");
      return 1;
    }

    // Make stdout file descriptor point to the output file
    if (dup2(newout, STDOUT_FILENO) == -1) {
      perror("dup2");
      return 1;
    }

    // Replace the current process with the print-fd program
    // Notice that we are replacing the process of the forked child, not the original process
    char * newargv[] = {
      "/print-fd",
      NULL
    };
    execvp(newargv[0], newargv);

    // execvp only returns if there is an error
    perror("execvp");
    return 1;
  } else {
    // We are in the parent process
    // Wait for the child to finish
    if (waitpid(pid, & status, 0) == -1) {
      perror("waitpid");
      return 1;
    }

    // Check child's exit status
    if (WIFEXITED(status)) {
      printf("Child exited with status %d\n", WEXITSTATUS(status));
    } else {
      printf("Child did not exit cleanly\n");
    }
  }

  return 0;
}
```

Now, let's compile and run it:

```bash
$ gcc -o redirect redirect.c
$ ./redirect
Child exited with status 0
$ $ tail -n +1 /*.log
==> /error.log <==
I'm writing to stderr

==> /out.log <==
I'm writing to stdout
```

## Conclusion

In this post, we learned what file descriptors are, how they work, and how to implement redirections from scratch. I hope you found this post useful and learned something new.