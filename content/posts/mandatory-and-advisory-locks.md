+++
title = 'Mandatory and advisory locks'
date = 2026-05-12T11:43:00+02:00
draft = false
readtime = true
toc = false
summary = " "
+++

While implementing [Bitcask](https://riak.com/assets/bitcask-intro.pdf) from scratch I had to ensure that at any time one single process could append data to the _data file_.

I've enforced this invariant by creating a _lockfile_ into each database directory, acquiring a **file lock** on the lockfile every time a process opens the database. The process errors out if it tries to acquire a lock on an already locked database.

```rust
        let lockfile = File::options()
            .write(true)
            .truncate(false)
            .create(true)
            .open(path.join(Self::LOCKFILE_NAME))?;
        if let Err(err) = lockfile.try_lock() {
            return Err(err.into());
        }
```

The [`File::try_lock`](https://doc.rust-lang.org/beta/std/fs/struct.File.html#method.try_lock) (and [`File::try_lock_shared`](https://doc.rust-lang.org/beta/std/fs/struct.File.html#method.try_lock_shared)) method requests a mandatory or advisory lock on the file.

Operating systems implement two types of file locks:

* **advisory locks**: they are not enforced by the OS, instead relying on processes _cooperating_ with each other
* **mandatory locks**: they are enforced by the OS, but they are [notoriously badly implemented](https://www.kernel.org/doc/Documentation/filesystems/mandatory-locking.txt) or not implemented at all on certain UNIX-like systems (e.g. Linux, macOS)

Mandatory locks [were removed](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?h=linux-5.15.y&id=f7e33bdbd6d1bdf9c3df8bba5abcf3399f957ac3) from Linux in 2021.

Advisory locks are bound to the **open file description**, and the lock is automatically released once all **file descriptors** associated to that open file description are closed. e.g. the `lockfile` variable goes out-of-scope. There is a difference between open file descriptions and file descriptors:

* open file description: refers to the entry in the kernel's **open file table**
* file descriptor: refers to the entry in the process' file descriptors table

If a process opens the same file twice there will be two file descriptors in the process table, and two entries in the open file table.
If a process uses the [`File::try_clone`](https://doc.rust-lang.org/std/fs/struct.File.html#method.try_clone) to duplicate a file descriptor, both file descriptors will point to the same open file description entry therefore the lock will be held by both.

The `File::try_lock` method requests an **exclusive lock** which means that a single file descriptor can hold the lock. On the other hand, **shared locks** allow for multiple file descriptors to hold a lock on the same file.

Under-the-hood, Rust is using the [`flock` syscall](https://man7.org/linux/man-pages/man2/flock.2.html) on Linux.

I wasn't aware of this system call, as I really never had a use case where file locking was needed. Overall, it was very interesting to dive into file locking and learning more about operating system file management.