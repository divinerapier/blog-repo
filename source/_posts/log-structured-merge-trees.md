---
title: Log Structured Merge Trees
date: 2020-08-26 21:37:03
tags:
  - lsm tree
  - sstable
  - storage engine
---

转自 [Log Structured Merge Trees](https://medium.com/swlh/log-structured-merge-trees-9c8e2bea89e8)。

LSM tree is at the heart of most storage systems that provide high write throughput, be it a key-value storage like dynamodb/cassandra or a messaging system like pulsar which is backed by bookkeeper.

The various components of a typical LSM backed system are shown below.

![01](/images/log-structured-merge-trees/01.png)

The main reason why LSM provides high write throughput is that every write request is actually performed only **in-memory** in contrast to traditional B-Tree based implementation where the updates are done to disk which can trigger an update to an index making it very expensive.

So the obvious question is, how does LSM achieve durability? that’s where WAL comes into the picture.

## WAL

WAL is a write-ahead log that is used to provide the durability of data during system failures, what it means is that when a request for write comes in, the data is first added to a WAL file (sometimes called journal) and flushed to the disk (using [direct io](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/global_file_system/s1-manage-direct-io)) before updating the in-memory data structure.

This allows for systems to recover from the WAL if it crashes before persisting the in-memory data structure to disk.

Why not directly update the write to the disk instead of updating WAL? it’s because WAL updates are cheaper as it’s append-only and doesn’t require any restructuring of data on disk.

## MemTable

The in-memory data structure is called a memtable, there are various implementations of memtable, but you can think of memtable as just a binary search tree for the sake of simplicity.

So now for every write request, the data is appended to WAL and the memtable is updated and a successful response is returned to the client.

For java implementations, the memtable is usually stored off-heap (direct memory) to avoid GC load

## SSTable (Sorted Strings Table)

As it’s obvious that we cannot keep adding data to memtable to bloat the RAM, the memtable is frequently flushed to disk as an SSTable.

SSTable, as the name indicates, is a sorted array of keys persisted on disk.

The reason it is sorted is to make it easy to look up the data for readings.

Okay, now that is the essence of how LSM provides high throughput using a WAL, MemTable & SSTable.

Usually, even every delete request for a key is also added to memtable with a marker indicating it’s deleted and the same information is flushed to the SSTable.

## Compactor

As we keep flushing SSTables to disk, the same key may be present in multiple SSTables, although the latest data of a key is present in the most recent SSTable, it’s presence in all previous SSTable needs to be cleaned up.

This is the job of a compactor which usually runs in the background, It merges SSTables by removing redundant & deleted keys and creating a compacted/merged SSTables.

the compactor also is responsible for updating an index (typical B-Tree based index) to locate SSTable a key is present in.

## Index

The index data structure created is used to locate the correct SSTable for a key, once an SSTable is located, it is easy to locate the actual key inside the SSTable as it’s sorted, a binary search in-memory is sufficient.

Also, the size of SSTables is chosen in such a way that it corresponds to the operating system [page size](https://en.wikipedia.org/wiki/Page_(computer_memory)) (usually multiples of disk bock size) making it easier to load the data to memory faster.

Although the Index along with SSTable help in faster lookup of keys, all read requests are first consulted in the memtable as it should contain the most recent change. If the key is not in the memtable, then the index is used to identify the possible SSTable the key may be present and then search inside the SSTable in-memory.

Since every read has to check the memtable, index & SSTable to look for a key, it makes read requests very expensive especially for keys that are not present!

For keys that are recently updated, the read request will easily locate it in the memtable, but for keys not recently updated, and for keys that are not present in the system the read path is expensive!

Bloom Filters are used to improve read performance especially for the cases where the key is not present in the system.

## Bloom Filter

A Bloom filter is a probabilistic data structure, which at a high level helps you check if a key is present or not in the system with O(1) complexity in memory.

With bloom filter, False positive match is possible, which means, it may indicate a key is present although it’s not in the system. But false-negative match won’t happen, which means if bloom filter indicates a key is not present, then it is definitely not present in the system, so we could avoid taking the expensive read path.

So the presence of a bloom filter improves the read performance for keys that are missing in the system but for the keys that are present in the system, the read is still expensive as we have to look into the memtable, index and the SSTable.

## Summary

So every time you come across a system that promises high write throughput, you can assume there will be a variant of LSM tree underneath that helps achieve the throughput and also understand why reads are expensive.
