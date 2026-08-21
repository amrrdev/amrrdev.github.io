---
title: "Idempotency in Production: Making Your System Safe to Retry"
date: 2025-10-16
categories: ["System Design"]
tags: [idempotency, reliability, api-design, distributed-systems]
---
## The Real Cost - What happens when you don't have idempotency

Every backend engineer has experienced this: a user clicks a button, the request times out, they panic and click it again. Now something happened twice. Or the network was flaky and the request actually made it through the first time but the response got lost. The user never knew. Now you have duplicate data.

This is the idempotency problem, and it's one of the most common sources of subtle bugs in production systems.

## What Is Idempotency?

Start with the math definition. A function is idempotent if calling it multiple times with the same input produces the same result as calling it once.

```ts
f(x) = x * 0; // idempotent: 0 * 0 = 0, 0 * 0 = 0
f(x) = x + 1; // not idempotent: f(5) = 6, f(6) = 7
```

In backend systems, Idempotency means that performing the same operation multiple times has the same effect as performing it once. The system recognizes that you're asking for the same thing and doesn't do the work twice.

**Example:**

```ts
GET / user / 123; // idempotent: fetching the same user 100 times returns the same data
POST / orders; // NOT idempotent: creating the same order 100 times creates 100 orders
DELETE / user / 123; // idempotent: deleting the same user 100 times is the same as deleting once
```

HTTP verbs capture this:

- GET, PUT, DELETE are supposed to be idempotent
- POST is not idempotent by default

But here's the thing: just because HTTP says DELETE should be idempotent doesn't mean your code makes it that way. You have to build it.

## Why Idempotency Matters

Network failures happen constantly. Here are the scenarios:

**Scenario 1: Request timeout**

```ts
User clicks "Pay $100"
  |
  v
Request sent to payment service
  |
  v (30 seconds later, no response)
Request times out on client side
  |
  v
User clicks "Pay $100" again
  |
  v
Server processes both payments
  |
  v
User charged $200 for one thing
```

Your system needs to recognize that the second request is a retry of the first one.

**Scenario 2: Response lost**

```ts
User clicks "Pay $100"
  |
  v
Server receives request
Server processes payment ($100 charged)
Server sends response "Success"
  |
  v
Response is lost on the network
  |
  v
User never sees "Success," assumes it failed
User clicks "Pay $100" again
  |
  v
Server processes second payment
  |
  v
User charged $200 for one thing
```

The payment went through the first time, but the user never knew. Now they retry and it happens again.

**Scenario 3: Duplicate delivery from message queue**

```ts
Message queue delivers message: "Process order 12345"
Server processes order
Server ACKs the message
  |
  v
ACK is lost
Message queue doesn't know it was processed
  |
  v
Message queue redelivers: "Process order 12345"
Server processes order again
  |
  v
Order duplicated
```

These scenarios aren't rare. They happen constantly in production. Your payment processor might retry. Your message queue might redeliver. Your load balancer might route a retry to a different server that didn't process it the first time.

The only way to survive this is idempotency. Build systems that are safe to retry.

## The Foundation: Idempotent Keys

An idempotent key (also called a request ID, correlation ID, or deduplication key) is a unique identifier that you send with your request. The server stores this key along with the result of the operation. If the same key arrives again, the server returns the cached result instead of processing it again.

**Basic concept:**

```ts
Request 1:
  POST /payments
  {
    "amount": 100,
    "account_id": 123,
    "idempotency_key": "user-click-12345-timestamp-1698765432"
  }

Server stores: idempotency_key -> {status: "success", payment_id: 999}

Request 2 (retry, same idempotency key):
  POST /payments
  {
    "amount": 100,
    "account_id": 123,
    "idempotency_key": "user-click-12345-timestamp-1698765432"
  }

Server sees the key exists
Server returns cached result: {status: "success", payment_id: 999}
No new payment created
```

The user makes the same request, but only pays once.

**Where does the idempotency key come from?**

The client generates it. Usually some combination of:

- User ID or session ID
- Action type
- Timestamp
- Random component

```ts
POST /payments HTTP/1.1
Host: api.example.com
Idempotency-Key: user-123-transfer-1698765432-abc123xyz
Content-Type: application/json

{
  "from_account": 456,
  "to_account": 789,
  "amount": 100
}
```

Or it can be in the request body. The key point: the client sends it, and the server respects it.

**Why the client generates it:**

If the server generated it, the client would receive it in the response. But if the response gets lost, the client doesn't know what key the server used. So it can't retry with the same key. The server would see a new key and process the request again.

Only the client knows if it's retrying. So the client generates the key.

## Building Idempotency: Simple Version

Let's start with a simple payment system:

```typescript
// Without idempotency
app.post("/payments", async (req: Request, res: Response) => {
  const { amount, account_id } = req.body;

  // Process payment immediately
  await chargeAccount(account_id, amount);

  // Create payment record
  const payment = await Payment.create({ amount, account_id });

  return res.json({ payment_id: payment.id, status: "success" });
});
```

This breaks immediately with retries. Same request twice equals two charges.

**Version 1: Simple idempotency key storage**

```typescript
import express, { Request, Response } from "express";

// Simple in-memory storage (don't do this in production!)
const idempotencyCache = new Map<string, any>();

app.post("/payments", async (req: Request, res: Response) => {
  const idempotencyKey = req.headers["idempotency-key"] as string;

  if (!idempotencyKey) {
    return res.status(400).json({ error: "Idempotency-Key header required" });
  }

  // Check if we've seen this key before
  if (idempotencyCache.has(idempotencyKey)) {
    const cachedResult = idempotencyCache.get(idempotencyKey);
    return res.status(200).json(cachedResult);
  }

  const { amount, account_id } = req.body;

  // New request, process it
  await chargeAccount(account_id, amount);
  const payment = await Payment.create({ amount, account_id });

  const result = {
    payment_id: payment.id,
    status: "success",
  };

  // Store the result
  idempotencyCache.set(idempotencyKey, result);

  return res.status(200).json(result);
});
```

This works but has problems:

- In-memory cache is lost if the server restarts
- Multiple servers would have separate caches and could process the same key twice
- No way to clean up old keys

## Production Version: Database-Backed Idempotency

Real systems store idempotency keys in the database.

**Schema:**

```sql
CREATE TABLE idempotency_keys (
    id BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(255) NOT NULL UNIQUE,
    request_method VARCHAR(10) NOT NULL,
    request_path VARCHAR(255) NOT NULL,
    response_status INTEGER,
    response_body JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_idempotency_keys_key ON idempotency_keys(idempotency_key);
```

Why include request_method and request_path? Because the same idempotency key might be used for different operations. It's safer to only deduplicate if it's truly the same request.

**Implementation:**

```typescript
import express, { Request, Response } from "express";
import { db } from "./db"; // Your database connection

interface IdempotencyKey {
  id: number;
  idempotency_key: string;
  request_method: string;
  request_path: string;
  response_status: number | null;
  response_body: Record<string, unknown>;
  created_at: Date;
  expires_at: Date;
}

app.post("/payments", async (req: Request, res: Response) => {
  const idempotencyKey = req.headers["idempotency-key"] as string;

  if (!idempotencyKey) {
    return res.status(400).json({ error: "Idempotency-Key header required" });
  }

  // Check if we've seen this key before
  const existing = await db.query<IdempotencyKey>(
    `SELECT * FROM idempotency_keys 
     WHERE idempotency_key = $1 
     AND request_method = $2 
     AND request_path = $3`,
    [idempotencyKey, "POST", "/payments"]
  );

  if (existing.rows.length > 0) {
    const row = existing.rows[0];
    return res.status(row.response_status).json(row.response_body);
  }

  // New request, process it inside a transaction
  const client = await db.connect();
  try {
    await client.query("BEGIN");

    const { amount, account_id } = req.body;

    // Charge the account
    await chargeAccount(account_id, amount);

    // Create payment record
    const paymentResult = await client.query(
      `INSERT INTO payments (account_id, amount, status) 
       VALUES ($1, $2, 'completed') 
       RETURNING id`,
      [account_id, amount]
    );
    const paymentId = paymentResult.rows[0].id;

    const response = {
      payment_id: paymentId,
      status: "success",
    };

    // Store the idempotency key and response
    await client.query(
      `INSERT INTO idempotency_keys 
       (idempotency_key, request_method, request_path, response_status, response_body, expires_at)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [
        idempotencyKey,
        "POST",
        "/payments",
        200,
        JSON.stringify(response),
        new Date(Date.now() + 24 * 60 * 60 * 1000),
      ]
    );

    await client.query("COMMIT");
    return res.status(200).json(response);
  } catch (error) {
    await client.query("ROLLBACK");

    const errorResponse = {
      error: error instanceof Error ? error.message : "Unknown error",
    };

    // Store the error as the result
    await db.query(
      `INSERT INTO idempotency_keys 
       (idempotency_key, request_method, request_path, response_status, response_body, expires_at)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [
        idempotencyKey,
        "POST",
        "/payments",
        400,
        JSON.stringify(errorResponse),
        new Date(Date.now() + 24 * 60 * 60 * 1000),
      ]
    );

    return res.status(400).json(errorResponse);
  } finally {
    client.release();
  }
});
```

Now if the same key arrives:

- Request 1: Charges account, creates payment, stores result
- Request 2 (retry): Sees the key exists, returns the cached result without charging again

## The Critical Detail: Atomicity

Here's where most implementations fail. Look at this code:

```typescript
// WRONG - Race condition
const existing = await db.query("SELECT * FROM idempotency_keys WHERE idempotency_key = $1", [key]);

if (existing.rows.length > 0) {
  return existing.rows[0].response_body;
}

// Two requests with same key both get here
// Both see "not existing" and both process the payment

await chargeAccount(account_id, amount);
const payment = await createPayment();
await db.query("INSERT INTO idempotency_keys ...");
```

The check and the insert are separate operations. If two requests arrive simultaneously with the same key:

1. Request A checks: key doesn't exist
2. Request B checks: key doesn't exist (A hasn't inserted it yet)
3. Request A charges and inserts
4. Request B charges and inserts
5. Both payments went through

You need atomicity. The database must prevent this race condition.

**Solution: Use database constraints and insert first**

```typescript
async function createPaymentWithIdempotency(
  amount: number,
  accountId: number,
  idempotencyKey: string
) {
  const client = await db.connect();

  try {
    // Try to insert the idempotency key first as a lock
    // This acts as an atomic operation
    const insertResult = await client.query(
      `INSERT INTO idempotency_keys 
       (idempotency_key, request_method, request_path, response_status, response_body, expires_at)
       VALUES ($1, $2, $3, $4, $5, $6)
       ON CONFLICT DO NOTHING
       RETURNING *`,
      [
        idempotencyKey,
        "POST",
        "/payments",
        null,
        JSON.stringify({}),
        new Date(Date.now() + 24 * 60 * 60 * 1000),
      ]
    );

    if (insertResult.rows.length === 0) {
      // Key already exists, we lost the race
      const existing = await client.query(
        `SELECT * FROM idempotency_keys WHERE idempotency_key = $1`,
        [idempotencyKey]
      );

      const row = existing.rows[0];

      if (row.response_status === null) {
        // Still processing
        return { status: 202, body: { status: "processing" } };
      }

      // Response is ready
      return { status: row.response_status, body: row.response_body };
    }

    // We inserted the key, we own this request
    await client.query("BEGIN");

    try {
      await chargeAccount(accountId, amount);
      const paymentResult = await client.query(
        `INSERT INTO payments (account_id, amount, status) 
         VALUES ($1, $2, 'completed') 
         RETURNING id`,
        [accountId, amount]
      );

      const response = {
        payment_id: paymentResult.rows[0].id,
        status: "success",
      };

      // Update with the response
      await client.query(
        `UPDATE idempotency_keys 
         SET response_status = $1, response_body = $2
         WHERE idempotency_key = $3`,
        [200, JSON.stringify(response), idempotencyKey]
      );

      await client.query("COMMIT");
      return { status: 200, body: response };
    } catch (error) {
      await client.query("ROLLBACK");

      const response = {
        error: error instanceof Error ? error.message : "Unknown error",
      };

      await client.query(
        `UPDATE idempotency_keys 
         SET response_status = $1, response_body = $2
         WHERE idempotency_key = $3`,
        [400, JSON.stringify(response), idempotencyKey]
      );

      return { status: 400, body: response };
    }
  } finally {
    client.release();
  }
}
```

Now:

1. Request A inserts the key first (succeeds)
2. Request B tries to insert the same key (fails with unique constraint)
3. Request B sees the key exists and waits or returns processing status
4. Request A processes the payment
5. Request B retries and gets the result

Only one request actually processes the payment.

## Handling Long-Running Operations

What if the operation takes 30 seconds? The retry mechanism needs to handle this.

**Pattern 1: Wait for the result**

```typescript
const existing = await db.query<IdempotencyKey>(
  "SELECT * FROM idempotency_keys WHERE idempotency_key = $1",
  [idempotencyKey]
);

if (existing.rows.length > 0 && existing.rows[0].response_status === null) {
  // Still processing, wait
  for (let attempt = 0; attempt < 30; attempt++) {
    // Wait up to 30 seconds
    await new Promise((resolve) => setTimeout(resolve, 1000));

    const updated = await db.query<IdempotencyKey>(
      "SELECT * FROM idempotency_keys WHERE idempotency_key = $1",
      [idempotencyKey]
    );

    if (updated.rows[0].response_status !== null) {
      // Processing complete
      return res.status(updated.rows[0].response_status).json(updated.rows[0].response_body);
    }
  }

  // Timed out waiting
  return res.status(202).json({ error: "Still processing" });
}
```

**Pattern 2: Return 202 Accepted**

```typescript
const existing = await db.query<IdempotencyKey>(
  "SELECT * FROM idempotency_keys WHERE idempotency_key = $1",
  [idempotencyKey]
);

if (existing.rows.length > 0 && existing.rows[0].response_status === null) {
  // Still processing, come back later
  return res.status(202).json({ status: "processing" });
}
```

The client gets a 202 (Accepted) response and knows to check back later.

**Pattern 3: Use the same idempotency key to poll for status**

```typescript
app.get("/payments/status", async (req: Request, res: Response) => {
  const idempotencyKey = req.headers["idempotency-key"] as string;

  const keyRecord = await db.query<IdempotencyKey>(
    "SELECT * FROM idempotency_keys WHERE idempotency_key = $1",
    [idempotencyKey]
  );

  if (keyRecord.rows.length === 0) {
    return res.status(404).json({ error: "Not found" });
  }

  const record = keyRecord.rows[0];

  if (record.response_status === null) {
    return res.status(202).json({ status: "processing" });
  }

  return res.status(record.response_status).json(record.response_body);
});
```

## Beyond Simple Keys: Advanced Patterns

### Pattern 1: Request Body Hashing

Instead of using a random key, hash the request body. Same request body always produces the same hash.

```typescript
import crypto from "crypto";

function getRequestHash(method: string, path: string, body: unknown): string {
  const content = `${method}${path}${JSON.stringify(body)}`;
  return crypto.createHash("sha256").update(content).digest("hex");
}

app.post("/payments", async (req: Request, res: Response) => {
  const idempotencyKey = getRequestHash(req.method, req.path, req.body);

  // Use hash as idempotency key
  // Same request body = same hash = idempotent
});
```

Advantage: Automatic idempotency even if the client doesn't send an idempotency key.
Disadvantage: Relies on request body staying exactly the same. If the client changes field order, it's a different hash.

### Pattern 2: Distributed Idempotency with Redis

For high-throughput systems, checking the database for every request is slow. Use Redis as a fast cache.

```typescript
import redis from "redis";

const redisClient = redis.createClient();

async function createPaymentWithRedisCache(
  amount: number,
  accountId: number,
  idempotencyKey: string
) {
  const cacheKey = `idempotency:${idempotencyKey}`;

  // Check Redis first (fast)
  const cached = await redisClient.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // Process payment
  let response: Record<string, unknown>;
  let statusCode: number;

  try {
    await chargeAccount(accountId, amount);
    const payment = await createPayment(accountId, amount);

    response = {
      payment_id: payment.id,
      status: "success",
    };
    statusCode = 200;
  } catch (error) {
    response = {
      error: error instanceof Error ? error.message : "Unknown error",
    };
    statusCode = 400;
  }

  // Store in Redis (fast) and database (durable)
  const result = JSON.stringify(response);
  await redisClient.setEx(cacheKey, 86400, result); // 24 hours

  // Also store in database for durability
  await db.query(
    `INSERT INTO idempotency_keys 
     (idempotency_key, response_body, response_status)
     VALUES ($1, $2, $3)`,
    [idempotencyKey, result, statusCode]
  );

  return response;
}
```

Fast path: Check Redis, return cached result
Slow path: Process and store in both Redis and database

But be careful: Redis is not durable. If Redis crashes, you lose the cache but the database still has the record. That's fine for idempotency.

### Pattern 3: Idempotency with Message Queues

When processing messages from a queue, store the message ID or a hash of the message content.

```typescript
import { Consumer } from "kafkajs";

interface Message {
  id?: string;
  user_id: number;
  items: Array<{ product_id: number; quantity: number }>;
}

async function processOrderMessage(message: Message) {
  const messageId = message.id || hashMessage(message);

  // Check if we've processed this message
  const processed = await db.query("SELECT * FROM processed_messages WHERE message_id = $1", [
    messageId,
  ]);

  if (processed.rows.length > 0) {
    // Already processed, skip
    return;
  }

  // Process the order
  const order = await db.query(`INSERT INTO orders (user_id) VALUES ($1) RETURNING id`, [
    message.user_id,
  ]);

  // Insert order items
  for (const item of message.items) {
    await db.query(
      `INSERT INTO order_items (order_id, product_id, quantity) 
       VALUES ($1, $2, $3)`,
      [order.rows[0].id, item.product_id, item.quantity]
    );
  }

  // Record that we processed it
  await db.query("INSERT INTO processed_messages (message_id) VALUES ($1)", [messageId]);
}
```

This prevents duplicate orders if the message queue redelivers.

### Pattern 4: Conditional Idempotency Based on Operation Type

Different operations need different idempotency strategies.

```typescript
app.post("/orders", async (req: Request, res: Response) => {
  // This is a CREATE, needs idempotency
  const idempotencyKey = req.headers["idempotency-key"] as string;
  if (!idempotencyKey) {
    return res.status(400).json({
      error: "Idempotency-Key required for create",
    });
  }

  return createOrderWithIdempotency(idempotencyKey, req.body, res);
});

app.put("/orders/:orderId", async (req: Request, res: Response) => {
  // This is an UPDATE, also needs idempotency
  const idempotencyKey = req.headers["idempotency-key"] as string;
  if (!idempotencyKey) {
    return res.status(400).json({
      error: "Idempotency-Key required for update",
    });
  }

  return updateOrderWithIdempotency(req.params.orderId, idempotencyKey, req.body, res);
});

app.get("/orders/:orderId", async (req: Request, res: Response) => {
  // This is a READ, no idempotency needed
  // GET is safe to call multiple times anyway
  const order = await db.query("SELECT * FROM orders WHERE id = $1", [req.params.orderId]);

  return res.json(order.rows[0]);
});
```

CREATEs and UPDATEs need idempotency. GETs don't.

## Common Mistakes

**Mistake 1: Storing only successful responses**

```typescript
// WRONG
if (operationSucceeded) {
  await IdempotencyKey.create({ response: result });
}
// If operation fails, no key is stored
// Next retry will try again
```

Store both successes and failures:

```typescript
// CORRECT
let result: any;
try {
  result = await operation();
} catch (error) {
  result = { error: error instanceof Error ? error.message : "Unknown error" };
}

await IdempotencyKey.create({ response: result });
```

**Mistake 2: Not cleaning up old keys**

Idempotency keys pile up in the database forever. Add a cleanup job:

```typescript
// Run daily
async function cleanupOldIdempotencyKeys() {
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - 30); // 30 days ago

  await db.query("DELETE FROM idempotency_keys WHERE created_at < $1", [cutoffDate]);
}
```

**Mistake 3: Idempotency key is too short**

```typescript
// BAD: Collision risk
const idempotencyKey = userId.toString();

// GOOD: Unique per attempt
const idempotencyKey = `${userId}-${action}-${timestamp}-${randomString}`;
```

**Mistake 4: Different servers don't share idempotency store**

If you have multiple API servers and each stores idempotency keys locally:

```typescript
// Request 1 -> Server A: Creates payment, stores key in A's cache
// Request 2 (retry) -> Server B: No key found in B's cache, creates payment again
```

Use a shared database or Redis for idempotency keys.

**Mistake 5: Returning wrong status codes**

```typescript
// WRONG: Returning 200 for an error
if (idempotencyKeyExistsWithError) {
  return res.status(200).json(errorResponse); // Client thinks it succeeded
}

// CORRECT: Returning the same status code as before
if (idempotencyKeyExistsWithError) {
  return res.status(statusCodeFromBefore).json(errorResponse);
}
```

## Real Production Example: Payment System

Let's put it all together with a real payment system:

```typescript
import express, { Request, Response, NextFunction } from "express";
import { db } from "./database";

interface IdempotencyKeyRecord {
  id: number;
  idempotency_key: string;
  request_method: string;
  request_path: string;
  response_status: number | null;
  response_body: Record<string, unknown>;
  processing_started_at: Date;
  processing_completed_at: Date | null;
  expires_at: Date;
}

interface PaymentRecord {
  id: number;
  user_id: number;
  amount: number;
  status: string;
  idempotency_key: string;
  created_at: Date;
}

// Middleware to handle idempotency for POST/PUT requests
async function handleIdempotency(req: Request, res: Response, next: NextFunction) {
  const idempotencyKey = req.headers["idempotency-key"] as string;

  if (["POST", "PUT"].includes(req.method) && !idempotencyKey) {
    return res.status(400).json({ error: "Idempotency-Key header required" });
  }

  if (idempotencyKey) {
    // Check if we have a record of this request
    const record = await db.query<IdempotencyKeyRecord>(
      `SELECT * FROM idempotency_keys 
       WHERE idempotency_key = $1 
       AND request_method = $2 
       AND request_path = $3`,
      [idempotencyKey, req.method, req.path]
    );

    if (record.rows.length > 0) {
      const row = record.rows[0];

      if (row.response_status === null) {
        // Still processing
        return res.status(202).json({ status: "processing" });
      }

      // Return cached response
      return res.status(row.response_status).json(row.response_body);
    }

    // Insert a placeholder to lock this request
    try {
      const expiresAt = new Date();
      expiresAt.setDate(expiresAt.getDate() + 1); // 24 hours from now

      await db.query(
        `INSERT INTO idempotency_keys 
         (idempotency_key, request_method, request_path, response_status, 
          response_body, processing_started_at, expires_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        [idempotencyKey, req.method, req.path, null, JSON.stringify({}), new Date(), expiresAt]
      );
    } catch (error) {
      // Another request beat us to it, try again
      const retry = await db.query<IdempotencyKeyRecord>(
        "SELECT * FROM idempotency_keys WHERE idempotency_key = $1",
        [idempotencyKey]
      );

      if (retry.rows.length > 0) {
        const row = retry.rows[0];
        if (row.response_status === null) {
          return res.status(202).json({ status: "processing" });
        }
        return res.status(row.response_status).json(row.response_body);
      }
    }
  }

  // Store the idempotency key in res.locals for cleanup
  res.locals.idempotencyKey = idempotencyKey;
  next();
}

// Error handler to store failed responses
async function storeIdempotencyResult(
  idempotencyKey: string,
  statusCode: number,
  responseBody: Record<string, unknown>
) {
  await db.query(
    `UPDATE idempotency_keys 
     SET response_status = $1, 
         response_body = $2, 
         processing_completed_at = $3
     WHERE idempotency_key = $4`,
    [statusCode, JSON.stringify(responseBody), new Date(), idempotencyKey]
  );
}

const app = express();
app.use(express.json());

app.post("/payments", handleIdempotency, async (req: Request, res: Response) => {
  const { user_id, amount } = req.body;
  const idempotencyKey = res.locals.idempotencyKey as string;

  if (!user_id || !amount) {
    const errorResponse = { error: "user_id and amount required" };
    if (idempotencyKey) {
      await storeIdempotencyResult(idempotencyKey, 400, errorResponse);
    }
    return res.status(400).json(errorResponse);
  }

  try {
    // Charge the account (external service call)
    const chargeResult = await chargePaymentProvider(user_id, amount);

    if (!chargeResult.success) {
      throw new Error(`Payment failed: ${chargeResult.error}`);
    }

    // Create payment record
    const paymentResult = await db.query<PaymentRecord>(
      `INSERT INTO payments (user_id, amount, status, idempotency_key, created_at)
       VALUES ($1, $2, $3, $4, $5)
       RETURNING *`,
      [user_id, amount, "completed", idempotencyKey, new Date()]
    );

    const payment = paymentResult.rows[0];
    const response = {
      payment_id: payment.id,
      status: "completed",
      amount: payment.amount,
    };

    // Store the successful result
    if (idempotencyKey) {
      await storeIdempotencyResult(idempotencyKey, 200, response);
    }

    return res.status(200).json(response);
  } catch (error) {
    const errorResponse = {
      error: error instanceof Error ? error.message : "Unknown error",
    };

    // Store the error result
    if (idempotencyKey) {
      await storeIdempotencyResult(idempotencyKey, 500, errorResponse);
    }

    return res.status(500).json(errorResponse);
  }
});

app.delete("/jobs/cleanup-idempotency", async (req: Request, res: Response) => {
  // Clean up old idempotency keys (run daily)
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - 30); // 30 days ago

  const result = await db.query("DELETE FROM idempotency_keys WHERE expires_at < $1 RETURNING *", [
    cutoffDate,
  ]);

  return res.json({ deleted: result.rowCount });
});
```

Usage:

```typescript
// Client code
import axios from "axios";
import { v4 as uuidv4 } from "uuid";

async function makePayment(userId: number, amount: number): Promise<any> {
  const idempotencyKey = `${userId}-payment-${uuidv4()}`;

  try {
    const response = await axios.post(
      "https://api.example.com/payments",
      {
        user_id: userId,
        amount: amount,
      },
      {
        headers: {
          "Idempotency-Key": idempotencyKey,
        },
      }
    );

    if (response.status === 202) {
      // Still processing, retry later
      await new Promise((resolve) => setTimeout(resolve, 1000));
      return makePayment(userId, amount); // Retry with same key
    }

    return response.data;
  } catch (error) {
    throw error;
  }
}
```

## Monitoring Idempotency

Track these metrics to know if idempotency is working:

```typescript
// How many requests are idempotent retries?
const totalKeys = await db.query("SELECT COUNT(*) as count FROM idempotency_keys");
const retryKeys = await db.query(
  "SELECT COUNT(*) as count FROM idempotency_keys WHERE processing_completed_at IS NULL"
);

const idempotencyHitRate = retryKeys.rows[0].count / totalKeys.rows[0].count;

// Are we getting duplicate payments?
const duplicates = await db.query(`
  SELECT idempotency_key, COUNT(*) as count 
  FROM payments 
  GROUP BY idempotency_key 
  HAVING COUNT(*) > 1
`);

const duplicateCount = duplicates.rowCount;

// How long does it take to process?
const avgProcessingTime = await db.query(`
  SELECT AVG(
    EXTRACT(EPOCH FROM (processing_completed_at - processing_started_at))
  ) as avg_seconds
  FROM idempotency_keys
  WHERE processing_completed_at IS NOT NULL
`);

console.log(`Average processing time: ${avgProcessingTime.rows[0].avg_seconds}s`);
```

## Wrapping Up

Idempotency is one of those things that's invisible when it works and catastrophic when it doesn't. A user clicks a button twice and you charge them twice. A payment processor retries and you process it twice. A message gets delivered twice and you duplicate an order.

The pattern is simple:

1. Client generates a unique key per request
2. Server checks if the key was seen before
3. If yes, return cached result
4. If no, process and store the result with the key
5. Client can safely retry with the same key

Implement it early. Build it into your payment processing, order creation, and any other mutation. The cost of adding it is minimal. The cost of not having it is everything.
