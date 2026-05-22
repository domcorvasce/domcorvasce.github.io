+++
title = 'Bitcask from scratch'
date = 2026-05-22T14:20:21+01:00
draft = false
readtime = true
toc = false
summary = " "
+++

*Disclaimer: All writing is my own. I did use LLMs to fact-check and catch spelling errors.*

I’ve been getting into database internals lately, and I wanted to have something to practice with. One of the first steps when developing your own database system is to write the **storage engine**: the piece of the software that handles storing data to disk and retrieving it back.

There are several storage engine architectures, some more complex than others. I’ve landed on [Bitcask](https://riak.com/assets/bitcask-intro.pdf) because its architecture is simple enough it can be implemented over a weekend.

Bitcask was designed for [Riak K/V](https://docs.riak.com/riak/kv/latest/index.html) database by Basho Technologies. The project was forked some time ago by the community to create [OpenRiak](https://openriak.org/).

## How it works

![Diagram of the Bitcask architecture](/images/bitcask-architecture.png)

There are three components to the Bitcask architecture:
- a set of **data files** that make up a database instance: at any time only one process can be writing data to the active data file. There can be other processes reading data, but for the first iteration I did not assume that
- an in-memory data structure holding ALL the keys and their location on disk. This data structure is called **keydir**. The keydir is updated after the record has been written to disk
- a **merge process** for compacting data files and getting rid of dead keys

This blog post will be covering the first two components. The merge process deserves its own blog post.

The paper leaves out some implementation details so I will refer to the [original implementation](https://github.com/basho/bitcask) (in Erlang) from time to time.

You check out my Bitcask clone at https://github.com/domcorvasce/rcask.

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

At any time there can be a single process writing to the active data file.

I can guarantee this by creating a lockfile in each database directory. The program will crash when attempting to acquire an [exclusive lock](https://domcorvasce.com/posts/mandatory-and-advisory-locks/) on a lockfile that is already locked.

```rust
        let lockfile = Self::get_lockfile(path)?;
        if let Err(err) = lockfile.try_lock() {
            return Err(err.into());
        }
```

At any time there is one active data file receiving writes. A new data file is created every time the database is opened:

```rust
        let mut data_files = Self::get_data_files(path)?;

        let data_file = DataFile::create(path, Self::compute_next_data_file_timestamp(path)?)?;
        let active_data_file_timestamp = data_file.timestamp();
        data_files.insert(active_data_file_timestamp, data_file);
```

We also build the keydir from scratch reading through all the previous data files in the data directory:

```rust
        let keydir = Self::build_keydir(&data_files);
```

## Data files

The only concern I had with regards to file naming was that it should be reasonably easy to sort data files by the most recent.

First, I considered using sequential numbers (`0.dat`, `1.dat`, and so on).

The original implementation uses a [slightly different approach](https://github.com/basho/bitcask/blob/d8958d98d6619a1a8b9e71a7a7b19a7d9fb38ef0/src/bitcask_fileops.erl#L66).
The first data file name is set to the current UNIX timestamp. All subsequent data file names increment that timestamp by one.

The `Store::compute_next_data_file_timestamp` implements the logic for picking the next data file name:

```rust
    fn compute_next_data_file_timestamp(path: &Path) -> std::io::Result<u64> {
        Ok(std::fs::read_dir(path)?
            .filter_map(|entry| {
                if let Ok(file) = entry
                    && let Some(ext) = file.path().extension()
                    && ext == "bitcask"
                    && let Some(prefix) = file.path().file_prefix()
                    && let Ok(timestamp) = u64::from_str(prefix.to_str()?)
                {
                    return Some(timestamp);
                }

                None
            })
            .max()
            .map_or(
                SystemTime::now()
                    .duration_since(SystemTime::UNIX_EPOCH)
                    .unwrap()
                    .as_secs(),
                |timestamp| timestamp + 1,
            ))
    }
```

## Writing data

Each key-value pair is a record of the form

| Field      | Byte size | Description      |
| ---------- | --------- | ---------------- |
| CRC        | 8         | Record checksum  |
| Timestamp  | 8         | Record timestamp |
| Key size   | 8         |                  |
| Value size | 8         |                  |
| Key        | Variable  |                  |
| Value      | Variable  |                  |

We represent keys and values as arbitrary sequence of bytes, which length is given by the key size and value size fields.

All deleted records are marked by a tombstone value.

```rust
    pub fn delete(&mut self, key: &[u8]) -> std::io::Result<()> {
        self.put(key, Self::TOMBSTONE_VALUE.as_bytes())?;

        if self.keydir.contains_key(key) {
            self.keydir.remove(key);
        }

        Ok(())
    }
```

The original implementation provides [several strategies](https://docs.riak.com/riak/kv/2.2.3/setup/planning/backend/bitcask/index.html#sync-strategy) for syncing data to disk, which are:

1. sync to disk after each write
2. let the OS internal buffer handle it
3. let the OS decide when to write to disk
4. write to disk at regular intervals

I’ve decided to implement the first strategy for the first iteration of this project as it is pretty straightforward. As result of that, each data file is opened with the [`O_DSYNC` flag](https://man7.org/linux/man-pages/man2/open.2.html#:~:text=or%20tape%20device.-,O_DSYNC,-Write%20operations%20on):

```rust
    pub(crate) fn new(path: &Path, timestamp: u64, create_new: bool) -> Result<Self> {
        let handle = File::options()
            .create_new(create_new)
            .append(true)
            .read(true)
            .truncate(false)
            .custom_flags(libc::O_DSYNC)
            .open(path.join(format!("{}.{}", timestamp, DataFile::EXT)))?;
```

The `Store::put` starts by checking if the active data file reached its [max file size](https://docs.riak.com/riak/kv/latest/configuring/backend/index.html#:~:text=bitcask.max_file_size). If that's the case, the active data file is swapped for a new one.

```rust
    pub fn put(&mut self, key: &[u8], value: &[u8]) -> std::io::Result<()> {
        if let Ok(active_data_file) = self.get_active_data_file()
            && active_data_file.size() >= self.options.max_file_size
        {
            let new_data_file = DataFile::create(&self.path, self.active_data_file_timestamp + 1)?;
            let active_data_file_timestamp = new_data_file.timestamp();
            self.data_files
                .insert(active_data_file_timestamp, new_data_file);
            self.active_data_file_timestamp = active_data_file_timestamp;
        }
```

The method proceeds to call `DataFile::put` which writes the actual data to disk:

```rust
        let value_offset = active_data_file.put(key, value)?;
```

```rust
    pub(crate) fn put(&mut self, key: &[u8], value: &[u8]) -> Result<u64> {
        let record_timestamp = Self::get_unix_timestamp()?.as_secs();
        let entry = [
            &record_timestamp.to_ne_bytes(),
            &(key.len() as u64).to_ne_bytes(),
            &(value.len() as u64).to_ne_bytes(),
            key,
            value,
        ]
        .concat();

        self.handle
            .write_all(&[&CRC32::from(&entry).to_ne_bytes(), entry.as_slice()].concat())?;
        self.current_file_size = self.handle.stream_position()?;
        Ok(self.current_file_size - (value.len() as u64))
    }
```

Finally, `Store::put` updates the in-memory keydir.

```rust
        let keydir_entry = KeydirEntry::new(
            self.active_data_file_timestamp,
            value.len() as u64,
            value_offset,
        );
        self.keydir.insert(key.into(), keydir_entry);
```

## Reading data

The process of reading a key's value is two-step:
1. we look up the key in the keydir to fetch the data file, value size, file offset
2. we read the value at the specified file offset

```rust
    pub fn get(&self, key: &[u8]) -> std::io::Result<Vec<u8>> {
        if let Some(entry) = self.keydir.get(key) {
            let data_file = self
                .data_files
                .get(&entry.file_id)
                .ok_or(Error::other(format!(
                    "Cannot access data file {}",
                    entry.file_id
                )))?;
            return data_file.get(entry.value_size, entry.value_offset);
        }

        Err(Error::other("Missing key"))
    }
```

## Keydir

The keydir can be populated either of two ways:
* incrementally, when adding or deleting a new key-value pair
* from scratch, when opening an existing database

The incremental approach is straightforward: every time we commit a write to disk we update the keydir with the file ID and the offset at which the key is stored. We saw the code that handles this logic in the [Writing data](#writing-data) section.

The process of building the keydir from scratch is slightly more involved.
We need to iterate over each data file, populating the keydir as we iterate over each record, taking care of discarding delete records marked by the tombstone value.

```rust
    fn build_keydir(data_files: &HashMap<u64, DataFile>) -> HashMap<Vec<u8>, KeydirEntry> {
        let mut keydir = HashMap::new();
        let mut data_files = data_files.values().collect::<Vec<&DataFile>>();
        data_files.sort_by_key(|file| file.timestamp());

        for data_file in data_files {
            for record in data_file.iter().unwrap() {
                if record.value == Self::TOMBSTONE_VALUE.as_bytes() {
                    keydir.remove(&record.key);
                } else {
                    keydir.insert(
                        record.key,
                        KeydirEntry {
                            file_id: data_file.timestamp(),
                            value_size: record.value_size,
                            value_offset: record.value_offset,
                        },
                    );
                }
            }
        }

        keydir
    }
```

The original paper introduces [hint files](https://docs.riak.com/riak/kv/2.2.3/setup/planning/backend/bitcask/index.html#hint-file-crc-check) to speed up the keydir initialization. These are generated as part of the _merge process_, which I decided to not implement for now.

## Wrapping up

Overall it was fun to tackle this project. I'm pretty happy with how it turned out. I learned a lot about Linux I/O, Rust's file API, and how to approach storage engine design.

Now I feel excited about diving into more complex architectures.