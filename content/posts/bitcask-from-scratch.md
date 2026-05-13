+++
title = 'Bitcask from scratch'
date = 2026-05-13T08:20:21+01:00
draft = false
readtime = true
toc = false
summary = "Let's implement the Bitcask storage engine from scratch in Rust"
+++

*Disclaimer: All writing is my own. I did use LLMs to fact-check and catch spelling errors.*

I’ve been getting in database internals lately, and I wanted to have something to practice with. One of the initial steps when developing your own database system is to write the **storage engine**: the piece of the software that handles storing data to disk and retrieving it back.

There are several storage engine architectures, some more complex than others. I’ve landed on [Bitcask](https://riak.com/assets/bitcask-intro.pdf) because its architecture is simple enough it can be implemented over a weekend.

Bitcask was designed for [Riak K/V](https://docs.riak.com/riak/kv/latest/index.html) database by Basho Technologies. The project was forked some time ago by the community to create [OpenRiak](https://openriak.org/).

## How it works

![](/images/bitcask-architecture.png)

There are three components to the Bitcask architecture:
- a set of **data files** that make up a database instance: at any time only one process can be writing data to the active data file. There can be other processes reading data, but for the first iteration I did not assume that
- an in-memory data structure holding ALL the keys and their location on disk. This data structure is called **keydir**. The keydir is updated after the record has been written to disk
- a **merge process** for compacting data files, getting rid of dead keys and generating **hint files** for speeding up the keydir generation when an existing database is opened

This blog post will be covering the first two components. The paper leaves out some nitty-gritty implementation details so I will refer to the original implementation (in Erlang) from time to time. You can look it up at https://github.com/basho/bitcask.

Let’s get started!

## Data directory

Each Bitcask database is a directory of data files.

```rust
pub fn open(path: &PathBuf, options: StoreOptions) -> std::io::Result<Self> {
    if let Err(err) = std::fs::create_dir(path)
        && err.kind() != ErrorKind::AlreadyExists
    {
        return Err(err);
    }
```

At any time there can be a single process writing to the active data file. I guarantee this by creating a lockfile in each database directory. The program will crash if it tries to claim an [exclusive advisory lock](https://domcorvasce.com/posts/mandatory-and-advisory-locks/) on a lockfile that already has one.

```rust
    let lockfile = Self::get_lockfile(path)?;
    if let Err(err) = lockfile.try_lock() {
        return Err(err.into());
    }
```

The lock is released as soon as the file handle goes out-of-scope, so I keep the file handle as part of the database instance.

```rust
    Ok(Self {
        path: path.into(),
        options,
        lockfile,
        data_file: DataFile::new(path)?,
        keydir: HashMap::new(),
    })
}
```

At any time there is one active data file receiving writes. Once the file reaches a certain size (see [`bitcask.max_file_size`](https://docs.riak.com/riak/kv/latest/configuring/backend/index.html#:~:text=bitcask.max_file_size)) it is closed and a new data file is created. This logic is handled in the `put` method:

<<code>>

## Data files

The `DataFile::new` method is responsible for creating a new data file.

The only concern I had with regards to file naming was that it should be reasonably easy to sort data files by the most recent.
Initially, I considered using sequential numbers (`0.dat`, `1.dat`, and so on).

The original implementation follows a [slightly different approach](https://github.com/basho/bitcask/blob/d8958d98d6619a1a8b9e71a7a7b19a7d9fb38ef0/src/bitcask_fileops.erl#L66).
The first data file name is set to the current UNIX timestamp. All subsequent data file names increment that timestamp by one.

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

The utility computing the timestamp used in the file name looks like this:

```rust
    fn get_max_data_file_timestamp(path: &Path) -> Result<u64> {
        Ok(std::fs::read_dir(path)?
            .filter_map(|entry| {
                if /* is data file then extract `timestamp` from file name */
                {
                    return Some(timestamp);
                }

                None
            })
            .max()
            .map_or(Self::get_unix_timestamp(), |timestamp| timestamp + 1))
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

## Reading data

The process of reading a key's value is two-step:
1. we look up the key in the keydir to fetch the data file, value size, file offset
2. we read the value at the specified file offset

The keydir can be populated either of two ways: incrementally, when adding or deleting a new key-value pair, or from scratch on opening an existing Bitcask database.

The keydir is simply an in-memory hash table.

The incremental approach is simple enough it doesn’t require a thorough discussion. Everytime we commit a write to disk we update the keydir with the file ID and the offset at which the key is stored.

```rust
    pub fn put(&mut self, key: &[u8], value: &[u8]) -> std::io::Result<()> {
        let value_offset = self.data_file.put(key, value)?;
        let keydir_entry = KeydirEntry::new(
            self.data_file.timestamp,
            value.len().try_into().unwrap(),
            value_offset,
        );
        self.keydir.insert(key.into(), keydir_entry);

        Ok(())
    }
```

The process of building the keydir from scratch is slightly more involved.

We need to iterate over each data file, starting from the most recent one, populating the keydir as we go, and discarding tombstones when we encounter.

```
```

> Note: The original paper introduces **hint files**. These files are generated as part of the merge process, and they provide information about the keys location on disk. They speed up the keydir initialization.

The code for reading the value associated to a key is pretty simple:

```rust
            let mut buf = vec![0u8; entry.value_size.try_into().unwrap()];
            file.read_exact_at(buf.as_mut(), entry.value_offset)
                .unwrap();
```

It requires a single file seek.

## Wrapping up

* strace
* benchmark
* next steps