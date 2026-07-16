---
layout: post
title: "Understanding the Gossip Protocol"
tags: [Distributed Systems, Gossip, Go]
image: /assets/images/gossip/gossip-membership-flow.svg
image_fit: contain
---

I have been studying how distributed systems keep track of their members, and the Gossip Protocol is one of those ideas that explains how a cluster can share information without placing a central coordinator in the middle of every decision.

The core rule is:

> Each node periodically tells a few randomly chosen peers what it knows, and those peers repeat the process.

No single message reaches the entire cluster. Instead, information spreads over several rounds, much like a rumour. Given enough exchanges, the nodes converge on a similar view of the world.

Gossip is commonly used for information that can be **eventually consistent**: cluster membership, health observations, service metadata, configuration hints, or other state where a brief delay between nodes is acceptable. It is not a consensus protocol. It does not make every node agree at the same instant, nor does it decide the order of critical operations.

## Why a cluster needs gossip

Imagine a cluster with hundreds of nodes. A central registry can tell every member about a join, a departure, or a failure, but that registry becomes an important dependency. It must be reachable, it must handle all update traffic, and it must be highly available if the rest of the system depends on it.

Gossip distributes both the work and the knowledge. Each node only communicates with a few peers during a round, rather than broadcasting directly to every other member. Nodes also relay information they learned from somebody else, so a new update can move through the cluster even if its original sender does not contact everyone itself.

This makes gossip useful when the cluster should continue operating through the loss of an individual node or a central control-plane component. A node still needs an initial contact point—a seed peer—to enter the group, but that seed is an entry point rather than the permanent owner of membership.

## How information spreads

A simplified gossip round looks like this:

1. A node observes or creates an update, such as “node C joined” or “the version of this metadata is now 8.”
2. It selects one or more peers, usually at random.
3. It sends the update, or a compact summary of the updates it knows.
4. Recipients merge any information that is newer than their local state.
5. Those recipients choose peers in later rounds and relay the update further.

The random selection is important. Repeated random exchanges spread information widely without requiring every node to maintain a full broadcast schedule. Many implementations use a push model, where a node sends updates; a pull model, where it asks peers for missing state; or a push-pull combination.

## The membership view is local

A gossip-based membership system does not usually have one authoritative table that every process reads from. Each process keeps its own local view and updates it as messages arrive.

For a short time, node A may know about a join or a failure that node B has not heard about yet. That is not necessarily an error; it is the expected consequence of eventual consistency. The protocol is designed so that the difference disappears as updates are relayed.

To make merging deterministic, an update needs a version. A simple representation is:

```go
type Version struct {
    Epoch     int64
    Heartbeat uint64
}
```

The exact format varies, but the rule is the same: a receiver accepts a record only if it is newer than what it already knows. A heartbeat can advance while a process is alive. An epoch, incarnation number, or generation lets a restarted process begin a fresh sequence without being mistaken for an old message.

For example, an implementation might order versions like this:

```text
incoming epoch > local epoch
or
incoming epoch == local epoch and incoming heartbeat > local heartbeat
```

This protects the cluster from accepting stale state after messages are delayed, duplicated, or received out of order. A new epoch wins even when its heartbeat starts again at `1`.

## Failure detection is a suspicion, not a fact

Gossip is often paired with failure detection, but they are separate concerns. Gossip spreads the observation; failure detection decides when there is enough evidence to treat a peer as unavailable.

One simple approach is to track when a peer last sent newer state:

```text
no newer heartbeat for longer than the timeout → suspect the peer
```

The word *suspect* matters. A node may be slow, temporarily unreachable, partitioned from the observer, or affected by packet loss. None of those conditions proves that its process has stopped. More complete membership protocols therefore use suspicion periods, indirect checks, and refutation mechanisms before spreading a final failure state.

Timeouts are a real tradeoff:
  - A shorter timeout reacts quickly but increases false positives.
  - A longer timeout is more conservative but leaves failed members visible for longer.
  - A network partition can make two healthy groups each conclude that the other group is unavailable.

The important lesson is that failure detection describes an observer's view of the network, not an absolute fact about another machine.

## What gossip gives us

Gossip is a good fit when a system values availability, decentralisation, and scalable dissemination.

- **No central dissemination bottleneck:** each member helps distribute updates.
- **Resilience to individual failures:** the loss of one node does not prevent other nodes from sharing information.
- **Scalable communication pattern:** a node talks to only a small subset of the cluster in each round.
- **Simple membership growth:** a new member can join through a seed peer and learn about the rest of the cluster over time.
- **Natural fit for soft state:** information such as membership and health can tolerate brief disagreement while it converges.

These properties are why gossip appears in systems such as [Apache Cassandra](https://cassandra.apache.org/doc/latest/cassandra/architecture/gossip.html), which uses it to share cluster state, and in HashiCorp's [Memberlist](https://github.com/hashicorp/memberlist), a library for cluster membership and failure detection.

## The costs and limitations

Gossip avoids a central bottleneck by accepting different costs. It is not the right primitive for every distributed-systems problem.

- **Eventual, not immediate, consistency:** different nodes can briefly hold different views of the cluster.
- **Network overhead:** nodes keep sending background messages even when little state changes.
- **Convergence time:** an update needs several rounds to reach most or all members.
- **False failure reports:** timeouts and partitions can make healthy nodes look unavailable.
- **Payload management:** naively sending the full membership table becomes expensive as the cluster grows.
- **No agreement on critical decisions:** gossip cannot safely replace consensus when the system needs one ordered, durable answer—such as electing a leader, committing a transaction, or issuing a unique lock.

Good implementations control these costs with bounded fan-out, compact state, retransmission limits, failure-suspicion rules, and tuning for the network they operate in. The details differ by system, but the central balance is consistent: faster dissemination costs more traffic; more conservative detection reduces false positives but delays reaction.

## A small Go proof of concept

To make the protocol concrete, I built a [minimal Go implementation](https://github.com/alex-carvalho/sandbox/tree/master/distributed-systems/gossip/minimal). It uses UDP, gossips a full table of `(epoch, heartbeat)` values, selects random active peers, and marks a peer as failed when its heartbeat stops advancing past a timeout.

It is intentionally a learning tool rather than a production design. In particular, sending the complete table makes the merge rule easy to see, but makes each message grow with membership size.

I then rebuilt the same membership experiment with [HashiCorp Memberlist](https://github.com/alex-carvalho/sandbox/tree/master/distributed-systems/gossip/memberlist). The application provides node identity, bind address, seed peers, and join/leave callbacks, while Memberlist handles the specialised membership and failure-detection machinery. A clean shutdown calls `Leave`, so the rest of the cluster can learn about the departure without waiting for a timeout.

## Final thought

Gossip does not create a perfectly synchronized cluster. It gives a distributed system a decentralised way to spread information until members converge, without making one node responsible for every update.

The central tradeoff is also its strength: gossip gives up immediate global agreement in exchange for resilience and scalable dissemination. The two PoCs helped me see the mechanics, but the protocol is useful because of that broader design choice.

## References

- [Minimal heartbeat gossip PoC](https://github.com/alex-carvalho/sandbox/tree/master/distributed-systems/gossip/minimal)
- [HashiCorp Memberlist gossip PoC](https://github.com/alex-carvalho/sandbox/tree/master/distributed-systems/gossip/memberlist)
- [HashiCorp Memberlist](https://github.com/hashicorp/memberlist)
- [Apache Cassandra: Gossip](https://cassandra.apache.org/doc/latest/cassandra/architecture/gossip.html)
