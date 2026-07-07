---
layout: post
title: "Understanding Write-Ahead Logging (WAL)"
tags: [Storage, Databases, Reliability]
---

I have been studying storage internals, and Write-Ahead Logging (WAL) is one of those ideas that looks simple at first but explains a lot about how databases survive crashes.

The core rule is:

> Before changing the main data structure, write the change to a durable log.

That rule is the foundation for crash recovery in many databases and storage engines. If a process dies after the log record is safely persisted but before the main data files are updated, the system can recover by replaying the log instead of guessing what happened.

Writing every changed data page directly to disk on every commit would be expensive. A WAL makes the commit path cheaper because the database can append records sequentially, flush that log, and apply the heavier data-file work later.

PostgreSQL describes this same central idea in its [WAL documentation](https://www.postgresql.org/docs/current/wal-intro.html): data-file changes are written only after the corresponding WAL records have been flushed to permanent storage.

![WAL recovery flow](/assets/images/wal/wal-recovery-flow.svg)

## What WAL gives us

- **Durability:** once the log record is safely flushed, the change can be recovered.
- **Crash recovery:** after a restart, the database can replay log records that were not yet reflected in the main storage.
- **Sequential write path:** appending to a log is usually cheaper than immediately rewriting many random data pages.
- **Replication and backups:** many systems stream or archive WAL records to replicas or backup storage.
- **Point-in-time recovery:** if old WAL segments are archived, the database can restore a previous backup and replay logs until a chosen moment.

WAL does not make storage simple. It moves complexity into a controlled place: the log format, flush policy, recovery rules, checkpoints, and cleanup.

## The write path

A simplified WAL write path looks like this:

1. A transaction changes data.
2. The database creates one or more WAL records describing the change.
3. The WAL records are appended to the current log file.
4. On commit, the required WAL bytes are flushed to durable storage.
5. The change can be applied to memory or data pages.
6. Dirty pages can be written to the main data files later.

The important ordering is: the log must reach durable storage before the database depends on the corresponding data change being durable.

Some systems flush on every commit. Others group many commits into one flush to improve throughput. Some expose weaker durability settings where a committed transaction may be lost after a power failure. That is a performance and durability tradeoff.

## WAL record anatomy

A WAL is a sequence of records. The exact format depends on the database, but a record usually contains enough information to validate, order, and replay a change.

Common fields are: an **LSN** to place the record in the log timeline, a **record type** to say what happened, optional **transaction metadata**, a **payload length**, the **payload** itself, and often a **checksum** to detect corrupted or partial writes.

![WAL binary record format](/assets/images/wal/wal-record-format.svg)

The diagram is simplified, but the principle is the same: recovery must be able to scan a byte stream and know exactly where each record starts, where it ends, whether it is valid, and how it should be replayed.

## LSN: the timeline of the log

The LSN gives the log a stable order. It can be a record number, byte offset, or another monotonically increasing value.

LSNs help the database answer practical questions: which changes happened first, what has already been flushed, where recovery should start, which segments can be deleted, and how far a replica is behind. In many systems, data pages also store the latest LSN that affected them, so recovery can skip changes already present on disk.

## Checksums and partial writes

Crashes do not always leave files in a clean state. The last WAL record might be complete, missing bytes, or partially written. Storage can also return corrupted data.

That is why production WAL formats usually include checksums or CRCs. During recovery, the database validates record length and checksum, then stops at the first invalid tail record instead of replaying garbage bytes.

## Commit records and transaction boundaries

For a single operation, WAL recovery is easy: replay the operation.

Transactions make the problem more interesting. A transaction may generate many WAL records before it commits. After a crash, recovery must know whether those records belong to a committed transaction or an incomplete one.

Many systems solve this by writing explicit transaction records:

```text
BEGIN tx42
UPDATE account:1
UPDATE account:2
COMMIT tx42
```

If recovery sees the updates but not the commit, it should not expose the transaction as committed. Depending on the engine design, recovery may redo committed work, undo incomplete work, or use a combination of redo and undo.

## Checkpoints

If the database only appended to WAL forever, recovery would eventually become slow. Replaying months of logs after every restart is not practical.

A checkpoint records that the main data files are durable up to a known point in the log. After a checkpoint, recovery can start near that point instead of replaying from the beginning of time.

The simplified idea is:

1. Flush dirty data pages to disk.
2. Record a checkpoint LSN.
3. During recovery, start from the latest valid checkpoint.
4. Keep only WAL segments still needed for recovery, replicas, backups, or point-in-time restore.

The details differ by engine, but the motivation is similar: bound recovery time and control log growth.

## Rotation, truncation, and archiving

WAL files are usually split into segments instead of one giant file. Segmenting makes log management easier.

A database may:

- **Rotate** to a new WAL segment after the current segment reaches a size limit.
- **Recycle** old segment files by reusing their space.
- **Truncate** log records that are no longer needed.
- **Archive** old segments for backups or point-in-time recovery.
- **Retain** segments needed by replicas that are still catching up.

This is one of the operationally important parts of WAL. If old logs cannot be removed because a replica is down or archiving is broken, disk usage can grow until the database is in trouble.

## Recovery

On startup after an unclean shutdown, the database runs recovery.

A simplified recovery process looks like:

1. Find the latest valid checkpoint.
2. Open WAL from the checkpoint position.
3. Read records in LSN order.
4. Validate each record checksum and length.
5. Redo committed changes that are not reflected in data files.
6. Ignore, roll back, or clean up incomplete transactions, depending on the engine.
7. Stop at the first invalid or incomplete tail record.
8. Make the database available again.

The key idea is that WAL turns a crash into a deterministic replay problem.

## WAL and performance

WAL improves the write path because sequential appends are efficient. It also allows the database to delay random data-page writes.

But WAL is not free:

- Every durable commit eventually needs a flush.
- Checksums, encoding, and compression cost CPU.
- Checkpoints can create I/O spikes.
- Long-running transactions or replicas can prevent log cleanup.
- Large WAL files can slow recovery.
- Misconfigured durability settings can trade correctness for speed.

Good WAL design is always a balance between latency, throughput, recovery time, and storage usage.

## WAL in real systems

This idea appears in many storage systems, sometimes with slightly different names:

- **PostgreSQL:** uses WAL for crash recovery, replication, backups, and point-in-time recovery.
- **SQLite:** has a WAL mode where changes are appended to a separate WAL file before being checkpointed back into the database file.
- **MySQL/InnoDB:** uses a redo log for crash recovery. The redo log stores changes that can be replayed during initialization after an unexpected shutdown.
- **RocksDB:** uses WAL files to reconstruct memtables after a failure.
- **Cassandra:** uses a commit log before data is written into memtables and later flushed to SSTables.
- **etcd:** stores Raft entries in a WAL so the node can recover consensus state after restart.
- **Prometheus:** uses a WAL to protect the in-memory head block of its local time-series database. On restart, Prometheus replays the WAL to recover samples that were not yet compacted into blocks.
- **Grafana Loki:** uses a WAL in ingesters to persist acknowledged log data on local disk before it is flushed to long-term storage.

The names are not always identical, but the pattern is similar: append first, recover later.

## A small Go proof of concept

To make the idea concrete, I built a small Go proof of concept: [github.com/alex-carvalho/sandbox/devops/storage/wal](https://github.com/alex-carvalho/sandbox/tree/master/devops/storage/wal).

It is intentionally small: a key-value map, a binary `wal.log`, and a recovery step that rebuilds memory from the log.

The demo writes a few operations:

```go
write(wal.OpSet, "username1", "john")
write(wal.OpSet, "username2", "alex")
write(wal.OpDelete, "username1", "")
```

Each record contains:

- **LSN:** a monotonically increasing Log Sequence Number.
- **OpType:** the operation type, currently `SET` or `DELETE`.
- **KeyLen:** the number of bytes in the key.
- **ValLen:** the number of bytes in the value.
- **Key:** the key bytes.
- **Value:** the value bytes.

## Final thought

The main lesson is that durability does not require immediately rewriting the whole database. A sequential log can act as the bridge between fast in-memory changes and safe on-disk recovery.

That is the elegance of WAL: the database can move quickly while still leaving itself a path back after a crash.

## References

- [My Go WAL PoC](https://github.com/alex-carvalho/sandbox/tree/master/devops/storage/wal)
- [PostgreSQL: Write-Ahead Logging](https://www.postgresql.org/docs/current/wal-intro.html)
- [SQLite: Write-Ahead Logging](https://sqlite.org/wal.html)
- [MySQL InnoDB: Redo Log](https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html)
- [RocksDB: Write Ahead Log File Format](https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format)
- [Cassandra: Storage Engine](https://cassandra.apache.org/doc/latest/cassandra/architecture/storage-engine.html)
- [etcd: WAL package](https://pkg.go.dev/go.etcd.io/etcd/server/v3/storage/wal)
- [Prometheus: Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Grafana Loki: Write Ahead Log](https://grafana.com/docs/loki/latest/operations/storage/wal/)
