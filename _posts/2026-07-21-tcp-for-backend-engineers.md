---
title: "TCP for Backend Engineers: From Zero to Practical Mastery"
date: 2026-07-21
categories: ["Distributed Systems"]
tags: [tcp, networking, backend, distributed-systems]
---
## 1. What TCP Actually Is, and Why It Exists

Start with the network underneath TCP, because that's the whole reason TCP exists. The internet moves data in packets, and the protocol that handles that movement, IP, makes no promises. It does not guarantee your packet arrives. It does not guarantee it arrives in order. It does not guarantee it only arrives once. A router can drop a packet because its queue is full, a path can change mid-transfer and cause two packets sent seconds apart to arrive in reverse order, and in rare cases a packet can get duplicated by a retry somewhere along the path. IP shrugs at all of this. It was designed to move packets, not to guarantee anything about them.

That's a problem for almost every application you'd want to build, because nobody wants to write "check if this arrived, check if it arrived twice, check if it arrived in the wrong order" into every single app. So TCP was built as a layer above IP that takes on exactly that job. It keeps state on both ends of a connection, numbers everything it sends, waits for confirmation that it arrived, and resends anything that seems to have gone missing. The result is something that feels, from the application's point of view, like writing into one end of a pipe and having the exact same bytes come out the other end, in the same order, with nothing missing and nothing duplicated.

That's the whole value proposition in one sentence: TCP turns an unreliable network into something predictable enough that you can build software on top of it. It does this without changing anything about how packets get routed. IP still just moves packets and TCP still just tracks what happened and fixes what needs fixing.

## 2. The Handshake and the Life of a Connection

A TCP connection doesn't just start when you call `connect()`. Something happens on the wire first, and it's worth knowing what, because a stuck or slow connection is often stuck at exactly this step.

It's a three-step exchange. Your client sends a packet with the SYN flag set, which is basically "I want to open a connection, here's the sequence number I'm starting from." The server responds with SYN-ACK, which means "got it, here's my own starting sequence number, and I acknowledge yours." Your client responds with ACK, confirming the server's number, and at that point both sides have agreed on where to start counting bytes and the connection is considered open. This is the three-way handshake, and until it completes, your `connect()` call is just sitting there blocked.

This matters practically because that handshake takes at least one full round trip, and if the server is slow to accept, or the SYN packet gets lost and has to be retried, your app is going to see that as latency before it's even sent a single byte of actual data. If you've ever seen a connection take way longer to establish than the request itself, this is often where the time went.

Closing works differently depending on how it happens. A clean close is done with the FIN flag, each side sends FIN when it's done writing, the other side acknowledges it, and eventually both directions are shut. This is what a normal, graceful disconnect looks like, and it's what you want. But connections don't always close cleanly. If something goes wrong, like a peer trying to write to a connection it already considers dead, the other side sends a RST, a reset. A reset is abrupt. It tells the receiving side "forget this connection ever existed," and any buffered data that hadn't been delivered yet is just gone. The distinction between FIN and RST matters a lot when you're debugging, because a FIN usually means the other side intentionally hung up, while a RST usually means something broke, like the app crashed, or a firewall killed the connection, or you tried writing to a socket the peer had already closed.

Here's what that lifecycle actually looks like in Node, using the raw `net` module instead of something higher-level like HTTP, so you can see each step:

```js
const net = require("net");

const socket = net.connect({ host: "example.com", port: 80 }, () => {
  // this callback only fires after the three-way handshake finished
  console.log("connected, handshake done");
  socket.write("GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n");
});

socket.on("data", (chunk) => {
  console.log("got data:", chunk.length, "bytes");
});

socket.on("end", () => {
  // the peer sent a FIN, this is a clean close
  console.log("peer closed its end (FIN)");
});

socket.on("error", (err) => {
  // ECONNRESET here would mean the peer sent a RST instead
  console.log("socket error:", err.code);
});

socket.on("close", (hadError) => {
  console.log("socket fully closed, hadError:", hadError);
});
```

The `connect` callback is your `connect()` completing, the `end` event is a FIN arriving, and if you instead saw an `error` event with `ECONNRESET`, that's a RST.

## 3. It's a Byte Stream, Not Messages

This is the part that trips people up the most when they're new to networking, because it doesn't match the mental model you build from using higher-level tools like HTTP libraries or message queues.

TCP does not preserve the boundaries of your writes. If you call `send()` three times with three separate chunks of data, TCP does not guarantee the receiver gets three separate reads matching those three chunks. It might combine two of your writes into a single segment and deliver them together. It might split one write across two segments if the data was large. On the receiving end, calling `recv()` might give you less than what was sent in a single write, or it might give you data from multiple writes glued together. What TCP guarantees is that the bytes come out in the same order they went in. It says nothing about where one message ends and another begins.

This is exactly why every real protocol built on top of TCP has to invent its own way of marking message boundaries. HTTP does it by putting a `Content-Length` header so the receiver knows exactly how many bytes to read for the body, or by using chunked encoding where each chunk announces its own size. Some protocols use a fixed-size length prefix before every message, so the reader always knows how many bytes to pull off the stream to get one complete message. Others use a delimiter character, like a newline, to mark where a message ends, and just scan for it.

If you're writing your own protocol over raw TCP, and plenty of backend systems still do this for internal services, this is the first design decision you have to make and get right. If you don't frame your messages explicitly, you will eventually see bugs where a single message gets split across two reads under load, or two small messages get glued into one read, and your parser breaks in ways that are hard to reproduce because it depends on timing and packet sizes.

Here's a small example of what that actually looks like, a simple length-prefixed framer that buffers incoming chunks and only emits a complete message once it has enough bytes:

```js
const net = require("net");

function makeFramer(onMessage) {
  let buffer = Buffer.alloc(0);

  return function onData(chunk) {
    buffer = Buffer.concat([buffer, chunk]);

    // keep pulling messages out as long as we have a full length prefix + body
    while (buffer.length >= 4) {
      const messageLength = buffer.readUInt32BE(0);
      if (buffer.length < 4 + messageLength) break; // haven't got the full message yet

      const message = buffer.subarray(4, 4 + messageLength);
      onMessage(message);
      buffer = buffer.subarray(4 + messageLength);
    }
  };
}

const server = net.createServer((socket) => {
  socket.on(
    "data",
    makeFramer((msg) => {
      console.log("got one complete message:", msg.toString());
    }),
  );
});

server.listen(9000);
```

Without something like this, `socket.on('data', ...)` just hands you whatever arrived in that particular read, which could be half a message, one full message, or three messages stuck together. This framer is what turns that raw byte stream back into discrete messages your application can actually work with.

## 4. What You Actually See Through the Socket API

Most of what a backend engineer needs to know about TCP shows up as specific, nameable things you encounter through the socket API or through whatever library wraps it. Knowing what these mean saves you a lot of guessing during incidents.

A successful `connect()` means the three-way handshake finished and the kernel has connection state set up on your end. If `connect()` hangs, you're either waiting on the network, waiting on a slow or overloaded server, or hitting something like a firewall silently dropping your SYN packets instead of rejecting them outright, which is worse because you don't get a fast failure, you just wait until your timeout fires.

A `recv()` call that returns `0` bytes means the peer performed an orderly close. It sent a FIN, it's done writing, and there's nothing more coming on this connection. This is different from an error. It's the normal way a TCP connection tells you "the other side is finished," and your code should treat it as a clean end of stream, not as a failure.

`ECONNRESET` is what you get when the connection was torn down with a RST instead of a FIN. A very common real scenario: you have a connection sitting idle in a pool, a load balancer or NAT device decides it's been idle too long and kills it on its end, then your app tries to write to that connection without knowing it's already dead, and the OS reports `ECONNRESET` back to you because the peer answered with a reset. This is one of the most common causes of intermittent errors in systems that reuse connections from a pool, especially when the pool doesn't validate connections before handing them out.

`EPIPE`, or "broken pipe," is closely related. It shows up when you try to write to a socket that's already been closed on the other end, often after you've already gotten a reset or a close and tried writing again anyway. On some systems this also triggers a `SIGPIPE` signal that can kill your process outright if you're not handling it, which is a classic gotcha in low-level networking code.

In Node, all three of these show up through the `error` event, and the `err.code` is what tells you which one you're actually looking at:

```js
socket.on("error", (err) => {
  switch (err.code) {
    case "ECONNRESET":
      console.log("peer reset the connection, probably an idle timeout somewhere");
      break;
    case "EPIPE":
      console.log("tried writing to a socket the peer already closed");
      break;
    case "ETIMEDOUT":
      console.log("connect() or a read never got a response in time");
      break;
    default:
      console.log("other socket error:", err.code);
  }
});
```

Node won't crash your process with a raw `SIGPIPE` the way a C program might, it converts that into a normal `EPIPE` error event, but the underlying cause is exactly the same thing: you wrote to a connection that was already dead on the other end.

And then there's the timeout, which isn't really a distinct error from TCP's point of view, it's just the absence of a response within however long you decided to wait. TCP itself doesn't have a built-in concept of "this request is taking too long," that's an application-level decision you make yourself.

## 5. How TCP Actually Recovers From Loss and Corruption

Now the mechanics behind the reliability, because "TCP resends lost data" is true but too vague to be useful when you're trying to reason about behavior under real network conditions.

Every byte TCP sends is numbered, using a sequence number. When the receiver gets a segment, it sends back an acknowledgment naming the sequence number it's expecting next, effectively saying "I've got everything up to this point." If the sender doesn't get an acknowledgment for something within a certain window of time, it assumes that segment was lost and sends it again. That's retransmission, and it's the core mechanism that makes TCP reliable even though IP underneath it is not.

There's a subtlety here that matters a lot in practice: TCP delivers to the application in order, which means if segment 3 is lost but segments 4 and 5 already arrived, the receiver holds onto 4 and 5 without handing them to your application yet, because handing them over would break the ordering guarantee. Your app doesn't see any of that data until segment 3 gets retransmitted and arrives. This is called head-of-line blocking, and it's the reason a single lost packet can cause a visible stall even when most of your data already made it across.

Corruption is a separate problem from loss, and it's handled differently. Every TCP segment carries a checksum, computed from the segment's contents. When a segment arrives, the receiver recomputes the checksum and compares it to what was sent. If they don't match, something got corrupted in transit, maybe a stray bit flipped due to a hardware issue somewhere along the path, and the receiver just discards that segment as if it never arrived. From the sender's perspective this looks identical to loss: no acknowledgment shows up, so it gets retransmitted. It's worth knowing that TCP's checksum is a fairly weak one by modern standards, it can miss some patterns of corruption, which is one of the reasons application-level integrity checks, like checksums or hashes on top of your actual payload, still matter for anything where correctness is critical, even though TCP is already checking underneath.

## 6. Flow Control and Congestion Control

These are two different problems and it's worth keeping them separate in your head, because they get invoked to explain very different kinds of slowness.

Flow control protects the receiver from being overwhelmed by data it can't process fast enough. Each side advertises a receive window, telling the other side how many unacknowledged bytes it's willing to have in flight at once. If your receiver is slow to read from its socket buffer, the window shrinks, and the sender has to slow down, no matter how fast the network itself could carry data. This is why a slow consumer on one end of a connection can end up throttling a fast producer on the other end, purely through the mechanics of the receive window.

Congestion control protects the network itself, not just the receiver. TCP starts a new connection cautiously, sending a small amount of data and watching for signs of trouble, then gradually sends more if things are going well, a pattern usually called slow start. If it detects loss, which it interprets as a signal of congestion somewhere on the path, it backs off, sending less for a while before ramping back up. This is why a fresh connection with no prior history is often slower at the very start than a connection that's been running for a while and has had time to ramp up, and it's part of why persistent connections tend to perform better than opening a new one for every request.

For anyone dealing with large uploads, streaming responses, or clients on flaky or high-latency networks, both of these mechanisms are usually the actual explanation when throughput looks worse than the raw bandwidth of the link would suggest. It's rarely "the network is slow," it's more often "the window is small" or "congestion control just backed off."

## 7. Nagle's Algorithm and TCP_NODELAY

This one is small but it's caused a lot of confusing latency bugs over the years, so it's worth knowing by name.

By default, TCP tries to avoid sending a lot of tiny packets, because each packet carries overhead, and a stream of many small writes turned into many small packets is inefficient. Nagle's algorithm handles this by holding onto small amounts of unacknowledged data briefly, waiting to see if more data is about to be written so it can be bundled into a single packet, instead of firing off a packet for every small write.

That's a reasonable trade-off for something like a file transfer, where throughput matters more than the latency of any individual write. It's a bad trade-off for something like a chat app or an API that sends small, latency-sensitive messages back and forth, where you actually want each write to go out immediately rather than sitting around waiting to see if it can be combined with something else. This is exactly what `TCP_NODELAY` is for. Setting that socket option disables Nagle's algorithm for that connection, so small writes go out right away instead of being held. If you've ever seen mysterious, consistent latency in the tens of milliseconds range on a connection sending small messages, and nothing about the network itself explains it, this setting is one of the first things worth checking.

In Node, this is a single call on the socket:

```js
const socket = net.connect({ host: "api.internal", port: 8080 }, () => {
  socket.setNoDelay(true); // disables Nagle's algorithm for this connection
});
```

Worth doing this on connections where you're sending small, frequent, latency-sensitive writes, like an internal RPC connection between services, and probably not worth touching on something like a bulk file transfer where you'd rather have fewer, fuller packets.

## 8. Keepalive vs. Application-Level Heartbeats

TCP has its own keepalive mechanism, but it's easy to misunderstand what it's actually for. When enabled, TCP keepalive periodically sends a probe on an idle connection to check that the peer is still there, and if it stops getting responses, it eventually gives up and reports the connection as dead. The problem is that the default intervals for this are usually very long, often measured in hours, and they're a low-level, coarse mechanism meant to catch cases like "the machine on the other end lost power and vanished without ever sending a FIN."

For a backend service, this is almost never fast enough to be useful for detecting real application-level problems, so most systems build their own heartbeat on top of it. Instead of relying on TCP's own probes, the application sends periodic small messages, like a ping, at whatever interval actually matches how fast you need to detect a dead peer, and expects a response within some timeout. If nothing comes back, the app decides the connection is dead and closes it itself, rather than waiting for TCP's much slower built-in mechanism to eventually notice. Long-lived connections, like WebSocket connections or persistent internal service links, almost always need this kind of application-level heartbeat, because relying on TCP's default keepalive settings alone tends to mean you find out a connection died far later than you'd like.

A minimal version of this over a raw TCP socket looks something like this, sending a ping on an interval and closing the connection if no pong comes back in time:

```js
function attachHeartbeat(socket, { intervalMs = 15000, timeoutMs = 5000 } = {}) {
  let awaitingPong = false;
  let timeoutHandle;

  const interval = setInterval(() => {
    if (awaitingPong) {
      // no pong arrived since the last ping, treat the connection as dead
      socket.destroy(new Error("heartbeat timeout, no pong received"));
      return;
    }
    awaitingPong = true;
    socket.write(JSON.stringify({ type: "ping" }) + "\n");
    timeoutHandle = setTimeout(() => {
      if (awaitingPong) socket.destroy(new Error("heartbeat timeout, no pong received"));
    }, timeoutMs);
  }, intervalMs);

  socket.on("data", (chunk) => {
    if (chunk.toString().includes('"type":"pong"')) {
      awaitingPong = false;
      clearTimeout(timeoutHandle);
    }
  });

  socket.on("close", () => clearInterval(interval));
}
```

This is deliberately simple, real implementations usually parse frames properly rather than checking for a substring, but the shape is the same: send something on a short interval, expect a response within a short window, and give up on the connection yourself instead of waiting for TCP's own keepalive to eventually notice, which could take a lot longer than your application can tolerate.

## 9. Bugs You'll Actually Run Into in Production

A handful of patterns show up again and again once TCP is running under real production load, and it's worth being able to recognize them by shape.

Connection pool exhaustion happens when your pool of reusable connections gets held onto for longer than expected, often because something downstream is slow to respond and requests are piling up waiting for a connection to free up, and eventually new requests just can't get a connection at all and start failing or queuing.

Half-open connections happen when one side thinks a connection is still alive but the other side has already silently dropped it, often because of a device in between, like a NAT gateway or load balancer, that closed it after seeing no activity for a while, without ever telling either endpoint. The next time your app tries to actually use that connection, it gets a reset it wasn't expecting.

Timeouts and resets look similar from a distance, "the request failed," but they mean different things and point you in different directions. A timeout usually means nothing came back at all within the time you were willing to wait, which points at things being slow, whether that's the network, the server, or something queued up in between. A reset means the connection was actively torn down by something, which points at things being explicitly killed, whether that's the peer, a firewall, or a load balancer's idle timeout.

And then there's the classic "it works locally but breaks in production" pattern, which is almost always about the difference in environment between your laptop and a real deployment: no load balancer or reverse proxy in the middle silently closing idle connections, no real network latency, no packet loss, and often a connection pool that's actually sized for one user hitting it, you, rather than production traffic. TCP itself doesn't change between environments, but everything sitting around it, load balancers, proxies, firewalls, NAT devices, does, and a lot of what looks like a TCP bug is actually one of those middleboxes behaving differently than you assumed.

## 10. Debugging Tools and What to Actually Look For

When something's wrong at the TCP level, a handful of tools cover most of what you need.

`netstat` and its more modern replacement `ss` show you the state of connections on a machine: which ones are established, which are stuck in a state like `TIME_WAIT` or `CLOSE_WAIT`, and how many connections exist to a given remote address. A pile of connections sitting in `CLOSE_WAIT` on your server usually means your application accepted a close from the peer but never actually closed its own end of the socket, which is a real, fixable bug in your code, not a network issue. A large number of connections in `TIME_WAIT` is normal after a lot of connections have closed and is usually not something to worry about, though it can matter if you're opening and closing a huge number of short-lived connections very quickly.

`tcpdump` captures the actual packets going across the wire, and it's what you reach for when you need to see exactly what happened at the protocol level, whether the handshake completed, whether a FIN or a RST closed the connection, whether retransmissions are happening, and how much time passed between each step. It's more effort to read than `ss`, but it's the only one of these tools that shows you the literal sequence of events on the wire rather than just the current state, which matters a lot when you're trying to reconstruct what happened during an intermittent failure.

Between the two, `ss` is usually where you start, since it's fast and shows you the shape of the problem, like whether connections are piling up in a state they shouldn't be in, and `tcpdump` is where you go next, once you need to see exactly what happened on the wire during a specific failure.
