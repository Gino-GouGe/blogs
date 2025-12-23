---
title: Raft Log Replication
date: 2025-12-22 00:00:00+0000
categories:
    - Distributed System
    - Raft
draft: true
---


Raft's log grows indefinitely, but in a practical system, it can't grow without bound. As longer log occupies more space and needs more time to replay during startup. Snapshot is the simplest approach to compaction. In snapshot, the entire system state is written to a snapshot on non-volatile storage, then the entire log is truncated up to that ponit. In additional to snapshot, additional state `last_snapshot_index` and `last_snapshot_term` are required to support AppendEntries consistency check.

The leader would send snapshot to followers only when the leader's log is discarded the log entry that are supposed to sent to followers. This happens when leader discovers a slower follower or a new joining node without any state. The leader uses a new RPC called InstallSnapshotRPC to communicate the snapshot to the follower.

However, Raft is a consensus algorithm and has no knowledge on how to handle application states. In other words, what snapshot looks like. So it's the application's responsibility to determine the snapshot and periodically send it to the leader. For example, in a KV store, the applicate states are KV pairs snapshot at a given moment and this KV pairs should be sent to Raft periodically. The frequency of sending snapshot is usually decided upon on the # of bytes of the logs.

When it comes to implementation, a similar tricky part as AppendEntries is the way follower handles out-of-order RPCs. The follower may receive an "old" InstallSnapshotRPC with only prefix log entries. In this case, the follower should only discard the prefix log and preserve the rest. For example,
1. Leader has log entries [0,1,1,1,2,2,2,3], follower has [] and snapshot at 0
2. Leader sends [0,1,1,1,2,2,2,3] to follower
3. Leader decided to snapshot at 3, so the log becomes [2,2,2,3]
4. Leader doesn't get response from follower and the already discarded the log. So it needs to send a snapshot with included index at 3 to follower
5. Follower receives [0,1,1,1,2,2,2,3] first and successfully applies it
6. Follower receives snapshot RPC with index 3, a prefix snapshot. It should only discard the prefix log [0,1,1,1] and preserve the [2,2,2,3] in it's log
