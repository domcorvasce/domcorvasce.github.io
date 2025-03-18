+++
title = 'Notes on Implementing Database Systems: Disk and Memory Management'
date = 2025-03-18T14:40:00+01:00
draft = true
readtime = true
toc = false
summary = ""
+++

_I have been reading [Database Design and Implementation],
along with implementing my own database system in Rust to practice what I learn. This post collects my notes on memory management. You can refer to chapters 3 and 4 of the book to learn more._

Database systems aim at processing datasets larger than main memory while minimizing disk accesses as not to degrade performance

We accomplish both goals by manipulating data files as _sets of blocks_. The in-memory data structure storing the contents of a file block is called a **page**. Once a block has been loaded into a page, the database system can write and read from it without accessing the disk. Eventually, modified pages are written back to disk.

This setup resembles what the OS does with the **virtual memory** which is organised as set of fixed-size **virtual pages**. The OS handles the translation between virtual memory addresses and physical ones, and so does the database system via its **file manager**.

The OS also handles **page faults**: virtual memory is larger than physical memory so not all virtual pages can fit in memory. The OS chooses which pages to remove from memory, writes their content to disk, and frees space for other pages. Similarly, a database system's **buffer manager** maintains a **buffer pool** to process multiple file blocks (or pages) in memory.

## Blocks and pages

Blocks have a fixed-size. It usually matches the memory page size of the operating system (e.g. 4KB on Linux, 16KB on macOS).

## File manager

## Buffer manager

[Database Design and Implementation]: https://link.springer.com/book/10.1007/978-3-030-33836-7
[The Internals of PostgreSQL]: https://www.interdb.jp/pg