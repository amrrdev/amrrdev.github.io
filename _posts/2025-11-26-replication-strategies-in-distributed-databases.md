---
title: "Replication Strategies in Distributed Databases: What Actually Happens When You Copy Your Data"
date: 2025-11-26
categories: ["Distributed Systems"]
tags: [replication, leader-follower, distributed-systems, database]
---
## Why Replication Exists

Your database will crash. Not maybe. It will. Hard drives fail, memory corrupts, someone runs the wrong command, power goes out. When that happens, you need another copy of your data somewhere else.

That's replication. You keep multiple copies of your data on different servers. But the moment you do this, you've introduced a set of problems that don't exist with a single server.

A user updates their profile. You write it to server A. Server B is supposed to have a copy. Does it have the update yet? If another user reads from server B right now, do they see the old profile or the new one?

This isn't theoretical. This is the core problem with replication. You have multiple copies of data, and keeping them in sync is harder than it looks.

## Leader-Based Replication (Single Leader)

This is how most databases work. One server is the leader. All writes go to the leader. Other servers are followers. They copy everything the leader does.

When a user updates data, your application sends the write to the leader. The leader writes it. Then the leader sends the change to all followers. The followers apply it. Done.

### Where Do Reads Go?

You have two options:

**Read from the leader.** Every read hits the leader. You always get the latest data. But the leader is handling all writes and all reads. This doesn't scale well.

**Read from followers.** Reads are distributed across all servers. The leader only handles writes. This scales much better. But followers might be slightly behind. You might read stale data.

Most systems read from followers. The lag is usually milliseconds. For most applications, that's fine.

But not always.

### Read-Your-Own-Writes

A user changes their username. The write goes to the leader. Success. The app redirects them to their profile. The profile loads from a follower. The follower hasn't caught up yet. The user sees their old username.

This feels broken. The user just changed it. The app said it worked. But they're seeing the old value.

The fix: after a user writes something, route their next few reads to the leader. Or include a timestamp with the write, and only serve reads from followers that are at least that fresh. Most systems do one of these.

### When the Leader Dies

The leader crashes. You have followers with all the data, but they don't accept writes. You need to promote one follower to be the new leader. This is called failover.

Three hard problems:

**Detecting the leader is down.** Is it actually down or just slow? Too aggressive and you get false positives. You promote a follower while the old leader is still running. Now you have two leaders accepting writes. Your data splits. Too conservative and you wait too long. Writes fail. Users see errors.

**Picking which follower to promote.** You want the one that's most caught up. But how do you know? If you pick one that's missing writes, you've lost data.

**Telling everyone about the new leader.** Application servers need to know where to send writes now. If some don't get updated, they'll keep trying the old leader. Those writes fail.

Failover works when you've tested it. Kill your leader in staging and watch what happens. Tune your timeouts. Make sure your detection logic is solid. The teams that skip this learn during a real outage.

### Synchronous vs Asynchronous Replication

When the leader writes data, does it wait for followers to confirm before returning success?

**Synchronous:** The leader waits for at least one follower to confirm. You're guaranteed the data exists on multiple servers before you tell the user it succeeded. But if that follower is slow or unreachable, every write blocks. Your system gets slow.

**Asynchronous:** The leader returns success immediately. Followers catch up in the background. Writes are fast. But if the leader crashes before followers catch up, recent writes are lost.

Most systems use asynchronous replication with one synchronous follower. You get speed and durability. If the leader crashes, at least one follower is guaranteed to have all the data.

## Multi-Leader Replication

You have users in the US and users in Europe. One leader in the US means European users send every write across the ocean. That's 150ms of latency. It feels slow.

So you run two leaders. One in the US, one in Europe. US users write to the US leader. European users write to the European leader. Both leaders replicate to each other.

This is faster. But now you have a new problem.

### Write Conflicts

A US user and a European user both update the same record at the same time. The US leader writes one version. The European leader writes a different version. Both leaders then try to sync. They both say "here's an update." They both realize there's a conflict.

Now what?

**Last write wins.** Pick whichever write has the later timestamp. Simple. But timestamps from different servers aren't perfectly synchronized. Clocks drift. You might pick the wrong one.

**Keep both versions.** Store both and let the application decide. When you edit a Google Doc from two devices simultaneously, Google keeps both edits and merges them. This works for some data types.

**Version vectors.** Tag every write with which server wrote it and in what order. When conflicts happen, you can see the causality. You know which write logically came first, regardless of timestamps.

All of these work. But they're complex. Multi-leader replication is powerful for global applications. But you pay for it in complexity. Most systems don't need it.

## Leaderless Replication

No leader at all. When you write data, you send it to multiple servers. You wait for a majority to confirm. Then you return success.

When you read data, you read from multiple servers. If they all agree, great. If they don't, you figure out which version is correct.

This is what Dynamo and Cassandra do.

### Why This Works

There's no single point of failure. No leader to die. You have N servers. As long as a majority are alive, the system works.

But the application has more work to do. It sends writes to multiple servers. It handles responses from multiple servers. It resolves conflicts.

### Quorums

You configure two numbers: W and R.

W is how many servers must confirm a write before you return success. R is how many servers you read from.

If W + R > N (total number of servers), you're guaranteed to read your writes. At least one server in your read set was part of your write set.

Common configs:

- N=3, W=2, R=2. You can lose one server and keep working.
- N=5, W=3, R=3. You can lose two servers.

But quorums don't solve everything. If writes happen concurrently, you still need conflict resolution. And if a write goes to W servers but one of them is slow to apply it, a read from R servers might not see it yet even with W + R > N.

### When to Use Leaderless

Shopping carts. If you add an item and the write goes to three servers but only two confirm, you can read from any two later and reconstruct your cart. Even if they disagree slightly, you merge them. Worst case, you have an extra item and you remove it.

Bank accounts? No. You can't merge balances. If two servers have different balances, one is wrong. You need stronger guarantees.

## Replication Lag: The Real Problem

All of this comes down to one thing: replication takes time. Data written to one server doesn't instantly appear on other servers. There's lag.

With asynchronous replication, this lag is unavoidable. Followers are always slightly behind. Usually milliseconds. Sometimes seconds. If the network is bad or a follower is overloaded, maybe minutes.

This creates three problems you need to think about:

### Reading Your Own Writes

You write something. You immediately read it. You might not see it yet. We covered the fix earlier: route reads to the leader for a bit after writes, or use timestamps.

### Monotonic Reads

You read data from one follower. You see version 5. You read again a second later. You hit a different follower. It's lagging. You see version 3.

Time went backwards from your perspective. This feels broken.

The fix: stick each user to one follower for reads. They always read from the same server. They might see slightly stale data, but at least it moves forward in time.

### Consistent Prefix Reads

User A sends a message. User B replies. The reply gets replicated to a follower faster than the original message. Someone reading from that follower sees the reply before the original. The conversation is out of order.

This happens when different pieces of data replicate at different speeds.

The fix depends on your data. If you need causality (replies must come after originals), you need to either write related data to the same partition or track causality explicitly with version vectors.

## What Actually Matters

Replication keeps your system alive when servers die. It scales your reads. It reduces latency for global users.

But it introduces complexity. You have to think about what happens when:

- Followers lag behind the leader
- The leader dies
- Two writes conflict
- The network splits

Most systems use single-leader replication. It's simple. It works. One server handles writes. Many servers handle reads. When the leader dies, you promote a follower.

Multi-leader is for when you need writes in multiple regions. It's faster but more complex. You have to handle conflicts.

Leaderless is for high availability. No single point of failure. But the application does more work.

Pick based on your needs. For most applications, single-leader is the right choice. Test your failover. Think about replication lag. Handle read-your-own-writes properly.

That's replication. It's not magic. It's just copying data and dealing with the problems that creates.
