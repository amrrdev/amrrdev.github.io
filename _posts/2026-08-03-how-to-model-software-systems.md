---
title: "How to Model Software Systems: From Vague Ideas to Working Programs"
date: 2026-08-03
categories: ["Software Design"]
tags: [software-design, modeling, architecture, ddd]
---
## Introduction

There is a moment in programming that nobody warns you about properly.

You know the language. You know the database. You know the framework. You can write handlers, queries, goroutines, tests, components, migrations. If someone gives you a clear task, you can usually implement it.

Then someone says:

"Build a message queue."

And suddenly the problem is not syntax anymore.

What is the queue, exactly? Is it a list? A table? A channel? What is a producer? Is it an object in the code, a separate service, or just a role? What does a consumer need to remember? When does a message disappear? What happens if a consumer crashes after doing the work but before acknowledging the message?

This is the gap between knowing tools and knowing how to model.

Most programming material teaches tools. It teaches the language, the library, the API, the storage engine, the command. These are necessary, but they are not enough. The harder skill is taking an unclear process in the real world and turning it into a small set of concepts that can be represented in code.

This article is about that skill.

Not design patterns. Not UML. Not a lecture about clean architecture. Modeling is simpler and more practical than that:

> Modeling is deciding what things exist in your system, what state they own, what operations can change that state, and what must never become false.

If you can learn to do that deliberately, large parts of programming become less mysterious.

---

## The Real Problem Is Usually Not the Tool

Suppose we want to build a message queue.

A beginner may ask:

- Should I use Redis?
- Should I use Kafka?
- Should I use RabbitMQ?
- Should I use Postgres?
- Should I write it in Go?
- Should workers be goroutines?

These are useful questions, but they are not the first questions.

The first questions are:

- What is a message?
- Where does it live before it is consumed?
- Who is allowed to remove it?
- What does "consumed" mean?
- Can two consumers receive the same message?
- What happens if processing fails?
- What happens if a consumer dies halfway through?
- Do we care about ordering?
- Do we care about duplicate delivery?
- Do we care about losing messages?

Notice that none of these questions mention Redis, Kafka, Go, or Postgres.

That is the point. Tools implement a model. They do not replace the need for one.

If the model is unclear, every tool feels confusing. If the model is clear, the tool choice becomes much easier. You can look at Redis Streams, SQS, Kafka, Postgres `SKIP LOCKED`, or an in-process Go channel and ask a concrete question: does this tool support the behavior my model needs?

---

## What Is a Model?

A model is a simplified representation of the system you are building.

It is not the whole reality. It deliberately ignores most details. A good model keeps only the details that matter for the behavior of the program.

For a message queue, the real world may contain servers, disks, network packets, customer requests, retry storms, worker processes, logs, metrics, dashboards, and deployment scripts.

The first useful model might contain only this:

```
Producer -> Queue -> Consumer
              |
           Message
```

This is not enough to implement a reliable queue, but it is a start. It says:

- Producers put messages into the queue.
- Consumers take messages out of the queue.
- The queue owns the waiting messages.

Then we refine it.

```
Producer -> Queue -> Delivery -> Consumer
              |
           Message
```

The new word is `Delivery`. This is a major modeling improvement.

Why? Because receiving a message is not the same as deleting it. A queue may hand a message to a consumer, but keep the message until the consumer acknowledges successful processing. That temporary relationship between message and consumer is its own concept.

Without `Delivery`, many bugs hide in vague language:

"The consumer got the message."

What does "got" mean? Is it removed? Is it locked? Is it invisible? Can someone else get it? For how long?

With `Delivery`, the questions become sharper:

- A message can be delivered to one consumer.
- A delivery has a deadline.
- If the consumer acknowledges before the deadline, the message is complete.
- If the deadline expires, the delivery is cancelled and the message becomes available again.

That is modeling. You are not adding complexity for its own sake. You are giving a real behavior a name so the code can handle it explicitly.

---

## Start With Stories, Not Structures

A common mistake is starting with nouns.

"I need a `Queue` class, a `Producer` class, a `Consumer` class..."

Sometimes that works. Often it creates fake objects that sound correct but do not actually clarify the system.

Start with stories instead.

For the message queue:

```
1. A producer publishes a message.
2. A consumer receives a message.
3. The consumer processes the message successfully.
4. The consumer acknowledges the message.
5. The queue removes the message.
```

That is the happy path. It is useful, but incomplete. Real models are shaped by the unhappy paths:

```
1. A producer publishes a message.
2. A consumer receives the message.
3. The consumer crashes before acknowledging it.
4. The queue waits for the delivery timeout.
5. The message becomes available again.
6. Another consumer receives it.
```

Now the model needs more than a list.

It needs message state:

```
ready -> in_flight -> acknowledged
ready -> in_flight -> ready
ready -> in_flight -> dead
```

In words:

- `ready`: the message can be delivered.
- `in_flight`: the message has been delivered but not acknowledged.
- `acknowledged`: the message was processed and can be removed.
- `dead`: the message failed too many times and should not be retried automatically.

Now we can see the system more clearly.

A message queue is not primarily a data structure. It is a state machine around delivery.

That sentence is worth sitting with. A queue looks like a list from far away. Up close, the hard part is not appending and popping. The hard part is deciding when a message is visible, invisible, retried, acknowledged, or dead.

The model should capture the hard part.

---

## The First Rule: Find the State

Most modeling problems become easier when you ask:

> What state exists, and who owns it?

State is the information that must survive between operations.

In a message queue, a message may need:

```go
type Message struct {
    ID          string
    Body        []byte
    State       MessageState
    Attempts    int
    VisibleAt   time.Time
    CreatedAt   time.Time
    LastError   string
}
```

This is already a model. Not because it is Go code, but because it says what the system remembers.

`VisibleAt` is especially important. It replaces vague logic with a simple rule:

> A message can be delivered only when `State == ready` and `VisibleAt <= now`.

This one field supports delayed messages, retry backoff, and visibility timeouts.

The queue itself may own:

```go
type Queue struct {
    messages MessageStore
    clock    Clock
    policy   RetryPolicy
}
```

The producer may not need much state:

```go
type Producer struct {
    queue *Queue
}
```

The consumer may not need to be a domain object at all. It may just be a process that calls `Receive`, performs work, and calls `Ack`.

This is a subtle but important point:

> Not every noun in the real world deserves to become a central object in the model.

`Producer` and `Consumer` are often roles at the boundary of the queue. The core model is usually `Message`, `Queue`, `Delivery`, `Ack`, and `RetryPolicy`.

Good modeling is not adding more objects. It is finding the objects that carry the rules.

---

## Operations Are Controlled State Changes

Once you know the state, ask:

> What operations are allowed to change it?

For a basic queue:

```go
type Queue interface {
    Publish(body []byte) (MessageID, error)
    Receive(consumerID string, now time.Time) (Delivery, error)
    Ack(deliveryID string) error
    Fail(deliveryID string, reason error) error
}
```

These operations are not random methods. Each one represents a state transition.

`Publish` creates a message:

```
nothing -> ready
```

`Receive` creates a delivery and makes the message unavailable:

```
ready -> in_flight
```

`Ack` completes the message:

```
in_flight -> acknowledged
```

`Fail` either schedules a retry:

```
in_flight -> ready
```

or gives up:

```
in_flight -> dead
```

This is where the program starts to almost write itself. The implementation may use SQL rows, Redis sorted sets, Kafka partitions, or an in-memory heap. The model is independent of that choice.

The model says what must happen. The tool decides how it happens efficiently.

---

## Invariants: The Things That Must Never Become False

The most useful word in modeling is `invariant`.

An invariant is a rule that must always be true.

For a message queue:

- A message can be acknowledged only if it is currently in flight.
- A delivery belongs to exactly one message.
- A message should not have two active deliveries at the same time.
- A dead message should not be delivered automatically.
- A message with `VisibleAt` in the future should not be delivered.
- Attempts should increase when a delivery fails.

These rules are the soul of the system.

You can write code without knowing the invariants, but then the rules are scattered across conditionals, database queries, retries, and worker behavior. When bugs appear, nobody knows which rule was violated because nobody wrote the rule down.

A good model makes invariants obvious.

For example:

```sql
SELECT *
FROM messages
WHERE state = 'ready'
  AND visible_at <= now()
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

This query is not just a query. It is enforcing part of the model:

- Only ready messages can be delivered.
- Future messages are hidden.
- Locked rows cannot be taken by another consumer.

The database tool is serving the invariant.

That is the relationship you want between model and implementation.

---

## Failure Cases Reveal the Real Model

Happy paths are too polite. They hide the real shape of the system.

Failures force precision.

Consider this queue scenario:

```
1. Consumer receives message M.
2. Consumer charges a credit card.
3. Consumer crashes before sending Ack.
4. Queue makes M visible again.
5. Another consumer receives M.
6. The credit card may be charged again.
```

The queue did nothing wrong. It promised at-least-once delivery, and it kept that promise. The bug is in the consumer's operation. Charging the card was not idempotent.

Now the model expands:

```
Message Queue
  - guarantees at-least-once delivery
  - may deliver duplicates

Consumer
  - must process messages idempotently
  - must persist result before acking
```

This is not an implementation detail. It is part of the contract between the queue and its users.

Many systems become understandable once you write down their contracts in failure cases:

- A cache may return stale data.
- A queue may deliver a message more than once.
- A payment API may receive the same request multiple times.
- A database transaction may abort and need retry.
- A lock may expire while the holder is still doing work.
- A scheduler may run the same job twice.

The model must say who is responsible for surviving those facts.

---

## A Reusable Modeling Process

Here is a practical process you can use for almost any system.

It is not perfect, but it is reliable enough to get you unstuck.

### 1. Write the stories

Write three to seven concrete stories.

For a queue:

```
Producer publishes message.
Consumer receives message.
Consumer acknowledges message.
Consumer crashes before ack.
Message fails many times.
Queue restarts.
Two consumers poll at the same time.
```

For a booking system:

```
User searches for available seats.
User holds a seat.
User pays for the seat.
Hold expires before payment.
Two users try to hold the same seat.
Payment succeeds but confirmation email fails.
```

Stories prevent abstract design. They force the model to serve behavior.

### 2. Find the durable facts

Ask what the system must remember.

For a booking system:

- A seat exists on a flight.
- A seat may be available, held, or booked.
- A hold belongs to a user.
- A hold expires at a specific time.
- A booking is created only after payment succeeds.

These are durable facts. They probably belong in the database.

Temporary details, like which HTTP handler called which service method, are not the model. They are plumbing.

### 3. Identify the states

Most important things in software move through states.

A booking seat:

```
available -> held -> booked
available -> held -> available
```

A payment:

```
created -> authorized -> captured
created -> failed
authorized -> refunded
```

A cache entry:

```
missing -> fresh -> stale -> refreshed
```

A deployment:

```
pending -> running -> healthy
pending -> running -> failed
healthy -> rolling_back -> healthy
```

If you cannot name the states, you probably do not understand the behavior yet.

### 4. Define the transitions

States alone are not enough. You need to know what can move the system from one state to another.

For a seat:

```
HoldSeat:     available -> held
ExpireHold:   held -> available
ConfirmSeat:  held -> booked
CancelHold:   held -> available
```

This immediately gives you useful rules:

- You cannot book an available seat directly.
- You cannot hold a booked seat.
- You cannot confirm an expired hold.
- Expiration is a transition, not just a background cleanup detail.

When transitions are explicit, invalid operations become visible.

### 5. Decide ownership

Ask:

> Who is allowed to change this state?

If everybody can change everything, the model will collapse.

In a booking system, only the booking service should change seat state. The payment service should not mark seats as booked directly. It should report payment results. The booking service decides what those results mean for seats.

In a queue, consumers should not delete messages directly. They should acknowledge deliveries. The queue decides whether the acknowledgement is valid.

Ownership gives you boundaries.

### 6. Write the invariants

Write the rules that must always be true.

For a booking system:

- A seat cannot have two active holds.
- A booked seat cannot become held again.
- A hold cannot be confirmed after it expires.
- A booking must reference a successful payment.
- A payment should not be captured twice for the same booking.

For a rate limiter:

- A request is allowed only if the key has remaining capacity.
- Tokens cannot exceed the bucket capacity.
- Refill is based on elapsed time, not number of requests.
- Every allowed request consumes exactly one token.

Invariants are design tests. If your model cannot enforce them, the model is incomplete.

### 7. Add failure cases

Ask what happens when each operation fails halfway.

For a queue:

- Publish succeeds but response to producer times out.
- Consumer processes message but crashes before ack.
- Ack succeeds but response to consumer is lost.
- Retry delay is scheduled but process restarts.

For a payment flow:

- Client retries after timeout.
- Payment provider charges card but callback is delayed.
- Database commit succeeds but event publish fails.
- Refund succeeds but local compensation update fails.

Failures often introduce the missing concepts: idempotency key, delivery, outbox event, reconciliation job, lease, fencing token, dead-letter queue.

### 8. Remove fake objects

After you model the behavior, remove objects that do not carry state, rules, or useful boundaries.

Maybe `Producer` is just an API client.
Maybe `Consumer` is just a worker loop.
Maybe `Manager` means nothing and should disappear.
Maybe `Service` is only a place where unrelated operations were dumped.

A smaller precise model is better than a large theatrical one.

---

## Example: Modeling a Rate Limiter

A rate limiter sounds simple:

"Allow 100 requests per minute."

The naive model is a counter:

```txt
key -> count
```

For each request, increment the count. If it is above 100, reject.

This is a model, but an incomplete one. It does not say when the count resets, what happens at the boundary between minutes, or how to handle bursts.

Let's write stories:

```
User makes a request.
System allows it if under limit.
User makes too many requests.
System rejects later requests.
Time passes.
User is allowed again.
```

Now we can choose a model.

One model is fixed window:

```txt
key + window_start -> count
```

If the current minute is `10:31`, all requests increment the counter for `10:31`. At `10:32`, the user gets a new counter.

This is simple, but it allows boundary bursts:

```
10:31:59 -> 100 requests allowed
10:32:00 -> 100 more requests allowed
```

The user effectively made 200 requests in two seconds.

Another model is token bucket:

```txt
Bucket:
  key
  tokens
  capacity
  refill_rate
  last_refilled_at
```

The rules:

- Every allowed request consumes one token.
- Tokens refill over time.
- Tokens never exceed capacity.
- If there are no tokens, reject.

This model naturally allows controlled bursts. A user can spend saved tokens quickly, but cannot exceed the long-term refill rate.

The operations are:

```txt
Refill(bucket, now)
AllowRequest(bucket, now)
```

The state transition for a request:

```
tokens > 0  -> tokens - 1, allowed
tokens == 0 -> unchanged, rejected
```

The invariant:

```
0 <= tokens <= capacity
```

Notice how the tool comes after the model.

You can implement this in Redis with Lua, in Postgres with row locks, or in memory with a mutex. The important part is that the token bucket model explains the behavior you want.

---

## Example: Modeling a Booking System

Booking systems are good modeling exercises because the wrong model fails in obvious ways.

Suppose two users try to book the same seat.

The naive model:

```txt
Seat:
  id
  is_booked
```

The operation:

```txt
if not seat.is_booked:
    charge_payment()
    seat.is_booked = true
```

This looks reasonable until two users do it at the same time:

```
User A reads seat.is_booked = false
User B reads seat.is_booked = false
User A charges payment
User B charges payment
User A marks booked
User B marks booked
```

Now two users paid for one seat.

The problem is not that we used the wrong framework. The model is too poor. A seat being booked is not the only important state. There is a temporary period where the seat is held while payment is being completed.

A better model:

```txt
Seat:
  id
  state: available | held | booked

Hold:
  id
  seat_id
  user_id
  expires_at

Booking:
  id
  seat_id
  user_id
  payment_id
```

State transitions:

```
available -> held
held -> booked
held -> available
```

Operations:

```txt
HoldSeat(user, seat)
ConfirmBooking(hold, payment)
ExpireHold(hold)
CancelHold(hold)
```

Invariants:

- A seat has at most one active hold.
- A booked seat has no active hold.
- A hold cannot be confirmed after expiration.
- A booking requires a successful payment.

Now the implementation has a target.

In SQL, you might enforce it with a transaction and row-level lock:

```sql
BEGIN;

SELECT *
FROM seats
WHERE id = $1
FOR UPDATE;

-- Only create a hold if the seat is available.
-- The lock prevents two transactions from holding the same seat.

COMMIT;
```

Or with a unique partial index:

```sql
CREATE UNIQUE INDEX one_active_hold_per_seat
ON holds (seat_id)
WHERE status = 'active';
```

Again, the database feature is not the model. The database feature enforces the model.

---

## Example: Modeling a Cache

A cache is often described as:

"Store data so reads are faster."

That description is too vague to implement correctly.

Start with stories:

```
Request asks for user profile.
Cache has fresh profile.
Return cached profile.

Request asks for user profile.
Cache has no profile.
Load from database.
Store in cache.
Return profile.

Request asks for user profile.
Cache has stale profile.
Decide whether to return stale data or wait for refresh.
```

Now we find the state:

```txt
CacheEntry:
  key
  value
  fetched_at
  expires_at
  refreshing
```

States:

```
missing -> fresh
fresh -> stale
stale -> fresh
```

The important design question is not "Redis or in-memory?" It is:

> What are we allowed to return when the cache entry is stale?

There are several valid models.

Strict cache-aside:

```txt
If missing or expired, block and load fresh value.
```

Stale-while-revalidate:

```txt
If stale, return stale value immediately and refresh in background.
```

Write-through:

```txt
Every write updates the database and cache together.
```

Write-behind:

```txt
Writes go to cache first and are persisted later.
```

Each model has different invariants and failure cases.

For strict cache-aside:

- Reads may be slower when cache misses.
- Returned data is fresher.
- Database can receive a burst if many keys expire together.

For stale-while-revalidate:

- Reads stay fast.
- Some callers may see stale data.
- Only one refresh should run per key.

That last rule introduces another concept: a per-key refresh lock.

The model grows because the behavior demands it.

---

## Example: Modeling Idempotency

Idempotency is one of the clearest examples of modeling something invisible.

The user clicks "Pay".

The browser sends:

```txt
POST /payments
```

The server charges the card.

Then the network connection drops before the response reaches the browser.

From the browser's point of view, the request failed. From the server's point of view, the payment may have succeeded. If the browser retries blindly, the user may be charged twice.

The missing model is a logical operation.

Two HTTP requests may represent the same logical payment attempt.

So we introduce an idempotency key:

```txt
PaymentAttempt:
  idempotency_key
  user_id
  amount
  status
  response
```

The invariant:

> One idempotency key can produce at most one payment side effect.

The operation:

```txt
CreatePayment(idempotency_key, user_id, amount)
```

The behavior:

```txt
If key does not exist:
  create attempt
  charge card
  store response
  return response

If key already exists:
  return stored response
```

The critical implementation rule:

> Store the idempotency record and the result atomically with the operation it protects.

Without the model, idempotency looks like a header or a database table. With the model, it becomes clear: we are representing the identity of a logical operation across retries.

---

## Names Should Explain Responsibility

Names are not decoration. Names are part of the model.

Weak names hide responsibility:

```txt
Data
Info
Manager
Processor
Handler
Service
```

These names are sometimes unavoidable at boundaries, but inside the model they are often signs of unclear thinking.

Better names describe domain responsibilities:

```txt
Delivery
Lease
Hold
Booking
RetryPolicy
IdempotencyKey
OutboxEvent
LedgerEntry
RefreshLock
```

Consider `Delivery` again.

Without it:

```txt
Consumer gets message.
```

With it:

```txt
Queue creates a delivery for a message.
Consumer acknowledges the delivery.
Delivery expires if not acknowledged.
```

The second version is longer, but it is much easier to implement correctly.

Good names make illegal states harder to express.

---

## Boundaries: What Is Inside the Model?

A system usually has a core and a boundary.

The core owns the rules. The boundary talks to the outside world.

For a message queue:

```txt
Core:
  Message
  Delivery
  RetryPolicy
  Queue operations

Boundary:
  HTTP API
  CLI
  worker process
  metrics endpoint
  storage adapter
```

For a booking system:

```txt
Core:
  Seat
  Hold
  Booking
  state transitions

Boundary:
  payment provider
  email service
  web controller
  database repository
```

This separation matters because boundaries are noisy. HTTP has status codes, headers, authentication, timeouts. Databases have indexes, transactions, query plans. Message brokers have partitions, offsets, acknowledgements.

All of that matters, but if it enters the model too early, you lose the simple shape of the problem.

First ask:

> What is true in the domain?

Then ask:

> How do I enforce this truth with the tools I have?

---

## Do Not Start With Patterns

Design patterns are names for recurring shapes. They are useful after you understand the problem. They are dangerous before that.

If you start by saying "I need a strategy pattern here" or "this should be event-driven" or "we need CQRS", you may force the problem into a shape it does not need.

Start with the model:

- What state exists?
- What changes it?
- What must never happen?
- What happens when operations fail?
- Who owns each decision?

Then patterns may appear naturally.

For example:

- A queue retry policy may become a strategy.
- A booking state transition may become a state machine.
- A payment side effect may need an outbox.
- A cache refresh may use singleflight.
- A distributed write may become a saga.

Now the pattern has a job. It is not architecture theater.

---

## The Model Can Be Smaller Than the Real World

A model should leave things out.

If you are building a queue, you do not model every machine, disk sector, TCP packet, and kernel buffer. You model messages, deliveries, acknowledgements, retries, and visibility.

If you are building a booking system, you do not model every detail of an airport. You model seats, holds, bookings, payments, and expiration.

If you are building a rate limiter, you do not model the entire user. You model a key, a capacity, a refill rate, and tokens.

The art is not modeling everything. The art is choosing what matters for the guarantees you need.

The wrong simplification causes bugs.

For a booking system, reducing the world to `seat.is_booked` is too simple. It cannot represent the important temporary state between selection and payment.

For a rate limiter, reducing the world to `request_count` is too simple. It cannot represent refill behavior or bursts.

For a queue, reducing the world to `list.pop()` is too simple. It cannot represent crashes before acknowledgement.

A good model is as small as possible, but not smaller than the failure cases.

---

## How Experienced Engineers Think

Experienced engineers are not necessarily smarter at the keyboard. They have seen more shapes.

When they hear "queue", they do not only imagine a list. They remember:

- visibility timeout
- acknowledgement
- retry
- dead-letter queue
- duplicate delivery
- idempotent consumer
- backpressure
- ordering

When they hear "booking", they remember:

- hold
- expiration
- uniqueness
- payment confirmation
- cancellation
- compensation

When they hear "cache", they remember:

- freshness
- invalidation
- stale reads
- thundering herd
- eviction
- refresh

This is why reading mature systems helps. You are not memorizing trivia. You are collecting models.

Read how real tools describe themselves:

- SQS talks about visibility timeouts and dead-letter queues.
- Kafka talks about logs, partitions, offsets, and consumer groups.
- Redis Streams talks about consumer groups and pending entries.
- PostgreSQL talks about transactions, locks, isolation, and constraints.
- Kubernetes talks about desired state, controllers, reconciliation, and resources.

Each mature system teaches you a vocabulary for a class of problems.

The goal is not to copy their architecture blindly. The goal is to notice the concepts they needed in order to make the system reliable.

---

## A Complete Modeling Walkthrough: Message Queue

Let's put the process together.

We want a durable queue with multiple consumers. We want messages not to be lost if a consumer crashes. We accept that messages may be delivered more than once. We want failed messages retried a few times and then moved aside.

### Stories

```
Producer publishes a message.
Consumer receives a message.
Consumer processes and acknowledges it.
Consumer crashes before acknowledgement.
Message becomes visible again after timeout.
Message fails repeatedly.
Message goes to the dead-letter queue.
Two consumers poll at the same time.
```

### Core concepts

```txt
Message:
  durable unit of work

Delivery:
  temporary assignment of a message to a consumer

Queue:
  owner of message visibility and state transitions

RetryPolicy:
  decides whether a failed message should be retried or killed

DeadLetterQueue:
  place for messages that should not be retried automatically
```

### State

```txt
Message:
  id
  body
  state
  attempts
  visible_at
  created_at

Delivery:
  id
  message_id
  consumer_id
  deadline
  acknowledged_at
```

### State machine

```txt
Publish:
  none -> ready

Receive:
  ready -> in_flight

Ack:
  in_flight -> acknowledged

Timeout:
  in_flight -> ready

Fail under max attempts:
  in_flight -> ready

Fail at max attempts:
  in_flight -> dead
```

### Invariants

```txt
A ready message has no active delivery.
An in-flight message has one active delivery.
An acknowledged message is never delivered again.
A dead message is never delivered by normal receive.
A delivery can be acknowledged only before or at its deadline.
Receiving a message increments or records an attempt.
```

### API

```go
type Queue interface {
    Publish(ctx context.Context, body []byte) (MessageID, error)
    Receive(ctx context.Context, consumerID string) (Delivery, error)
    Ack(ctx context.Context, deliveryID string) error
    Fail(ctx context.Context, deliveryID string, reason error) error
}
```

### Storage rules

If implemented in SQL:

- `Receive` must select a ready visible message inside a transaction.
- The selected row must be locked so another consumer cannot take it.
- The message state and delivery record must be updated atomically.
- `Ack` must verify that the delivery is still active.
- A background process must expire old deliveries.

If implemented in Redis:

- Ready messages may live in a list or sorted set.
- In-flight deliveries need a separate structure.
- Visibility timeout requires timestamps.
- Expiration requires a scanner or delayed queue.

If implemented in memory:

- Shared state needs a mutex or channel ownership.
- Restart loses messages unless persisted elsewhere.
- This may be fine for local background work but not for durable jobs.

The same model can have many implementations.

That is the payoff.

---

## How to Practice

You get better at modeling by practicing models directly, not only by writing code.

Take a system and write one page:

```txt
System:

Stories:

Core concepts:

State:

Transitions:

Invariants:

Failure cases:

Implementation choices:
```

Do this for:

- URL shortener
- rate limiter
- cache
- job queue
- notification system
- file upload service
- booking system
- payment workflow
- cron scheduler
- feature flag system
- collaborative document editor

At first it will feel slow. That is normal. You are training the part of your brain that turns vague requirements into explicit rules.

Over time, you will start recognizing familiar shapes:

- "This needs a lease."
- "This operation must be idempotent."
- "This is a state machine."
- "This needs an outbox."
- "This needs a uniqueness constraint."
- "This can return stale data."
- "This needs reconciliation."

That recognition is what people call experience.

---

## A Checklist for Getting Unstuck

When you cannot model a system, do not stare at the code.

Ask these questions:

```txt
1. What are the main stories?
2. What must the system remember?
3. What are the states of the important things?
4. What operations move things between states?
5. What rules must always be true?
6. Who owns each piece of state?
7. What happens if each operation fails halfway?
8. Can the same request happen twice?
9. Can two actors do this at the same time?
10. What can be stale, duplicated, lost, delayed, or retried?
11. Which parts are core rules and which parts are just I/O?
12. What tool can enforce the rules I need?
```

The order matters. Tool choice comes near the end.

If you choose the tool before understanding the rules, you will design around whatever the tool makes easy. Sometimes that is fine. Sometimes it gives you a system that works in demos and fails in production.

---

## Conclusion

Modeling is not a mystical talent. It is a habit of making hidden rules explicit.

You start with stories. You find the state. You name the transitions. You write the invariants. You assign ownership. You attack the model with failure cases. Then, and only then, you choose the tool that can enforce the behavior.

The message queue is not just a queue. It is messages, deliveries, acknowledgements, visibility timeouts, retries, and dead letters.

The booking system is not just a boolean called `is_booked`. It is seats, holds, expirations, payments, confirmations, and uniqueness.

The cache is not just a map. It is freshness, staleness, refresh, invalidation, and the decision of what stale data is allowed to mean.

This is the move from "I can write code" to "I can design software."

The more systems you model, the more shapes you carry around. Eventually, when someone says "build a queue" or "build a scheduler" or "build a payment flow", you will not see a blank page. You will see stories, state, transitions, invariants, and failure cases.

That is how people think about these things.

