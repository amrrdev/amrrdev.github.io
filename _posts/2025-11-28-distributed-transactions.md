---
title: "Distributed Transactions: The Hard Truth About Keeping Multiple Systems in Sync"
date: 2025-11-28
categories: ["Distributed Systems"]
tags: [distributed-transactions, 2pc, saga, consistency]
---
## Why Single-Database Transactions Don't Scale

A transaction in a single database is simple. You start a transaction, make changes, commit. Either everything commits or nothing does. The database handles this with a write-ahead log. If the database crashes mid-transaction, it recovers using that log. Either the transaction was fully written or it wasn't.

This works because one system controls everything. One transaction log. One coordinator. One source of truth.

Now split that across multiple databases. Your payment data is in one system. Your inventory is in another. Your order history is in a third. These systems don't share a log. They don't share anything. They're independent.

A user places an order. You need to charge their card, reduce inventory, and create an order record. Three writes. Three systems. If the payment succeeds but inventory fails, you charged someone for nothing. If inventory updates but payment fails, you gave away free stuff.

You can't use a local transaction anymore. The systems are separate. This is the distributed transaction problem.

## The Network Makes Everything Worse

When you write to a local database, you get a response. Success or failure. You know what happened.

When you call a remote system over the network, you send a request. Then you wait. Maybe you get a response. Maybe you don't.

If you don't get a response, what happened? Did the request fail? Did it succeed but the response got lost? Is the system just slow? You don't know.

This is the core problem. In a distributed system, you can't tell the difference between a slow response and a failed request. And you can't tell if a request succeeded when the response never arrives.

Charge a credit card. The request times out. Did Stripe charge the card? You don't know. If you retry, you might charge them twice. If you don't retry, you might not charge them at all.

This ambiguity doesn't exist in a single database. The database tells you exactly what committed. Over a network, you're guessing.

## Two-Phase Commit: The Textbook Solution

Two-phase commit (2PC) is the standard algorithm for distributed transactions. It works like this:

You have a coordinator and multiple participants. The coordinator asks everyone "can you commit this?" Each participant checks if it can do the work. If yes, it prepares. It writes everything to its local log but doesn't commit yet. It responds "yes" to the coordinator.

If all participants say yes, the coordinator tells everyone "commit." Each participant commits its local transaction. Done.

If any participant says no, the coordinator tells everyone "abort." They roll back.

**Phase 1: Prepare**

- Coordinator: "Can you all commit?"
- Each participant: Writes to local log, locks resources
- Each participant: "Yes" or "No"

**Phase 2: Commit or Abort**

- If all said yes: Coordinator: "Commit"
- If anyone said no: Coordinator: "Abort"
- Participants: Commit or roll back

This guarantees atomicity. Either everyone commits or everyone aborts.

## Why 2PC Fails in Production

Two-phase commit has a fatal flaw. If the coordinator crashes after the prepare phase but before sending commit/abort, all participants are stuck. They prepared. They locked resources. They're waiting for the coordinator to tell them what to do. The coordinator is gone.

Those participants can't commit on their own. They don't know if other participants succeeded. They can't abort either. The coordinator might come back and tell them to commit. They're blocked.

In a real system, this means:

- Locks held indefinitely
- Resources unavailable
- Timeouts everywhere
- System grinds to a halt

You can add logic to detect coordinator failure and elect a new one. But that's complex. You need consensus. You need everyone to agree on who the coordinator is. Now you're implementing Paxos or Raft on top of 2PC.

Even without crashes, 2PC is slow. You need two round trips. First to prepare, then to commit. Every participant has to respond twice. The whole transaction is as slow as the slowest participant.

Stripe takes 300ms to charge a card. Your inventory database takes 50ms. Your order database takes 100ms. Your transaction takes 300ms + 2 network round trips + coordination overhead. Probably 400-500ms total.

For high-throughput systems, this doesn't work. You can't hold locks for 500ms on every order.

## What Actually Gets Used: Sagas

A saga is a sequence of local transactions. Each step is a separate transaction in a separate system. If a step fails, you run compensating transactions to undo previous steps.

**The Booking Example**

User books a flight and hotel. Two systems. Flight booking system and hotel booking system.

**Without Saga:**

1. Start distributed transaction
2. Book flight (locks flight inventory)
3. Book hotel (locks hotel inventory)
4. Commit both or abort both
5. Hold locks for entire duration

**With Saga:**

1. Book flight (commit immediately)
2. Book hotel (commit immediately)
3. If hotel fails: Cancel flight (compensating transaction)

No distributed transaction. No locks held across systems. Each system commits its local transaction immediately.

The tradeoff: you might book the flight and fail to book the hotel. Then you have to cancel the flight. For a brief moment, the flight was booked. This is visible to the user. They might even get a confirmation email before you cancel it.

## Two Types of Sagas

**Choreography:** Each service knows what to do next. No central coordinator.

Order service receives order -> publishes "OrderCreated" event
Payment service listens -> charges card -> publishes "PaymentCompleted"
Inventory service listens -> reduces stock -> publishes "InventoryReduced"
Shipping service listens -> creates shipment

If payment fails, payment service publishes "PaymentFailed"
Inventory service listens -> doesn't reduce stock (or increases it back if it already did)

**Orchestration:** One service coordinates everything.

```md
Order Orchestrator:

1. Call payment service: Charge card
2. If success: Call inventory service: Reduce stock
3. If success: Call shipping service: Create shipment
4. If any fails: Call compensating transactions in reverse
```

Choreography is more decoupled. Services don't know about each other. They just listen to events. But it's harder to understand. The workflow is spread across multiple services. When something breaks, you're debugging event chains.

Orchestration is simpler to understand. One service has the whole workflow. You can see exactly what happens. But now that service is coupled to everything. It knows about payment, inventory, shipping. If you add a step, you modify the orchestrator.

Most teams start with orchestration. It's easier to build and debug. You move to choreography when you need more decoupling or when the orchestrator becomes a bottleneck.

## Compensating Transactions: The Hard Part

When a saga fails, you run compensating transactions. These undo previous steps. But compensation isn't always possible.

**Flight booking:** You can cancel a booking. Easy.
**Email sent:** You can't unsend an email. You can send another email saying "ignore the previous one." But the user saw the first email.
**Payment charged:** You can refund. But it takes days. The money already left their account.
**Inventory reduced:** You can add it back. But what if someone else bought that inventory in the meantime?

Compensating transactions are semantic undo, not technical undo. They logically reverse the effect. But they don't rewind time. The original action happened. Other things might have happened because of it.

This means:

- You might show inconsistent state to users temporarily
- You might send notifications you can't take back
- You might trigger side effects you can't reverse
- You need to design each step to be compensatable

## The Stripe Example Everyone Gets Wrong

Stripe has a two-step payment flow for exactly this reason.

**Wrong way:**

```md
1. Charge customer
2. Update order in database
3. If database fails: Refund customer
```

Problem: Refunds aren't instant. The customer was charged. Minutes or hours later, they get refunded. They see two transactions. They call support. Your support team has to explain it was a mistake.

**Right way:**

```md
1. Create PaymentIntent (authorized, not charged)
2. Update order in database
3. If database succeeds: Capture PaymentIntent (charge)
4. If database fails: Cancel PaymentIntent (never charged)
```

PaymentIntents let you authorize without charging. The money is held but not transferred. If your database write fails, you cancel the intent. The customer never sees a charge. No refund needed.

This is designing for compensation. Stripe built their API around the fact that you might need to undo things.

## Idempotency: The Other Half of the Solution

Sagas require retries. Payment succeeds but you don't get the response. Did it succeed? You don't know. You retry. Now you might charge them twice.

The fix: idempotency keys. Send a unique key with each request. If you retry with the same key, the system returns the previous result instead of executing again.

```ts
POST /charges
{
  "amount": 5000,
  "currency": "usd",
  "idempotency_key": "order_12345"
}
```

First request: Stripe charges the card, saves the result with that key
Second request (retry): Stripe sees the key, returns the saved result without charging again

Every step in your saga should be idempotent. You'll retry. Network failures happen. You need to be able to safely retry without creating duplicates.

## What Amazon Does: Eventually Consistent Workflows

Amazon doesn't use distributed transactions for orders. They use eventual consistency.

When you place an order:

1. Order is created (committed to database)
2. Payment is attempted asynchronously
3. Inventory is reduced asynchronously
4. Shipping is created asynchronously

Each step is independent. Each step can fail and retry. The order exists in multiple states: "PaymentPending", "PaymentComplete", "Shipped", etc.

If payment fails after several retries, the order is cancelled. You get an email. "Sorry, we couldn't charge your card."

If inventory isn't available, the order is cancelled. "Sorry, this item is out of stock."

The system is eventually consistent. For a few seconds or minutes, the order exists but payment hasn't completed. That's okay. The customer sees "Processing your order." Behind the scenes, the workflow is executing.

This is a saga with orchestration. The order service coordinates everything. Each step updates the order status. The customer sees the status change in real-time.

## The Outbox Pattern: Reliable Event Publishing

Common problem: You update your database and publish an event. The database commit succeeds. The event publish fails. Now your database is updated but other services don't know.

The outbox pattern fixes this:

1. Write your data change and an event to the outbox table in a single local transaction
2. Commit
3. Separate process reads from outbox and publishes events
4. Mark events as published

The event publish is decoupled from the database write. If publishing fails, the event is still in the outbox. The process retries. Eventually, the event gets published.

This is how you reliably trigger the next step in a saga. Your inventory service needs to tell shipping "inventory was reduced." You don't publish directly. You write to the outbox. A background job publishes.

```sql
BEGIN TRANSACTION;
  UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 123;
  INSERT INTO outbox (event_type, payload)
    VALUES ('InventoryReduced', '{"product_id": 123, "quantity": 1}');
COMMIT;
```

Now a separate process:

```python
while True:
    events = db.query("SELECT * FROM outbox WHERE published = false LIMIT 100")
    for event in events:
        message_queue.publish(event.payload)
        db.execute("UPDATE outbox SET published = true WHERE id = ?", event.id)
    sleep(1)
```

This guarantees at-least-once delivery. The event might be published multiple times if the process crashes after publishing but before marking it published. That's why you need idempotency on the receiving end.

## When to Actually Use What

**Single database transaction:** Always prefer this if possible. Don't split data across systems just because you can. If everything fits in one database, keep it there.

**Two-phase commit:** Almost never. The blocking behavior kills you. The only time I've seen it work is internal systems with low throughput and high consistency requirements. Even then, most teams regret it.

**Saga with orchestration:** Most of the time. When you have multiple services that need to coordinate. An order that touches payment, inventory, and shipping. A user signup that provisions accounts in multiple systems. The orchestrator tracks state and retries.

**Saga with choreography:** When services are loosely coupled and you want them to stay that way. Event-driven architectures. Services react to events without knowing who publishes them. Good for complex workflows where many services participate.

**Eventual consistency:** Default for everything else. Accept that systems are temporarily inconsistent. Show users "Processing" states. Use background jobs to reconcile. This is how the biggest systems in the world work.

## The Actual Hard Parts

Theory is clean. Practice is messy.

**Partial failures everywhere:** Payment succeeds, inventory call times out. Is inventory reduced or not? You don't know. You retry. Now you might reduce it twice. You need to track state. You need idempotency. You need retries with backoff.

**Debugging across services:** Order failed. Where? Payment service says it succeeded. Inventory service never got the request. Why? Network partition? Load balancer issue? Service deployed? You need distributed tracing. Request IDs that flow through every system. Logs that tell you the full story.

**Compensating transactions aren't perfect:** You can't always undo. Sometimes compensation is "send an apology email." Sometimes it's "flag for manual review." You need human processes for edge cases.

**Race conditions:** User cancels order while payment is processing. Both operations write to the order. Who wins? You need versioning. You need to detect conflicts. You need to decide what to do when they happen.

**Monitoring is critical:** How many sagas are in progress? How many succeeded? How many failed at which step? You need dashboards. You need alerts. You need to know when things are breaking before users complain.

## What to Actually Do

Start simple. Keep things in one database as long as possible. A single transaction is infinitely simpler than a saga.

When you must go distributed:

1. Use idempotency keys for everything
2. Make every step retryable
3. Design compensating transactions upfront (before you code the happy path)
4. Use the outbox pattern for reliable event publishing
5. Add request IDs and tracing from day one
6. Monitor saga completion rates

Most importantly: accept that things will be inconsistent for moments. Show that to users. "Processing your payment." "Confirming availability." "Creating your order." Users understand processing takes time. They don't understand seeing charged for something they didn't get.

## The Real Lesson

Distributed transactions are hard because distribution is hard. You're dealing with network failures, independent failures, partial failures, and timing issues all at once.

Two-phase commit tries to hide this. It pretends you can have ACID across systems. You can't. The coordinator becomes a single point of failure. The blocking makes systems slow.

Sagas accept reality. Systems are independent. Failures happen. You can't atomically commit across them. So you break it into steps. You handle failures explicitly. You compensate when things go wrong.

This is messier. You have more states. More error handling. More edge cases. But it actually works. It scales. It's what production systems use.

The companies that handle millions of transactions per day don't use distributed transactions. They use sagas, idempotency, eventual consistency, and really good monitoring. That's the real lesson.
