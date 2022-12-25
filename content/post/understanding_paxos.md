---
title: "Understanding Paxos"
date: 2022-12-25T10:42:59+08:00
draft: false
---
> https://people.cs.rutgers.edu/~pxk/417/notes/paxos.html

## The Consensus Problem

Suppose you have a collection of computers and want them all to agree on something. This is what **consensus** about;
consensus means agreement.

Consensus comes up frequently in distributed system design. There are various reasons why we want it: to agree on who
gets access to a resource (mutual exclusion), agree on who is in charge (elections), or to agree on a common ordering
of events among of a collection of computers (e.g. what action to take next, or state machine replication).

Replication may be the most common use of consensus. We set up collections (clusters) of servers, all of which will 
have replicated content. This provides fault tolerance: if any server dies, others are still running. It also provides
scale: clients can read data from any available server although writing data will generally require the use of all 
servers and not scale at well. For many applications, though, reads far outnumber writes. To keep the content 
replicated, we have two design choices. We can funnel all write requests through one coordinator, which will ensure 
in-order delivery to all replicas or we can send updates any system but that system will coordinate thoses update with
its replicas to ensure that all updates are applied in the same order on each replica. In the first case, we need 
consensus to elect that coordinator. In the second case, we need to run a consensus algorithm for each update to ensure
that everyone agrees on the order.

The consensus problem can be stated in a basic, generic manner: One or more systems may propose some value. How do we
get a collection of computers to agree on exactly one of those proposed values ?

The formal properties for asynchronous consensus are:
- Validity: only the proposed values can be decided. If a process decides on some value, v, then some process must have
proposed v.
- Uniform Agreement: no two correct processes (those that do not crash) can decide on different values
- Integrity: each process can decide a value at most once.
- Termination: all processes will eventually decide on a result

## Paxos

Paxos is an algorithm that is used to achieve consensus among a distributed set of computers that communicate via an
asynchronous network. One or more clients propose a value to Paxos and we have consensus when a majority of systems
running Paxos agrees on of the proposed values. Paxos is widely used and is legendary in computer science it is the 
first consensus algorithm that has been rigorously proved to be correct.

Paxos simply selects a single value from one or more values that are proposed to it and lets everyone know what that 
value is. A run of the Paxos protocol results in the selection of single proposed value. If you need to use Paxos to
create a replicated log (for a replicated state machine, for example), then you need to run Paxos repeatedly. This is
called **multi-Paxos**. 

Paxos provides **abortable consensus**. This means that some processes abort the consensus if there is contention 
while others decide on the value. Those processes that decide have to agree on the same value. Aborting allows a 
process to terminate rather than be blocked indefinitely. When a client proposes a value to Paxos, it is possible that
the proposed value might fail if there was a completing concurrent proposal that won. The client will then have to
propose the value again to another run of the Paxos algorithm.

Our assumptions for the algorithm are:
- Concurrent proposals: One or more systems may propose a value concurrently. If only one system would propose a value
then it is clear what the consensus would be. With multiple systems, we need to select one from among those values.
- Validity: The chosen value that is agreed upon must be one of the proposed values. The servers cannot just choose a
random number.
- Majority rule: Once a majority of Paxos servers agrees on one of the proposed values, we have consensus on that value
. This also implies that a majority of servers need to be functioning for the algorithm to run. To survive  m failures,
we need 2m + 1 systems.
- Asynchronous network:The network is unreliable and asynchronous : messages may get lost or arbitrarily delayed. The
network may also get partitioned.
- Fail-stop faults: System may exhibit fail-stop faults. They may restart but need to remember there previous state to
make sure they do not change their mind. Failures are not Byzantine.
- Unicasts: Communication is point-to-point. There is  no mechanism to multicast a message atomically to the set of
Paxos servers.
- Announcement: Once consensus is reached, the results can be make known to everyone.

## What we need in a distributed consensus protocol

We have an environment of multiple systems (nodes), connected by a network, where one or more of these nodes may 
concurrently propose a value (e.g., perform an operation on a server, select a coordinator, add to a log, whatever...).
We need a protocol to choose exactly one value in cases where multiple competing values may be proposed.

We will call the processes that are proposing values **proposers**. We will also have processes called **acceptors**
that will accept these values and help figure out which one value will be chosen.

### Multiple proposers, single acceptor

The simplest attempt to design a consensus protocol will use a single acceptor. We can elect one of our systesm to take
on this role. Multiple proposers will concurrently propose various to a single acceptor. The acceptor chooses one of 
those proposed values. Unfortunately, this does not handle the case where the acceptor crashes. If it crashes after
choosing a value, we will not know what value has been chosen and will have to wait for the acceptor to restart. We 
want to design a system that will be functional if the majority of nodes are running.

### Fault tolerance: multiple acceptors

To achieve fault tolerance, we will use a collection of acceptors. Each proposal will be sent to at least a mojority of
these acceptors. If a **quorum**, or majority, of these acceptors chooses a particular proposed value (proposal) then 
that value will be considered **chosen**. Even if some acceptors crash, others are up so we will know the outcome. If
an acceptor crashes after it accepted a proposal, other acceptors know that a value was chosen.

If an acceptor simply accepts the first proposed value it receives and cannot change its mind, it is possible that we
may have no majority of accepted value. Depending on the order in which messages arrive at various acceptors, each 
acceptor may choose a different value or small groups of acceptors choose different values such that there is no 
majority of acceptors that choose the same proposed values. This tells us that acceptors may need to change their mind
about which accepted value to choose. We want a value to be **chosen** only when a majority of acceptors accept the 
value.

Since we know that an acceptor may need to change its mind, we might consider just having an acceptor accept any value
given by any proposer. A majority of acceptor may accept a value, which means that value is chosen. However, anothoer
server may also tell a majority of acceptors to accept another value. Some acceptors will receive both of these 
proposals because to have a mojority means that both sets of proposals have at least one acceptor in common. That means
at least one acceptor had to change its mind about what value is ultimately chosen, violating the integrity property.
Once we choose a value, there is no going back.

To fix this, instead of just having a proposer propose a value, we will first ask it to contact a majority of acceptors
and check whether a value has already chosen. If it has, then the proposer must propose that chosen value to the other
acceptors. To implement this, we will need a two-phase protocol; check first and then provide a value.

Asking a proposer to check is not sufficient. Another proposer may come along after the checking phase and propose a 
different value. What if that second value is accepted by a majority and then the acceptors receive requests from 
the first proposer to accept its value? We again end up with two chosen values. The protocol needs to be modified to
ensure that once we accept a value, we will abort competing proposals. To do this, Paxos will impose an ordering on
proposals. Newer proposals will take precedence over old ones. If that first proposer tries to finish its protocol,
its request will fail.

## How Paxos works

### The case of characters

Paxos has three entities:

1. Proposers: Receive requests (values) from clients and try to convince acceptors to accept there proposed values.
2. Acceptors: Accept certain proposed values from proposers and let proposers know if something else was accepted.
A response from an acceptor represents a vote for a particular proposal.
3. Learners: Announce the outcome

In practice, a single node may run proposer, acceptor, and learner roles. It is common for Paxos to coexist with the
service that requires consensus (e.g., distributed storage) on a set of replicated servers, which each server taking
on all three roles rather than using separate servers dedicated to Paxos. For the sake of discussing the protocol,
however, we consider these to be independent entities.

### What Paxos does

A client sends a request to any Paxos proposer. The proposer then runs a two-phase protocol with the acceptors. Paxos
is a majority-wins proptocol. A majority avoids split-brain problems and ensures that if you made a proposal and asked
over 50% of the systems if somebody else made a proposal and they all reply non then you know for certain that no 
other system could have asked over 50% of the systems received the same answer. Because of this Paxos requires a 
majority of its servers to be running for the algorithm to terminate. A majority ensures that there is at least one node
in common from one majority to another if servers die and restart. The system requires 2m+1 servers to tolerate the
failure of m servers. As we shall see, Paxos requires a majority of acceptors. We can have one or a smaller number of
proposers and learners.

Paxos acceptors cannot forget what they accepted, so they need to keep track of the information they received from 
proposers by writing it to stable storage. This is storage such as flash memory or disk, whose contents can be retrieved
even if the process or system is restarted.

### The Paxos protocol (initial version)

Paxos is a two-phase protocol, meaning that the proposers interact with the acceptor twice. At a high level:

Phase 1: A proposer asks all the working acceptors whether anyone already received a proposal. If the answer is no, 
propose a value.

Phase 2: If a majority of acceptors agree on to this value then that is our consensus.

Right now, we will examine the mostly failure-free case. Later, we will augment the algorithm to correct for other 
failures.

When a proposer receives a client request to reach consensus on a value, the proposer must create a proposal number.
This number must have two properties:

1. It must be unique. No two proposers can come up with the same number.
2. It must be bigger than any previously used identifier used in the cluster. A proposer may use an incrementing counter
or use a nanosecond-level timestamp to achieve this. If the number is not bigger than one previously used, the proposer
will find out by having its proposal rejected and will have to try again.

Prepare 请求的作用：

1. 查找到可能已经选中的值。

2. 帮助未完成的proposal继续完成提案

3. 避免造成分裂的情况，比如 {S1, S2}, {S3}, {S4, S5}，先和 acceptor 打好招呼，不要接收比我编号更小的。比我更大的可以，因为当前 proposer
可以会挂掉，在它重启后可以选择一个更大的编号来中断之前未完成的提案过程；或者，其他的proposer替我继续这未完的旅程。

### Failure examples

#### Acceptor fails in phase 1

Suppose an acceptor fails during phase 1. That means it will not return a **PROMISE** message. As long as the proposer
still gets response from a majority of acceptors, the protocol can continue to make progress.

#### Acceptor fails in phase 2

Suppose an acceptor fails during phase 2. That means it will not be able to send back an **ACCEPTED** message. This is 
also not a problem as long as enough of the acceptors are still alive and will respond so that the proposer or learner
receives responses from a majority of acceptors.

#### Proposer fails in the Prepare phase

If the proposer fails before it sent any messages, then it is the same as it did not run at all.

What if the proposer fails after sending one or more **PREPARE** msgs? An acceptor would have sent **PROMISE** responses
back but no ACCEPT messages would follow, so there would be no consensus. Some other node will eventually run its 
version of Paxos and run as a proposer, picking its own ID. If the higher ID number works, then the algorithm runs.
Otherwise, the proposer would have its request rejected and have to pick a higher ID number.

What if the proposer fails during the ACCEPT phase? At least one ACCEPT message was sent. Some another node proposes a
new message with PREPARE(higher-ID). The acceptor responds by telling that proposer that an earlier proposal was already
accepted. PROMISE(higher-ID, <old_ID, Value>)

If a proposer gets any PROMISE responses with a value then it must choose the response with the highest accepted ID
and change its own value. It sends out: `ACCEPT(higher-ID, Value)`

If proposer fails in the ACCEPT phase, any proposer that takes over finishes the job of propagating the old value that 
was accepted. 

