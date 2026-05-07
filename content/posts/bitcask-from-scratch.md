+++
title = 'Bitcask from scratch'
date = 2026-04-29T08:20:21+01:00
draft = false
readtime = true
toc = false
summary = "Let's implement the Bitcask storage engine from scratch in Rust"
+++

_Disclosure: I didn’t use AI tools to help me write or revise this blog post. I did use Claude to help me navigate and interpret Bitcask’s original codebase as I am not familiar with Erlang. I explicitly specify all the places where Claude helped._

I’ve been getting in database internals lately, and I wanted to have something to practice with. One of the initial steps when developing your own database system is to write the **storage engine**: the piece of the software that handles storing data to disk and retrieving it back.

There are several storage engine architectures, some more complex than others. I’ve landed on [Bitcask](https://riak.com/assets/bitcask-intro.pdf) because its architecture is simple enough it can be implemented over a weekend. Bitcask is one of the storage engines behind the now-inactive [Riak K/V](https://docs.riak.com/riak/kv/latest/index.html) database by Basho Technologies. While gathering resources for this blog post I discovered the project was forked some time ago under the name [OpenRiak](https://openriak.org/).

## Background

![](/images/bitcask-architecture.png)

There are three components to the Bitcask architecture:
- a set of **data files** that make up a database instance: at any time only one process can be writing data. There can be other processes reading data, but for the first iteration of this project I will be assuming there is a single process reading and writing data
- an in-memory data structure holding the keys and their position in one of the data files. This is called keydir in the original paper, and it is a bastardization of the memtable concept. The invariants will be worrying about is updating the keydir only after the record has been written to disk, and keeping all keys in-memory not just a portion of them. A future iteration might involve maintaining a proper memtable to batch writes 
- a merge process for getting rid of dead key-value pairs and optimizing data usage on disk akin to the familiar compaction process 

The first part of this blog post will be covering the first two components. Once those two components are in place we can worry about compaction. As the paper leaves out some nitty gritty implementation details I might refer to the original implementation (in Erlang) from time to time. You can look it up at https://github.com/basho/bitcask.

Let’s get started!

## Data directory 

Each Bitcask database is a directory of data files.

```rust
pub fn open(path: &PathBuf) -> std::io::Result<Self> {
    if let Err(err) = std::fs::create_dir(path)
        && err.kind() != ErrorKind::AlreadyExists
    {
        return Err(err);
    }
```

At any time there can be a single process writing to the file, which we can enforce by putting an auxiliary lock on the file. Linux does not enforce auxilary locks at OS level so we have to worry about unlocking the file if the program crashes.

```rust
    let lockfile = Self::get_lockfile(path)?;
    if let Err(err) = lockfile.try_lock() {
        return Err(err.into());
    }
```

Tip: Rust’s File API unlocks the file as soon as the object holding the file descriptor goes out of scope so it will handle it for us.
Tip: You cannot flock an entire the directory

At any time there can be only one active data file receiving writes. Once the file reaches a certain size (the original implementation sets 2GB by default, see bitcask.max_file_size at https://docs.riak.com/riak/kv/latest/configuring/backend/index.html) or we close the file we open a new one and never write to the old one. 

```rust
    Ok(Self {
        path: path.into(),
        lockfile,
        data_file: DataFile::new(path)?,
    })
}
```

How are we going to name the data files? We have two requirements in this regard:
1. it should be reasonably easy to get the latest data file 
2. it should be reasonably easy to rebuild a keydir so we want to iterate on the most recent data files first 

I thought about naming data files using sequential numbers (0.dat, 1.dat, and so on). It turns out the original implementation does something very similar to that. The first data file name is set to the UNIX timestamp it was created. All subsequent data file names increment that timestamp. See the code at https://github.com/basho/bitcask/blob/d8958d98d6619a1a8b9e71a7a7b19a7d9fb38ef0/src/bitcask_fileops.erl#L66. I decided to follow this approach.

```rust
    pub(crate) fn new(path: &Path) -> Result<Self> {
        let timestamp = Self::get_max_data_file_timestamp(path)?;
        let handle = File::options()
            .create_new(true)
            .write(true)
            .open(path.join(format!("{}.{}", timestamp, Self::EXT)))?;

        Ok(Self { handle, timestamp })
    }
```

## Writing data

We got the data files. We now need to write data to it. Each key value pair is a record of the form:
- CRC: record checksum, but any quick-to-compute checksum which is available in your language of choice standard library should work 
- key size
- value size
- key
- value

There are three considerations to do with respect to records:
- how to represent keys and values: Claude says the original implementation represents them as arbitrary sequence of bytes 
- how to represent deleted pairs: Claude says the original implementation represents tombstone values as the sequence of bytes corresponding to the string “bitcask_tombstone”. Later versions used the file ID and the offset of the deleted key as tombstone value
- how often to write to disk: the original implementation provides three strategies which are (1) write to disk on each insertion, (2) let the OS decide when to write to disk, (3) write to disk at regular intervals (see https://docs.riak.com/riak/kv/2.2.3/setup/planning/backend/bitcask/index.html#sync-strategy). I’ve landed on the first strategy for the first iteration of this project as it is straightforward to implement. It is not the most optimal strategy though.

Let’s look at the code handling the writing of new records. There is an additional check that I didn’t mention above which returns an error if the write would put the file size above the threshold. The error is handled by the caller.

<<code>>

My implementation delegates the responsibility of creating new data files, and writing and flushing data to it, to the DataFile struct. At all times the Rcask struct maintains a reference to one DataFile instance which represents the data file accepting writes.

## Keydir

The keydir can be populated either of two ways: incrementally, when adding or deleting a new key-value pair, or from scratch on opening an existing Bitcask database. We represent the keydir in-memory as a hash table.

The incremental approach is simple enough it doesn’t require a thorough discussion. Everytime we commit a write to disk we update the keydir with the file ID and the offset at which the key is stored.

<<code>>

The process of building the keydir from scratch is slightly more convoluted. We need to iterate over each data file, starting from the most recent ones, collecting keys as we go, discarding tombstones when we meet them. The original paper describes hint files as a way to speed up building a keydir from scratch but I decided to discard them during this first iteration. Hint files are built during the merge process.

<<code>>

Bitcask’s key feature is that reads only require a seek syscall. The get method is thus implemented as:

<<code>>
