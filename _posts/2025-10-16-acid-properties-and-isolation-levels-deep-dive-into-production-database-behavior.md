---
title: "ACID Properties and Isolation Levels: Deep Dive into Production Database Behavior"
date: 2025-10-16
categories: ["Database Internals"]
tags: [acid, isolation-levels, transactions, mvcc, postgresql]
---
## Why This Matters in Production

Every backend engineer has a story about a production incident that made ACID properties real. Maybe it was a race condition in a payment system that took hours to trace. Maybe it was discovering that your carefully designed transaction logic didn't actually prevent duplicate orders. Maybe it was realizing that "eventually consistent" meant losing money for a few hours.

The problem isn't that ACID is hard to understand theoretically. The problem is knowing which guarantees you actually have in your specific database at your specific isolation level, and what happens when you make the wrong assumption.

## Understanding Transactions Beyond the Acronym

Before we talk about ACID, let's be clear about what a transaction actually is in production systems. It's not just a code block between BEGIN and COMMIT. It's a contract with your database: execute these operations atomically while this specific set of other transactions are also running.

The key phrase is "while other transactions are running." In a single-threaded scenario with zero concurrency, ACID is trivial. Everything works. The complexity emerges when you have hundreds of transactions fighting over the same data.

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;
```

This looks simple. But what if another transaction is reading account_id = 1 right now? What if it's updating both accounts too? What if the server crashes after the first UPDATE? These are the scenarios that matter.

## A: Atomicity - All or Nothing, But Not Instant

Atomicity is often described as "all or nothing," which is correct but incomplete. The real guarantee is: if anything goes wrong, we roll back everything. But the roll back isn't instant, and understanding the mechanics matters.

When you execute statements in a transaction, they don't go directly to disk. They go into a buffer. The database keeps a transaction log (WAL file in PostgreSQL, binlog in MySQL). If the transaction completes successfully, the database writes a COMMIT marker to the log. If something fails before that marker is written, the transaction never existed as far as the database is concerned.

**Real scenario: What happens during a server crash**

You're doing a transfer:

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;
```

The server crashes after the first UPDATE but before COMMIT. What happens?

With atomicity: When the database restarts, it reads the transaction log. It sees the first UPDATE was written to the log but there's no COMMIT marker. So it rolls back that UPDATE. The database recovers to the state before the transaction started. Account 1 still has its money.

Without atomicity: Account 1 loses 100 but account 2 never receives it. This is a disaster.

This is why you never shortcut ACID guarantees by running bare SQL outside of transaction blocks. And it's why the transaction log is critical infrastructure.

**The catch:** Atomicity doesn't mean fast. A rollback still takes time. If you have millions of changes in a transaction, the rollback operation can take minutes. This is why keeping transactions small and quick is a best practice.

## C: Consistency - Your Contract With Your Data

Consistency is where most misunderstanding happens. It's not about the database enforcing your rules. It's about you maintaining invariants that the database helps enforce.

If you have a constraint that "email must be unique," that's database-enforced consistency. If you have a rule that "total money in the system never changes, only moves between accounts," that's application-enforced consistency. If you have a rule that "order total must equal sum of line items," that's application-enforced consistency.

```sql
-- This maintains consistency
BEGIN TRANSACTION;
  INSERT INTO orders (user_id, total) VALUES (1, 100);
  INSERT INTO line_items (order_id, amount) VALUES (1, 50);
  INSERT INTO line_items (order_id, amount) VALUES (1, 50);
COMMIT;
-- total equals sum of line items: 100 = 50 + 50

-- This violates consistency (but the database won't stop you)
BEGIN TRANSACTION;
  INSERT INTO orders (user_id, total) VALUES (1, 100);
  INSERT INTO line_items (order_id, amount) VALUES (1, 50);
  INSERT INTO line_items (order_id, amount) VALUES (1, 30);
COMMIT;
-- total doesn't equal sum of line items: 100 != 50 + 30
-- Database is fine with this. You broke it.
```

The database prevents obvious consistency violations (foreign key constraint, NOT NULL constraint, unique constraint). Everything else depends on you. This is why bad data happens in production: someone wrote code that violated unstated consistency assumptions.

**In production:** Consistency failures are often invisible until they compound. You might have thousands of orders with wrong totals before anyone notices. By then, figuring out what's correct is nightmare fuel.

The lesson: every complex transaction should have explicit checks that you're maintaining your invariants. Not just on success, but also validate before you commit.

## I: Isolation - The Dangerous Middle Ground

Isolation is where most production bugs live. It's also where isolation levels come in, and this is where decisions matter.

The core concept: your transaction shouldn't see the incomplete work of other transactions, and other transactions shouldn't see yours. But what does "shouldn't" mean? And what does it cost to guarantee it?

Different isolation levels answer this question differently. Each answer is a trade-off between safety and performance. And in production, you usually find out which you chose by watching things break.

### What Problems Can Happen Without Isolation

**Dirty Read:** You read uncommitted data from another transaction. This is the easiest problem to understand.

```sql
-- Account balance is 1000

-- Transaction A
BEGIN TRANSACTION;
  UPDATE accounts SET balance = 500 WHERE account_id = 1;
  -- CRASH! Or just pause here

-- Transaction B (running at the same time)
SELECT balance FROM accounts WHERE account_id = 1;
-- Sees 500, even though Transaction A never committed
-- What if A rolls back? Now B has data that never existed
-- This is a dirty read
```

Dirty reads are bad because they break the entire concept of transactions. If you see uncommitted data and make decisions based on it, you're working with potentially false information.

**Non-Repeatable Read (Fuzzy Read):** You read the same row twice in one transaction and get different values because another transaction changed it and committed between your reads.

```sql
-- Account balance is 1000

-- Transaction A
BEGIN TRANSACTION;
  SELECT balance FROM accounts WHERE account_id = 1;  -- sees 1000

  -- Meanwhile, Transaction B happens and completes
  BEGIN TRANSACTION;
    UPDATE accounts SET balance = 500 WHERE account_id = 1;
  COMMIT;

  -- Back to Transaction A
  SELECT balance FROM accounts WHERE account_id = 1;  -- sees 500
  -- Same row, same transaction, different value
COMMIT;
```

This is nasty in production because your code might read a value, make a decision, read it again, and discover your decision was based on stale data.

```sql
-- Real scenario: approval workflow
BEGIN TRANSACTION;
  SELECT approval_budget FROM users WHERE user_id = 123;  -- sees 5000
  -- Check: can user approve this 4000 expense? Yes, they have budget.

  -- But someone just increased their budget
  UPDATE users SET approval_budget = 10000 WHERE user_id = 123;
  -- They committed

  -- Our code continues
  SELECT approval_budget FROM users WHERE user_id = 123;  -- sees 10000
  -- Non-repeatable read, but at least consistent now
  -- Problem: we made our approval decision based on 5000 budget
COMMIT;
```

**Phantom Read:** You run a query that matches N rows, then run the same query later and match different number of rows because another transaction inserted or deleted rows between your queries.

```sql
-- Table has 3 accounts with balance > 1000

-- Transaction A
BEGIN TRANSACTION;
  SELECT COUNT(*) FROM accounts WHERE balance > 1000;  -- returns 3

  -- Meanwhile, Transaction B inserts
  BEGIN TRANSACTION;
    INSERT INTO accounts (balance) VALUES (5000);
  COMMIT;

  -- Back to Transaction A
  SELECT COUNT(*) FROM accounts WHERE balance > 1000;  -- returns 4
  -- Same query, different result. Phantom read.
COMMIT;
```

Phantom reads are insidious. Your WHERE clause captures different rows at different times. If you're doing calculations or aggregations, you get inconsistent results.

**Lost Update:** Two transactions both read the same value, modify it independently, and write it back. One write overwrites the other.

```sql
-- Account balance is 100

-- Transaction A reads
SELECT balance FROM accounts WHERE account_id = 1;  -- sees 100

-- Transaction B reads
SELECT balance FROM accounts WHERE account_id = 1;  -- sees 100

-- Transaction A calculates and writes
UPDATE accounts SET balance = 100 + 50 WHERE account_id = 1;  -- now 150

-- Transaction B calculates and writes
UPDATE accounts SET balance = 100 + 75 WHERE account_id = 1;  -- now 175
-- But logically we added 50 and 75, total should be 225
-- The +50 from Transaction A is lost
```

This is why optimistic locking exists. If you read-modify-write, you need to check that nobody else wrote between your read and your write.

## The Isolation Levels: Trade-offs You Actually Make

Now let's talk about what the database actually gives you at each level.

### Read Uncommitted: Barely Controlled Chaos

Read Uncommitted is the lowest isolation level. The database barely locks anything. It's fast but provides almost no guarantees.

**What it prevents:**

- Nothing really. All the problems above can happen.

**What it allows:**

- Dirty reads
- Non-repeatable reads
- Phantom reads

**When you'd use it:** Almost never in production. You might use it for analytics queries where you're okay reading stale or inconsistent data, but even then, usually not.

**Real scenario where it fails:**

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- User 1: Checking balance before transfer
SELECT balance FROM checking_account WHERE user_id = 1;  -- sees 10000

-- User 2: Transferring money out (hasn't committed yet)
UPDATE checking_account SET balance = balance - 9000 WHERE user_id = 1;

-- User 1: Confirming the transfer
SELECT balance FROM checking_account WHERE user_id = 1;  -- sees 1000
-- User 1 thinks there's only 1000 left but User 2 hasn't committed yet
-- If User 2 crashes and rolls back, the money reappears
```

The problem: Your business logic can't trust what it reads. You're constantly working with potentially false data.

### Read Committed: The Default Compromise

Read Committed is the default in PostgreSQL and a common choice in production systems.

**What it prevents:**

- Dirty reads

**What it allows:**

- Non-repeatable reads
- Phantom reads

**How it works:**

You can only see data that other transactions have committed. If a transaction is modifying a row, you either wait for it to commit, or you see the last committed version.

**Real scenario:**

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Account has balance of 1000

-- Transaction A: Checking balance
BEGIN TRANSACTION;
  SELECT balance FROM accounts WHERE account_id = 1;  -- sees 1000

-- Transaction B: Transferring (in parallel)
BEGIN TRANSACTION;
  UPDATE accounts SET balance = 500 WHERE account_id = 1;
  -- Transaction B is holding a lock

-- Back to Transaction A: Trying to check balance again
SELECT balance FROM accounts WHERE account_id = 1;
-- Waits for Transaction B to release the lock
-- Once B commits (balance = 500), A sees 500
-- Or if B rolls back, A still sees 1000
-- Either way, A only sees committed data

COMMIT;
```

But non-repeatable reads can still happen:

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Transaction A
BEGIN TRANSACTION;
  SELECT balance FROM accounts WHERE account_id = 1;  -- sees 1000
  -- Does some processing

-- Transaction B (commits in between)
UPDATE accounts SET balance = 500 WHERE account_id = 1;
COMMIT;

-- Transaction A continues
SELECT balance FROM accounts WHERE account_id = 1;  -- sees 500
-- Same query in same transaction, different result
COMMIT;
```

**Why it's popular:** It prevents the most dangerous problem (dirty reads) while still allowing reasonable concurrency. Most applications can work around non-repeatable reads with proper application logic.

**The production gotcha:** If your code relies on reading a value multiple times and expecting consistency within a transaction, you'll find bugs. The classic mistake is reading a balance, checking it's sufficient, then reading it again for logging and finding it changed.

### Repeatable Read: The Safety Upgrade

Repeatable Read prevents dirty reads and non-repeatable reads. It uses snapshot isolation in PostgreSQL.

**What it prevents:**

- Dirty reads
- Non-repeatable reads

**What it allows:**

- Phantom reads (sort of, more on this later)

**How it works:**

When you BEGIN a transaction, the database takes a snapshot of the current state. Your entire transaction sees that snapshot. Other transactions' changes are invisible to you until you commit and start a new transaction.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Account has balance of 1000

-- Transaction A
BEGIN TRANSACTION;
  SELECT balance FROM accounts WHERE account_id = 1;  -- sees 1000
  -- Snapshot taken. This is what A will see for the rest of the transaction

-- Transaction B (commits in parallel)
UPDATE accounts SET balance = 500 WHERE account_id = 1;
COMMIT;

-- Back to Transaction A
SELECT balance FROM accounts WHERE account_id = 1;  -- still sees 1000
-- Same query in same transaction, same result
-- This is repeatable read in action
COMMIT;
```

**PostgreSQL's Repeatable Read is actually quite strong:**

PostgreSQL detects if your transaction would create an anomaly and aborts it. So while phantom reads are technically possible, you often get a serialization failure instead.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Transaction A
BEGIN TRANSACTION;
  SELECT COUNT(*) FROM accounts WHERE balance > 1000;  -- sees 5

-- Transaction B
INSERT INTO accounts (balance) VALUES (5000);
COMMIT;

-- Transaction A (if it tries to modify based on that count)
UPDATE accounts SET status = 'high_balance' WHERE balance > 1000;
-- PostgreSQL: Wait, the count of rows changed between A's read and write
-- ERROR: Serialization failure, retry the transaction
```

**When to use it:** When you need consistency within a transaction. Financial calculations, inventory reservations, anything where you read data multiple times and need the same answer.

```sql
-- Real scenario: Calculating available inventory
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN TRANSACTION;
  SELECT quantity FROM inventory WHERE product_id = 123;  -- sees 50
  -- Check: is 50 enough? Yes

  -- Do some processing...

  SELECT quantity FROM inventory WHERE product_id = 123;  -- still sees 50
  -- Not 45, not 55, still 50
  UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 123;
COMMIT;
```

**The production win:** You eliminate entire classes of race conditions. Code that reads then writes based on what it read is now safe.

### Serializable: When Nothing Else Works

Serializable is the highest isolation level. Your transaction executes as if it's the only transaction on the database.

**What it prevents:**

- Everything. Dirty reads, non-repeatable reads, phantom reads, lost updates.

**How it works:**

PostgreSQL uses "Serializable Snapshot Isolation." The database detects if your transaction conflicts with another and aborts one of them. You have to handle retries in your application.

Traditional databases like MySQL use locks: if you read rows, nobody else can modify them until you commit.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Transaction A
BEGIN TRANSACTION;
  SELECT COUNT(*) FROM accounts WHERE balance > 1000;  -- sees 5

-- Transaction B
INSERT INTO accounts (balance) VALUES (5000);
COMMIT;

-- Transaction A tries to write
UPDATE accounts SET status = 'high_balance' WHERE balance > 1000;
-- PostgreSQL detects conflict and aborts A
-- ERROR: Serialization failure, retry the transaction

-- You must retry from the beginning
BEGIN TRANSACTION;
  SELECT COUNT(*) FROM accounts WHERE balance > 1000;  -- now sees 6
  -- Continue with correct data
COMMIT;
```

**When to use it:** Critical operations where correctness is non-negotiable and you can afford retry logic. Payment processing, financial settlements, inventory allocation for limited resources.

**The production cost:** Performance degrades. Transactions fail and need retries. In high-contention scenarios, you might spend more time retrying than executing.

```sql
-- Example where Serializable prevents disaster

-- Airline seat booking
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- User 1
BEGIN TRANSACTION;
  SELECT available_seats FROM flights WHERE flight_id = 1;  -- sees 1
  UPDATE flights SET available_seats = 0 WHERE flight_id = 1;
  INSERT INTO bookings VALUES (user_id=1, flight_id=1);
COMMIT;  -- Success

-- User 2 (running in parallel)
BEGIN TRANSACTION;
  SELECT available_seats FROM flights WHERE flight_id = 1;  -- sees 1
  UPDATE flights SET available_seats = 0 WHERE flight_id = 1;
  INSERT INTO bookings VALUES (user_id=2, flight_id=1);
COMMIT;  -- ERROR: Serialization failure

-- User 2 retries
BEGIN TRANSACTION;
  SELECT available_seats FROM flights WHERE flight_id = 1;  -- now sees 0
  -- Can't book. Abort gracefully.
ROLLBACK;

-- Result: Only one person got the last seat, not both
```

Without Serializable, both users would have been booked on the same seat.

## Database Defaults Matter More Than You Think

Different databases make different default choices:

**PostgreSQL defaults to Read Committed**

- Philosophy: Safe enough for most workloads, good performance
- You opt-in to stronger isolation if needed

**MySQL (InnoDB) defaults to Repeatable Read**

- Philosophy: Stronger isolation by default
- Better protection out of the box, but slightly slower

**Oracle defaults to Serializable**

- Philosophy: Correctness first
- Heavy on locking, expensive in terms of resources

Most teams don't explicitly set isolation levels. They just use the default. This means your behavior depends entirely on what database you're running and whether your code happens to work with that default.

This is why it's crucial to:

1. Know your database's default
2. Know what isolation level your critical operations need
3. Explicitly set it rather than relying on defaults
4. Test concurrent scenarios

## Real Production Scenarios

Let's walk through scenarios that actually break systems:

**Scenario 1: The E-commerce Oversell**

```sql
-- Inventory has 10 units

-- Customer 1 (Repeatable Read)
BEGIN TRANSACTION;
  SELECT inventory FROM products WHERE product_id = 1;  -- sees 10
  -- Process order...
  UPDATE inventory SET inventory = inventory - 1 WHERE product_id = 1;
COMMIT;

-- Customer 2 (Repeatable Read, in parallel)
BEGIN TRANSACTION;
  SELECT inventory FROM products WHERE product_id = 1;  -- sees 10 (snapshot from before C1's update)
  UPDATE inventory SET inventory = inventory - 1 WHERE product_id = 1;
COMMIT;

-- Both see 10, both subtract 1, inventory ends at 9 instead of 8
-- You oversold by 1 unit
```

Fix: You need to either use Serializable, or use SELECT FOR UPDATE to lock the row:

```sql
BEGIN TRANSACTION;
  SELECT inventory FROM products WHERE product_id = 1 FOR UPDATE;  -- locks the row
  -- Now Customer 2 waits
  UPDATE inventory SET inventory = inventory - 1 WHERE product_id = 1;
COMMIT;
-- Customer 2 can now proceed with the correct value
```

**Scenario 2: The Money That Disappeared**

```sql
-- Account A: 1000, Account B: 1000

-- Transaction 1: Transfer 100 A -> B (Read Committed)
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;

-- Transaction 2: Audit check (Read Committed, in parallel)
BEGIN TRANSACTION;
  SELECT SUM(balance) FROM accounts;  -- sees 2000 (both before and after the transfer)
  -- Actually sees 1900 because one side executed
COMMIT;

-- The SUM momentarily doesn't equal what it should
```

Fix: Audit code needs Repeatable Read or Serializable to see a consistent snapshot.

**Scenario 3: The Duplicate Order**

```sql
-- Order entry system with Read Committed

-- User submits order (Connection 1)
BEGIN TRANSACTION;
  SELECT * FROM orders WHERE order_id = 12345;  -- sees nothing
  -- Order doesn't exist yet
  INSERT INTO orders VALUES (12345, ...) ;
COMMIT;

-- But there's a race: another connection was inserting the same order
-- Both connections see "no order exists," both insert
-- Now you have duplicate orders
```

Fix: Use unique constraints and handle the violation, or use Serializable, or use SELECT FOR UPDATE on a row that represents this logical operation.

## The Practical Strategy

Here's how to think about isolation levels in production:

1. **Start with Read Committed** unless you have a specific reason to use stronger isolation. It's fast and good enough for most workloads.

2. **Identify critical operations** that absolutely cannot tolerate inconsistency. These are your payment processing, inventory management, financial reporting. Upgrade these to Repeatable Read or Serializable.

3. **Use SELECT FOR UPDATE** for row-level locking when you need to read-check-update atomically. Don't rely on isolation level alone.

4. **Test concurrent scenarios.** Write tests that simulate multiple transactions running in parallel. Most race conditions won't show up in normal testing.

5. **Monitor for serialization failures.** If you use Serializable, monitor how often transactions abort. High rates mean high contention and you might need to rethink your design.

6. **Document your assumptions.** If your code depends on seeing consistent data across multiple reads, document that and set the isolation level explicitly.

7. **Measure the cost.** Stronger isolation is slower. Measure the actual performance impact before defaulting to Serializable everywhere.

## Wrapping Up

ACID properties and isolation levels aren't theoretical concepts. They're the difference between systems that lose money and systems that don't, between data that's correct and data that's mysteriously inconsistent. Every production incident related to data corruption or race conditions traces back to someone not fully understanding which guarantees they actually have.

The key insight: there's no one-size-fits-all answer. You choose based on your specific requirements. Understand the trade-offs, test your scenarios, and be explicit about what you're choosing.
