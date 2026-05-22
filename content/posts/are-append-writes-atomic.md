+++
title = 'Are append writes "atomic"?'
date = 2026-05-11T08:30:21+01:00
draft = false
readtime = true
toc = false
summary = " "
+++

*Disclaimer: All writing is my own. I did use LLMs to fact-check and catch spelling errors.*

I've been reimplementing the Bitcask storage engine from scratch.

Bitcask maintains all keys in-memory with pointers to the data location on disk (**keydir**), which is used to read data using a single file seek. The keydir is updated after each append write to the data file. If multiple processes were to write to the same database we would have to propagate keydir updates from one process to another.

We can avoid this complication by enforcing that *at any time there is at most one process writing to the database*, which is exactly what Bitcask does. Although the rationale behind it is to reduce the keydir maintenance cost, it made me think about file concurrency, and specifically, **append writes**.

I want to know if it is safe for multiple processes to be appending data to the same file at the same time.

The POSIX standard states that appends shall be _atomic_:

> If the `O_APPEND` flag of the file status flags is set, the file offset shall be set to the end of the file prior to each write and no intervening file modification operation shall occur between changing the file offset and the write operation.
>
> Source: https://pubs.opengroup.org/onlinepubs/009695399/functions/pwrite.html

The file offset update and the write operations run without interruptions.

Unfortunately, reality is more nuanced. For instance, Linux can transfer at most 2GB of data in a single `write` syscall. File systems such as [NFS don't support atomic writes](https://nfs.sourceforge.net/#:~:text=A9%2E%20Why%20does%20opening%20files%20with%20O%5FAPPEND%20on%20multiple%20clients%20cause%20the%20files%20to%20become%20corrupted).

A write is also said to be atomic if it protects against **torn writes**: writes where only part of the data ends up on disk. There have been long-running [efforts](https://lwn.net/Articles/1016406/) to implement torn-write protection on Linux but its availability depends on several factors: kernel version, file system, buffer size, hardware support.

To circle back to the question in the title: **are append writes "atomic"?**

Unfortunately, the answer is: it depends. As long as you don't use NFS, and limit each write to less than 2GB of data, you can reasonably assume that append writes from different processes will not corrupt each other.

The guarantees on torn-free writes are a lot weaker. The operating system cannot (yet) always guarantee that a file will not end up with partial data.
