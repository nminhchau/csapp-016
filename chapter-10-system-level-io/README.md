# Chapter 10: System-Level I/O

## Scope

This chapter explains Unix system-level I/O: how programs copy data between main memory and external devices such as disks, terminals, and networks using kernel-provided file abstractions.

Although most C programs use higher-level standard I/O functions such as `printf`, `scanf`, `fgets`, and `fwrite`, those functions are built on top of Unix I/O system calls such as `open`, `read`, `write`, and `close`. Understanding Unix I/O is essential for systems programming, networking, process management, redirection, and robust error handling.

## Key ideas

### 1. Unix represents many I/O objects as files

In Unix, a file is a sequence of bytes. Many devices are represented using the same file abstraction:

```text
regular files
directories
terminals
pipes
sockets
devices
```

This gives a uniform interface:

```c
open();
read();
write();
close();
```

### 2. File descriptors identify open files

A file descriptor is a small nonnegative integer used by a process to refer to an open file.

By convention:

```text
0  standard input  (stdin)
1  standard output (stdout)
2  standard error  (stderr)
```

Example:

```c
char buf[1024];
ssize_t n = read(0, buf, sizeof(buf));  /* read from stdin */
write(1, buf, n);                       /* write to stdout */
```

### 3. `open` creates a file descriptor

`open` opens a file and returns a descriptor.

```c
int fd = open("data.txt", O_RDONLY);
if (fd < 0) {
    perror("open");
    exit(1);
}
```

Common flags:

```text
O_RDONLY   read only
O_WRONLY   write only
O_RDWR     read and write
O_CREAT    create file if it does not exist
O_TRUNC    truncate existing file
O_APPEND   append writes to end of file
```

When using `O_CREAT`, a mode argument controls permissions:

```c
int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
```

### 4. `read` and `write` can transfer fewer bytes than requested

A call to `read` or `write` may return a short count. This is not always an error.

Reasons include:

```text
EOF reached while reading
terminal input line available
network/socket behavior
interrupted system calls
kernel buffering limits
```

Example:

```c
ssize_t n = read(fd, buf, sizeof(buf));
if (n < 0) {
    perror("read");
}
```

A robust program must handle partial transfers.

### 5. The RIO package provides robust I/O wrappers

CS:APP introduces RIO (Robust I/O) to handle short counts automatically.

Two styles:

```text
unbuffered RIO   binary data, exact byte counts
buffered RIO     text lines and buffered reads
```

Conceptual robust write loop:

```c
ssize_t writen(int fd, const void *buf, size_t n) {
    size_t left = n;
    const char *p = buf;

    while (left > 0) {
        ssize_t written = write(fd, p, left);
        if (written <= 0)
            return -1;
        left -= written;
        p += written;
    }
    return n;
}
```

The key idea: keep calling `write` until all requested bytes are written or an error occurs.

### 6. File metadata describes files

Metadata includes information such as file type, size, permissions, owner, and timestamps. Unix exposes this via functions such as `stat` and `fstat`.

```c
struct stat st;
if (stat("data.txt", &st) == 0) {
    printf("size = %ld
", (long) st.st_size);
}
```

Common metadata questions:

```text
Is this a regular file?
Is this a directory?
How large is it?
What are its permissions?
```

### 7. Directories are files containing name-to-file mappings

A directory maps filenames to file information. Programs can read directory entries using functions such as `opendir`, `readdir`, and `closedir`.

```c
DIR *dir = opendir(".");
struct dirent *entry;
while ((entry = readdir(dir)) != NULL) {
    printf("%s
", entry->d_name);
}
closedir(dir);
```

### 8. Open file tables explain file sharing

The kernel tracks open files using several related data structures:

```text
per-process descriptor table
open file table
v-node / inode table
```

Important consequences:

```text
- duplicated descriptors can share the same file offset
- parent and child after fork can share open file table entries
- separate open calls usually have independent file offsets
```

Example:

```c
int fd1 = open("data.txt", O_RDONLY);
int fd2 = open("data.txt", O_RDONLY);
```

`fd1` and `fd2` generally have independent file positions.

But:

```c
int fd1 = open("data.txt", O_RDONLY);
int fd2 = dup(fd1);
```

`fd1` and `fd2` share the same open file table entry and file offset.

### 9. I/O redirection is descriptor manipulation

Shell redirection works by changing which open file a standard descriptor points to.

Example shell command:

```bash
./prog > out.txt
```

Conceptually:

```text
open out.txt
make descriptor 1 point to out.txt
exec prog
```

In C, `dup2` is commonly used:

```c
int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
dup2(fd, 1);  /* stdout now goes to out.txt */
close(fd);
```

### 10. Standard I/O is buffered on top of Unix I/O

C standard I/O (`stdio`) uses streams (`FILE *`) and buffering.

Examples:

```c
printf("hello
");
fgets(buf, sizeof(buf), stdin);
fwrite(data, size, count, fp);
```

Standard I/O is convenient and efficient for many cases, but mixing it with raw Unix I/O on the same descriptor can cause surprising bugs because stdio maintains its own buffers.

### 11. Choose I/O APIs deliberately

General guidance:

```text
Use standard I/O for ordinary disk and terminal I/O when buffering is helpful.
Use Unix I/O for low-level descriptor manipulation, network sockets, and robust systems code.
Use RIO-style wrappers when exact byte counts and short-count handling matter.
```

For network programming, robust Unix I/O is usually preferred over standard I/O because sockets and partial transfers need careful handling.

## Mini examples

### Copy stdin to stdout with Unix I/O

```c
char buf[8192];
ssize_t n;

while ((n = read(0, buf, sizeof(buf))) > 0) {
    if (write(1, buf, n) != n) {
        perror("write");
        return 1;
    }
}
if (n < 0) {
    perror("read");
}
```

This is the system-call-level idea behind a simple `cat`-like program.

### Redirect stdout to a file

```c
int fd = open("log.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
if (fd < 0) {
    perror("open");
    exit(1);
}

dup2(fd, STDOUT_FILENO);
close(fd);

printf("this goes to log.txt
");
```

After `dup2`, descriptor `1` points to `log.txt`.

### Check file type

```c
struct stat st;
if (stat(".", &st) == 0 && S_ISDIR(st.st_mode)) {
    printf("current path is a directory
");
}
```

## What to remember

- Unix I/O treats files, devices, pipes, terminals, and sockets through a common descriptor interface.
- File descriptors `0`, `1`, and `2` are standard input, output, and error.
- `open`, `read`, `write`, and `close` are the core system-level I/O calls.
- `read` and `write` can return short counts; robust code must handle them.
- RIO-style wrappers simplify robust byte and line I/O.
- `stat`/`fstat` expose file metadata.
- Directories map names to file entries and can be read programmatically.
- Descriptor tables and open file tables explain sharing after `dup`, `fork`, and multiple `open` calls.
- Redirection is implemented by remapping file descriptors, often with `dup2`.
- Avoid casually mixing standard I/O buffering with raw Unix I/O on the same descriptor.
