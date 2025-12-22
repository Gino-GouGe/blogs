---
title: Raft Leader Election
date: 2025-12-16 00:00:00+0000
categories:
    - Distributed System
    - Raft
# tags:
#     - Distributed System
#     - Raft
---

This is the 1st blog about my progress in [MIT 6.5840](https://pdos.csail.mit.edu/6.824/index.html). Came up this idea to 1st summarize my understanding and implementation and 2nd share my learnings. I would cover Lab1, MapReduce and Lab2, simple KV server in future posts.

According to the [policy](https://pdos.csail.mit.edu/6.824/labs/collab.html), my github repo is private and the code blocks below are pseudo-code explaining the algorithm.

## What is Raft?

Raft is a consensus algorithm for managing replicated state machines. The key compositions of Raft are Leader Election, Log Replication and Safety. Safety puts additional constrains on Leader Election and Log Replication to guarantee the elected leader preserves all the committed states. Other than algorithm correctness and efficiency, one of the goals of Raft is understandability. i.e. to facilitate the development of intuitions that are essential to system builders. System builders can not only understand how it works but why it works to adjust/improve the algorithm for practical use-cases.

A basic Raft requires only Leader Election and Log Replication. This blog is on Leader Election (w\o Safety. Safety would be expanded later on Log Replication).

## Leader Election

Raft operates on a cluster of odd numbered servers, e.g. a cluster of 3/5/7. 5 is a typical number and can tolerate 2 failures. The state machine and transition is depicted below

![State Machine Transition](state_machine.png)

1. Every server can be in 3 states "follower", "candidate", "leader". They start at "follower" state and wait for some time pass to start Leader Election. The preset timeout is called "Election Timeout"
2. After randomized Election Timeout, one server switches state to "candidate", bumps up its term, votes for itself and sends RequestVote RPCs to other servers in the cluster
3. Candidate, the sender collects "# of votes got" and "# of requests sent" and can run into 4 scenarios below
    * Gets majority votes (of a full cluster), steps up to "leader" and send HeartBeat (empty AppendEntries) immediately to establish authority
    * Split vote, i.e. a candidate doesn't get majority votes. It restarts Leader Election. Split votes usually caused by multiple servers start Leader Election simultaneously. To prevent constant split vote, a randomized Election Timeout is employed for each server
    * Discover a higher term, update term to higher term and step down to follower
    * Upon Election Timeout, the candidate restarts Leader Election and the pending election attempt is discarded or ignored.
4. Leader needs to send periodic HeartBeat to followers to maintain its authority

> **Note:** For the 3rd scenario in bullet 3, the primary goal to update term to higher term upon discovery is to bring eligible candidate up to the matching term with in-eligible (i.e. network partitioned) candidate with in-complete logs, where the in-eligible candidate just keeps bumping up its term. It will make more sense after Safety. A rough example,

* S0 is elected as leader and sending HeartBeat in term 1
* S2 is partitioned, doesn't receive HeartBeat, goes into leader election process
* S2 never successfully elected as a leader because of no majority votes from either S0 or S1. And it keeps triggering Leader Election with term bumping up.
* After some time the replicated logs are [0,1,1] (only term), [0,1,1] and [] respectively on S0, S1, S2. S0 is partitioned, S2 gets back and new leader should be elected between S1 and S2. In this case, S1 should be the only server elected because it has the complete logs. However S1 is on term 1 and S2 can be on term 100 and leader election will always fail without bring term up-to-date upon discovery.
{: .notice--info}

## Implementation

### Raft Structure

A recommended approach is to use different threads to handle LeaderElection, AppendEntries, ApplyStateMachine etc. It's beneficial in both debugging and decoupling function call frequency. Most of the time, sleep in LeaderElection loop should longer than 2-3 times of AppendEntries RPC round trips to avoid too-frequent attempt of Leader Election. Sometimes the network package just drops and server doesn't get response

```
func (rf *Raft) MakeRaftInstance() {
 // initialize states
 go initLeaderElection() {
  for (raft_is_not_killed) {
   if (state == follower || state == candidate) && electionTimeoutPassed {
    go attemptLeaderElection()
   }
   sleep((700 + jitter)ms) 
  }
 }
 go initAppendEntries() {
  for (raft_is_not_killed) {
   if state == leader {
    go sendAppendEntries()
   }
   sleep(100ms) 
  }
 }
}
```

### RequestVoteRPC

Election Timeout is the only thing determines whether a server should attempt Leader Election and Raft instance have a dedicate state for that. It's very critical to reset Election Timeout in and only in 3 scenarios: candidate attempts Leader Election, follower receives AppendEntries RPC and follower grants a vote. Otherwise, likely Raft cluster will enter a livelock without any progress and you will never find it without serious debugging (Log every RPC request/response, grab a piece of paper, draw all the events in a chronological order, notice the cluster is always doing Leader Election from different servers without converge and have a head-scratcher... like me spending couple hours :< ). A conceptual example can be illustrated below:

* S0, S1, S2, S3, S4 are having logs [0,1,2], [0,1,2], [], [], [] respectively. And suddenly S0 is partitioned
* The only server has the complete log is S1, however S1, S2, S3, S4 have equal probability to attempt Leader Election. And S1's Leader Election is likely to be interrupted by other servers b\c of split vote or higher term
* Resetting Election Timeout suppress the Leader Election from in-eligible servers, for example S3, S4 and S1 has a lower chance to be interrupted and consequently higher chance to be elected.

```
// Sender
func (rf *Raft) attemptLeaderElection() {
 state = Candidate
 currentTerm += 1
 votedFor = self
 reset_election_timeout
 votesReceivedCnt = 1
 sendRequestCnt = 1

 for follower in followers {
  go func() {
   ok := sendRequestVote(server, &args, &reply)
   if rpc_timeout_or_fail {
    return
   }

   if discover_higher_term {
    convert_to_follower
   }

   if receive_old_rpc {
    simply_discard
   }

   if receive_vote {
    votesReceivedCnt += 1
   }
   sendRequestCnt += 1
  }()
 }

 waitUntilEither(votesReceivedCnt >= len(followers)/2 or sendRequestCnt == len(followers))
 if get_majority_votes {
  state = Leader
  go sendAppendEntries()
 }
}

// Receiver
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
 if args.Term < rf.currentTerm {
  reject_vote
  return
 }
 /** Additional Safety constrains to determine whether a candidate can be elected **/
 if args.Term > rf.currentTerm {
  approve_vote
  reset_election_timeout
  return
 }

 if args.Term == rf.currentTerm {
  approve_vote && reset_election_timeout if neverVoted or votedFor == args.CandidateId
  else approve_vote
  return
 }
}
```

### AppendEntriesRPC

In Leader Election, AppendEntries RPC is like a heartbeat sending empty entries. The only thing is to reset Election Timeout on receiver

```
// Sender
func (rf *Raft) sendAppendEntries() {
 for follower in followers {
  go func() {
   ok := sendAppendEntries(server, &args, &reply)
  }()
 }
}

// Receiver
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
 if args.Term < rf.currentTerm {
  reject_entries
  return
 }

 reset_election_timeout
}
```

## Ref

* [Raft Paper](https://raft.github.io/raft.pdf)
* [Suggestions on Raft Structure](https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt)
* [Raft Locking Advice](https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt)
