---
title: Raft Log Replication
date: 2025-12-22 00:00:00+0000
categories:
    - Distributed System
    - Raft
---

Following the previous blog [Leader Election](/p/raft-leader-election/), Log Replication is covered today. Once a leader has been elected, it begins servicing client requests. Each request contains a command to be executed by the replicated state machine.

## Log Replication

![Log Replication](logs_examples.png)

1. Leader appends the command to its log and issues AppendEntries RPCs in parallel to all followers to replicate the entry. Leader will keep sending RPCs until all the followers catch up even though it replies to client after majority replication (e.g. 3/5). Example of logs are shown above
2. Leader keeps track of the highest index it knows to be committed by making sure the entry has replicated on majority of the servers. This index is communicated to followers so that followers can find out.
3. Server (both leader and follower) also have a state appliedAt index to track the highest index it has applied to the state machine. Upon discovering commit index is larger than appliedAt, a dedicated thread will start to apply the entry to state machine.
4. Leader employs Consistency Check protocol to ensure the log is consistent across all servers. Whenever an inconsistent log entry detected, leader will force the follower's logs to duplicate its own.

> **Note:**  The main reason to separate appliedAt and commit index is to avoid waiting for time-consuming IO.

## Safety

Safety put additional restrictions to guarantee the leader for any term contains all committed logs

1. Election Restriction: A candidate's log must at least up-to-date when it can be elected. Up-to-date is determined by comparing index and term of the last entry. If with different terms, the log with later term is up-to-date. If with same term, the longer is more up-to-date
2. Append Entries Restriction: In addition to majority rule, leader can only commit entries in current term. In other words, leader *CANNOT* mark an entry as committed when the entry gets replicated to majority but in previous term (This sounds counter intuitive but Figure 8 in the paper explains the extreme scenario quite well and my explanation can be no match for that. So highly recommend to read the paper)

> **Note:** You may notice after introducing the Append Entries Restriction, the cluster can only make progress (i.e. increment commitIndex and apply the cmd to state machine) when there are new requests on current term. This is not optimal because the system better makes progress during idle time. One optimization (e.g. implemented by etcd) is to commit a no-op entry at current term immediately upon the election of a new leader so the system can still make progress even no client request

## Implementation

### AppendEntries RPC

Upon receiving a request, leader will append the entry to its own log and issues AppendEntriesRPC. In order to know which entries to send to different followers, leader need to maintain a `nextIndex` array wrt each follower to track the next index haven't been replicated.

> **Note:**
> There is a tricky part in follower appends it's log. The rule according to the paper is "If an existing entry conflicts with a new one, delete the existing entry and all that follow it (§5.3)".
> In other words, follower should not do anything when no conflicts. My original implementation `rf.log = append(rf.log[:prevLogIndex], entries)` is not safe b\c of the example
>
> 1. Leader issues 2 AppendEntriesRPC with entries [2,2], [2,2,3] in chronological order.
> 2. The 2nd RPC gets to follower earlier b\c of unstable network and has follower's log to be [2,2,3], already updated.
> 3. Trim from the `prevLogIndex` is definitely not safe b\c it removes committed entry. Instead, the follower should scan the log and only delete when conflicts. When no conflict detected, the follower's log should stay

```go
// sender
entry = Entry(term, command)
logs.append(entry)

func (rf *Raft) sendAppendEntries() {
    for follower in followers {
        /** AppendEntries RPC Section **/
        nextIndex = nextIndex[followerId]
        request = AppendEntriesRequest(term, logs[nextIndex, len(logs) - 1])
        go func() {
            ok = sendAppendEntriesRPC(request, reply)
            if append_entry_succeed {
                return
            }
            if discover_higher_term {
                update_term
                step_down_to_follower
                return
            }

            if receive_old_rpc {
                simply_discard
                return
            }
        }
    }
}

// receiver
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
    if args.term < currentTerm {
        reply.success = false
        reply.term = currentTerm
        return
    }
    if found_conflicts {
        reply.success = false
        reply.term = currentTerm
        reply.conflictIndex = args.prevLogIndex
        return
    }
    
    // leader's term is equal or higher
    append_the_log
}
```

### CommitIndex

To handle high concurrency, leader would just start a thread to append entries to followers without any synchronization among followers. To keep track the index replicated to every follower, leader also maintains a `matchIndex` array. After every update, it checks whether an index appears on majority of the followers and update the commit index.

```go
// async
for follower in followers {
    /** AppendEntries RPC Section **/
    ...
    matchIndex[followerId] = leaderLastIndex

    /** Update matchIndex Section **/
    for i = [leaderLastIndex, 0) {
        cnt = 0
        for follower in followers {
            if i_replicated_on_follower {
                cnt += 1
            }
        }
        if cnt > N/2 {
            commitIndex = i
            break
        }
    }
}
```

### AppliedAt

```go
func (rf *Raft) applyStateMachine() {
    for raft_is_not_killed {
        for appliedAt < commitIndex {
            apply_to_state_machine
        }
        time.Sleep(10 ms)
    }
}
```

### Consistency Check

1. AppendEntries RPCs will include the index and term of immediate preceding entry in leader's log. If followers' log doesn't match (either doesn't have the index or term mismatch), the follower will refuse the entry and the leader will decrements to a previous entry and keeps trying until matches.
2. Optimization: Above mentioned consistency check protocol backs off per entry. To reduce the number of RPCs, the protocol can be optimized to be per term by having follower sends more information. Concretely, upon a mis-matching detected

* Follower sends back
  * `conflictTerm`, the term of mismatch index
  * `ConflictIndex`, the 1st index of the conflict term
* Leader scan for its log
  * If the log has `conflictTerm`, update `nextIndex` to the last index of `conflictTerm`
  * If the log doesn't have the term, update `nextIndex` to `conflictIndex`

Leader and Follower may need special handling when follower's log doesn't have the index of leader's log at all. My implementation sets conflict term to -1 and conflict index to len(log). On the leader side, set `nextIndex` to the `conflictIndex`

```go
// consistency check per entry
for follower in followers {
    /** AppendEntries RPC Section **/
    for state == leader {
        ...
        request = AppendEntriesRequest(term, logs[nextIndex, len(logs) - 1], prevLogIndex, prevLogTerm)
        ...
        /** Consistency Check Section **/
        if conflictTerm == -1 {
            nextIndex[follower] = conflictIndex
        }
        if found_conflict {
            for log in logs reverse order {
                if conflictTerm exist {
                    nextIndex[follower] = last_index_of_conflict_term
                } else {
                    nextIndex[follower] = conflictIndex
                }
            }
        }
    }
}

// follower optimization
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
    if prevLogIndex >= len(log) || log[prevLogIndex].Term != args.PrevLogTerm {
        if prevLogIndex >= len(rf.log) {
            reply.ConflictIndex = len(rf.log)
            reply.ConflictTerm = -1
        } else {
            reply.ConflictTerm = log[prevLogIndex].Term
            reply.ConflictIndex = first_index_of_the_term
        }
        reply.Term = currentTerm
        reply.Succeeded = false
        return
    }
}
```

### Safety

Straight-forward implementation

```go
// 1. Election Restriction
func (rf *Raft) isCandidateUpToDate(RequestVoteRequest args) {
    if args.lastLogTerm < logs[-1].Term {
        reply.voteGranted = false
        return
    }

    if args.lastLogTerm > logs[-1].Term {
        reply.voteGranted = false
        return
    }

    if args.lastLogTerm == logs[-1].Index {
        reply.voteGranted = args.lastLogIndex >= len(logs) - 1
        return
    }
}

func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
    ...
    /** Additional Safety constrains to determine whether a candidate can be elected **/
    if !isCandidateUpToDate(args) {
        reject_vote
        return
    }
    ...
}

// 2. Only commit entries in current term
func (rf *Raft) sendAppendEntries() {
    ...
    /** Update matchIndex Section **/
    for i = [leaderLastIndex, 0] {
        ...
        if cnt > N/2 && logs[i].term == currentTerm {
            commitIndex = i
            break
        }
    }
    ...
}

// 3. Safety Optimization.Append a noop at current term as soon as the leader is elected
...
state = leader
logs.append(Entry(currentTerm, 'noop'))
...
```

## Ref

* [Student Guide](https://thesquareplanet.com/blog/students-guide-to-raft/)