# Raft - Lab 2

I'm currently watching the MIT 6.824 Distributed Systems lectures - with which the Labs are available [here](https://pdos.csail.mit.edu/6.824/schedule.html) with Lab being available [here](http://nil.csail.mit.edu/6.5840/2023/labs/lab-raft.html)

The goal of this lab was to implement the RAFT conensus algorithm - this lab was broken down into 4 parts:

- A: Leader Election ✔️
- B: Log ⌛
- C: Persistence ❌
- D: Log Compaction ❌

Currently I have implemented the Leader Election - this ReadMe will be broken down for each part

## Get Started

To run this code:

1. Clone this repository
2. Navigate into the repository's directory with `cd src/raft`
3. run `go test -run 3A` to run the tests for part A

## Part A - Leader Election

### Implementation Details

Leader election in Raft ensures that the cluster maintains at most one leader at any time to prevent conflicting commands and state corruption.

### Key Components

- **RequestVote RPC** - This RPC is initiated by candidates to start an election. The server will not vote for a candidate if:
  - The candidate’s term is less than the server's current term.
  - The server has already voted for a different candidate in the current term.

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	reply.VoteGranted = false

	if args.Term < rf.currentTerm {
			return
	}

	if args.Term > rf.currentTerm {
			rf.currentTerm = args.Term
			rf.votedFor = -1
			rf.state = Follower
			reply.Term = rf.currentTerm
	}

	if rf.votedFor == -1 || rf.votedFor == args.CandidateId {
			rf.votedFor = args.CandidateId
			reply.VoteGranted = true
			rf.electionNeeded = false
	}
}
```

- **AppendEntries RPC** - This heartbeat RPC is called by the leader. Although log details are typically sent here, in this phase of the implementation, it is used merely to maintain leader authority without transmitting any log information.

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	reply.Success = false

	if args.Term < rf.currentTerm {
		return
	}

	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = Follower
		rf.votedFor = -1
	}

	reply.Term = rf.currentTerm
	reply.Success = true

	rf.electionNeeded = false
}
```

- **Election Timeout** - Each server has a randomized election timeout set between 250 & 400ms If during this period a server:
  1. Does not receive an `AppendEntries` RPC from a valid leader, or
  2. Does not grant its vote to another candidate,
     it assumes there is no active leader and transitions to a candidate state to initiate an election

```go
func (rf *Raft) ticker() {
	rf.mu.Lock()
	rf.electionNeeded = true
	rf.mu.Unlock()
	for !rf.killed() {
		sleepDuration := 250 + (rand.Int63() % 150)
		time.Sleep(time.Duration(sleepDuration) * time.Millisecond)
		if rf.state == Leader {
			continue
		}
		if rf.electionNeeded {
			go rf.startElection()
		}
	rf.mu.Lock()
	rf.electionNeeded = true
	rf.mu.Unlock()
	}
}
```

I manage this by maintaining a `electionNeeded` flag, which is set to true if one of the conditions for not needing an election is met. This flag facilitates tracking whether an election should occur.

- **Election Proccess** - When a server starts an election, it increments its currentTerm, sets itself as a candidate, and votes for itself. It then sends a RequestVote RPC to its peers. If the server receives a majority of votes while it is still a candidate, it transitions to the leader state.

```go
func (rf *Raft) startElection() {
	rf.mu.Lock()
	rf.currentTerm += 1
	rf.votedFor = rf.me
	rf.state = Candidate
	currentTerm := rf.currentTerm
	rf.electionNeeded = false
	rf.mu.Unlock()


	voteCount := 1
	receivedVotes := make(chan bool)

	for peerIndex := range rf.peers {
		if peerIndex != rf.me {
			go func(index int) {
				args := &RequestVoteArgs {
					Term: currentTerm,
					CandidateId: rf.me,
				}
				reply := &RequestVoteReply{}
				ok := rf.sendRequestVote(index, args, reply)
				if ok {
					rf.mu.Lock()
					if reply.Term > currentTerm {
						rf.state = Follower
						rf.currentTerm = reply.Term
						rf.votedFor = -1
						rf.electionNeeded = false
					} else if reply.VoteGranted && rf.state == Candidate {
						receivedVotes <- true
					}
					rf.mu.Unlock()
				}
			}(peerIndex)
		}
	}

	for i := 0; i < len(rf.peers) -1; i++ {
		if vote := <- receivedVotes; vote {
			voteCount++
			if voteCount > len(rf.peers)/2 {
				rf.mu.Lock()
				if rf.state == Candidate {
					rf.state = Leader
				}
				rf.mu.Unlock()
				break
			}
		}
	}

}
```

### Challenges Faced

- I struggled initially with where exactly to start - having read the paper I felt I had a fairly good grip on the rough methodology needed but struggled on exactly how to get started due to the volume of information - I think the fact that the Lab broke the task down into seperate parts really helped me with further breaking down the problem

- I made use of this figure to help me really understand the state variables I would need and the logic needed in the RPC calls
  ![raft-figure-2](https://imgur.com/hFEVuur.png)

## Future Improvements

Similar to the [MapReduce](https://github.com/alfphiee/MapReduce) Lab - I'm sure there are plenty of improvements that could be made to my code. Having done some reasearch I think there were better ways to handle the ElectionTimeout part of the code - I use a flag that is checked after a 250-400ms Sleep - I believe I could make use of `time.Ticker` here although the lab did suggest not to make use of this due to complexity.

I make a lot of use of `rf.mu.Lock()` I'm not sure if this is always ideal behaviour and potentially there are places I could make more use of channels.

## Resources

- [Go Programming Language Documentation](https://golang.org/doc/)
- [MIT 6.824: Distributed Systems](https://pdos.csail.mit.edu/6.824/schedule.html)
- [Raft Paper - In Search of an Understandable Consensu Algorithm (Extended Version)](http://nil.csail.mit.edu/6.824/2021/papers/raft-extended.pdf)
