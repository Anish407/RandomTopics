# TCP Congestion Window and How It Works with HTTP

## 1. What is the TCP Congestion Window?

The **TCP congestion window**, usually called **`cwnd`**, is TCP’s internal limit on **how much data a sender is allowed to have “in flight” before receiving acknowledgements**.

In simple terms:

> **`cwnd` controls how aggressively TCP sends data into the network.**

It is not the same as the receiver window.

---

## 2. The Core Idea

TCP does not blast unlimited packets into the network. It sends some data, waits for ACKs, and then decides whether it is safe to send more.

The congestion window is measured in bytes, but it is often explained in terms of **MSS-sized segments**.

Example:

```text
MSS = 1460 bytes
cwnd = 10 MSS

Sender can have:
10 × 1460 = 14,600 bytes

in flight before it must wait for ACKs.
```

**In flight** means:

```text
Data sent by the sender
but not yet acknowledged by the receiver
```

So if:

```text
cwnd = 10 MSS
```

TCP can send 10 packets, then it has to wait for ACKs before sending more.

---

## 3. Why Does TCP Need `cwnd`?

Because the internet is shared.

Imagine this path:

```text
Your laptop → Router → ISP → Server
```

There may be limited capacity somewhere in the middle. If every sender transmits at full speed, routers’ buffers fill up, packets get dropped, latency increases, and the network becomes unstable.

So TCP uses `cwnd` to estimate:

> “How much data can I safely push into the network without causing congestion?”

That estimate changes dynamically.

---

## 4. `cwnd` vs Receive Window

This is very important.

TCP sending is limited by two windows:

```text
actual send limit = min(congestion window, receive window)
```

### 4.1 Receive Window — `rwnd`

This is controlled by the **receiver**.

It says:

> “This is how much buffer space I have available.”

Example:

```text
Receiver says: I can accept 64 KB.
rwnd = 64 KB
```

### 4.2 Congestion Window — `cwnd`

This is controlled by the **sender**.

It says:

> “This is how much I believe the network can handle.”

Example:

```text
Sender estimates: network can handle 32 KB.
cwnd = 32 KB
```

Then TCP uses:

```text
min(rwnd, cwnd)
```

Example:

```text
rwnd = 64 KB
cwnd = 32 KB

actual sending limit = 32 KB
```

The receiver may be ready for more, but TCP does not trust the network enough yet.

---

## 5. Analogy

Imagine you are pouring water through a pipe.

```text
Your bucket = sender
Pipe = network
Other end = receiver
```

The receiver may say:

> “I have a big tank. Send me a lot.”

That is the **receive window**.

But the pipe may be narrow or congested.

The sender says:

> “I will start slowly and increase the flow only if the pipe handles it.”

That is the **congestion window**.

---

## 6. How `cwnd` Changes

TCP increases or decreases `cwnd` based on network feedback.

### 6.1 ACKs Arrive Successfully

TCP thinks:

> “Good. The network handled this. I can send more.”

So `cwnd` increases.

### 6.2 Packet Loss Happens

TCP assumes:

> “The network is congested. I sent too much.”

So `cwnd` decreases.

Packet loss is one of the main signals TCP uses to detect congestion.

---

## 7. Slow Start

When a TCP connection begins, TCP does not know the available network capacity.

So it starts with a small `cwnd` and grows quickly.

Example:

```text
Round 1: cwnd = 1 MSS
Round 2: cwnd = 2 MSS
Round 3: cwnd = 4 MSS
Round 4: cwnd = 8 MSS
Round 5: cwnd = 16 MSS
```

This is called **slow start**, which is a slightly misleading name because the growth is actually exponential.

The “slow” part means TCP starts cautiously, not that it grows slowly.

---

## 8. Congestion Avoidance

At some point, TCP stops growing aggressively and becomes more careful.

Instead of doubling the window every round trip, it increases more gradually.

Example:

```text
cwnd = 16 MSS
cwnd = 17 MSS
cwnd = 18 MSS
cwnd = 19 MSS
```

This is called **congestion avoidance**.

The goal is to probe for more bandwidth without overwhelming the network.

---

## 9. What Happens When Packet Loss Occurs?

Suppose `cwnd` reaches 32 MSS and packet loss happens.

TCP may reduce the window, often roughly like this:

```text
Before loss:
cwnd = 32 MSS

After loss:
cwnd = 16 MSS
```

Then it slowly increases again.

This behavior is often called **AIMD**:

```text
Additive Increase, Multiplicative Decrease
```

Meaning:

```text
Increase slowly.
Decrease sharply.
```

This is why TCP behaves fairly well when many connections share the same network.

---

## 10. Example Timeline

Imagine a TCP connection downloading a file.

```text
Start:
cwnd = 1 MSS

ACKs arrive:
cwnd = 2 MSS

More ACKs arrive:
cwnd = 4 MSS

More ACKs arrive:
cwnd = 8 MSS

Still no packet loss:
cwnd = 16 MSS

Packet loss happens:
cwnd drops to 8 MSS

Then it grows again:
9 MSS
10 MSS
11 MSS
...
```

So the sending rate looks like a sawtooth pattern:

```text
increase → increase → increase → loss → decrease → increase again
```

---

## 11. How `cwnd` Affects Speed

Your TCP throughput is roughly related to:

```text
throughput ≈ cwnd / RTT
```

Where:

```text
cwnd = amount of data allowed in flight
RTT = round-trip time
```

Example:

```text
cwnd = 64 KB
RTT = 100 ms
```

Throughput:

```text
64 KB / 0.1 seconds = 640 KB/s
```

So even if the network bandwidth is high, a small congestion window or high latency can limit throughput.

This is why long-distance connections can be slower unless TCP grows its window enough.

---

## 12. Why This Matters in System Design

For backend and system design, this matters when thinking about:

```text
API latency
file uploads/downloads
streaming
database replication
cross-region traffic
load balancers
CDNs
high-throughput services
```

Example:

If your client is in Sweden and your server is in `us-east-1`, the RTT is much higher than if your server is in Europe.

Higher RTT means ACKs take longer to return.

Since TCP uses ACKs to grow and maintain the congestion window, long-distance traffic may take longer to reach full throughput.

That is one reason CDNs like CloudFront help. They reduce distance between client and edge, improving latency and often throughput.

---

## 13. Common Misunderstanding

A lot of people think TCP speed is only about bandwidth.

That is wrong.

TCP performance depends heavily on:

```text
bandwidth
latency / RTT
packet loss
congestion window
receive window
MSS
TCP congestion algorithm
```

So a “1 Gbps network” does not automatically mean one TCP connection will instantly transfer at 1 Gbps.

The congestion window has to grow enough to fill the available pipe.

---

## 14. Bandwidth-Delay Product

**Bandwidth-delay product**, or **BDP**, means:

> How much data must be in flight to fully use the network?

Formula:

```text
BDP = bandwidth × RTT
```

Example:

```text
Bandwidth = 100 Mbps
RTT = 100 ms
```

Convert:

```text
100 Mbps = 12.5 MB/s
100 ms = 0.1 s
```

So:

```text
BDP = 12.5 MB/s × 0.1 s
BDP = 1.25 MB
```

That means you need around **1.25 MB of data in flight** to fully use that connection.

If your congestion window is only 64 KB, you cannot fully use the 100 Mbps link.

---

## 15. Simple Summary of TCP Congestion Window

The **TCP congestion window** is the sender-side limit that decides how much unacknowledged data TCP can put into the network.

It exists to prevent the sender from overwhelming the network.

```text
Small cwnd  → slower sending
Large cwnd  → faster sending
Packet loss → cwnd reduced
ACK success → cwnd increased
```

The final send limit is:

```text
min(cwnd, receiver window)
```

For system design, remember this:

> **TCP throughput is not just bandwidth. It is bandwidth + latency + packet loss + congestion window behavior.**

---

# 16. How Does This Work with HTTP?

HTTP does **not directly manage the TCP congestion window**.

HTTP says:

> “Send this request and response.”

TCP underneath decides:

> “How fast can I safely move these bytes across the network?”

So HTTP is the **application protocol**, and TCP is the **transport protocol** doing reliability, ordering, retransmission, flow control, and congestion control.

Think of it like this:

```text
HTTP:  GET /products/123
TCP:   I will split this into bytes/segments, send them reliably, wait for ACKs,
       grow/shrink cwnd depending on congestion, and deliver bytes in order.
IP:    I will route packets.
Ethernet/Wi-Fi/5G: I will move frames over the local network.
```

---

## 17. Simple Example: HTTP Request Over TCP

Suppose your browser opens:

```text
https://example.com/products
```

For HTTP over TCP, roughly this happens:

```text
1. DNS lookup
2. TCP connection established
3. TLS handshake, for HTTPS
4. Browser sends HTTP request
5. Server sends HTTP response
6. TCP controls how fast the bytes move
```

The HTTP request may look small:

```http
GET /products HTTP/1.1
Host: example.com
```

That request is just bytes. TCP takes those bytes and sends them through the network.

The response may be much larger:

```text
HTML page
CSS files
JavaScript files
images
JSON API responses
```

TCP does not know or care that it is HTML, JSON, CSS, or an image. To TCP, it is just a stream of bytes.

---

## 18. Where the Congestion Window Affects HTTP

Imagine the server is sending a 5 MB JavaScript file to your browser.

HTTP says:

```text
Here is the response body: 5 MB
```

TCP says:

```text
I cannot send all 5 MB immediately.
I can only send up to cwnd bytes before waiting for ACKs.
```

Example:

```text
cwnd = 10 MSS
MSS  = 1460 bytes

Initial send allowance ≈ 14.6 KB
```

So the server may initially send only around 14 KB, then wait for ACKs.

As ACKs return, TCP increases `cwnd` and sends more.

```text
Round 1: send 14 KB
ACKs arrive
Round 2: send more
ACKs arrive
Round 3: send more
...
```

So for large HTTP responses, `cwnd` directly affects download speed.

---

## 19. HTTP Does Not See Packets

This is a key mental model.

Your application code might do this in C#:

```csharp
return Results.File(fileBytes, "application/pdf");
```

Or:

```csharp
return Results.Json(product);
```

Your app is not manually choosing packets.

The layers below handle that:

```text
Your app writes bytes to socket
        ↓
TLS encrypts bytes, if HTTPS
        ↓
TCP splits byte stream into segments
        ↓
IP sends packets
        ↓
Network delivers them
```

So when your ASP.NET API returns a response, the app is not saying:

```text
Send 100 TCP packets now.
```

It is saying:

```text
Here are response bytes.
```

The OS TCP stack decides how much can actually be placed on the wire based on:

```text
congestion window
receiver window
socket buffer
MSS
RTT
packet loss
congestion algorithm
```

---

## 20. Small HTTP Response vs Large HTTP Response

### 20.1 Case 1: Small API Response

Example:

```json
{
  "id": 123,
  "name": "Keyboard",
  "price": 99
}
```

Maybe the response is only 1 KB.

If `cwnd` allows 14 KB initially, the whole response fits immediately.

```text
Initial cwnd ≈ 14 KB
Response size = 1 KB

Result:
response fits inside initial window
```

For small API responses, congestion window is usually not the biggest bottleneck.

Other things may matter more:

```text
latency
server processing time
TLS setup
database calls
DNS
serialization
authentication/authorization middleware
```

---

### 20.2 Case 2: Large HTTP Response

Example:

```text
100 MB video
50 MB file download
10 MB image
5 MB JavaScript bundle
large JSON export
```

Now the response cannot fit in the initial congestion window.

TCP has to gradually increase sending rate.

```text
Initial cwnd: small
ACKs arrive: cwnd grows
Loss occurs: cwnd drops
ACKs continue: cwnd grows again
```

For large downloads, `cwnd` matters a lot.

---

## 21. HTTP/1.1 and TCP Connections

With HTTP/1.1, browsers usually keep TCP connections alive and reuse them.

Old style:

```text
Request 1 → TCP connection → response → close
Request 2 → new TCP connection → response → close
```

Better style with keep-alive:

```text
TCP connection opened once

Request 1 → response
Request 2 → response
Request 3 → response
```

Why does this matter?

Because a new TCP connection starts with a relatively small congestion window.

If you reuse the same TCP connection, the congestion window may already have grown.

```text
New TCP connection:
cwnd starts small

Reused TCP connection:
cwnd may already be warm
```

This is one reason HTTP keep-alive matters.

---

## 22. The “Cold Start” Problem

When a new TCP connection starts, TCP does not know the network capacity yet.

So for a fresh connection:

```text
small cwnd
slow start
gradual ramp-up
```

For HTTP, this means the first request on a new connection can be slower, especially for larger responses.

Example:

```text
Client in Sweden → server in us-east-1
RTT is much higher than Sweden → Frankfurt
```

Higher RTT means each growth round takes longer.

So the congestion window grows more slowly in real time.

```text
Low RTT:
ACKs return quickly
cwnd grows quickly

High RTT:
ACKs return slowly
cwnd grows slowly
```

That is why geographic distance matters for HTTP performance.

---

## 23. Why CDNs Help HTTP Performance

Suppose your user is in Stockholm and your server is in the US.

Without CDN:

```text
Browser in Stockholm → origin server in us-east-1
```

Long RTT.

With CloudFront:

```text
Browser in Stockholm → nearby CloudFront edge
CloudFront edge → origin server
```

For cached content, the browser talks to a closer server.

That means:

```text
lower RTT
faster ACKs
faster cwnd growth
better throughput
lower latency
```

For static assets like images, JavaScript, CSS, and videos, this matters a lot.

For dynamic API responses, CDN can still help with:

```text
TLS termination
connection reuse
HTTP/2
edge caching where possible
reducing repeated long-distance handshakes
```

---

## 24. HTTP/2 Over TCP

HTTP/2 usually also runs over TCP.

The big difference is multiplexing.

With HTTP/1.1, one connection usually handles requests mostly sequentially, or the browser opens multiple TCP connections.

With HTTP/2:

```text
One TCP connection can carry many HTTP requests/responses at the same time.
```

Example:

```text
Single TCP connection:

Stream 1: /index.html
Stream 3: /app.js
Stream 5: /style.css
Stream 7: /logo.png
```

This is good because:

```text
fewer TCP connections
better connection reuse
less handshake overhead
better use of already-grown cwnd
```

But HTTP/2 over TCP has one nasty problem.

### 24.1 Head-of-Line Blocking at TCP Level

TCP guarantees ordered byte delivery.

So if one TCP packet is lost, TCP may hold back later bytes until the missing packet is retransmitted.

Even if those later bytes belong to a different HTTP/2 stream, the application may not receive them yet.

So HTTP/2 fixes some HTTP-level blocking, but it still suffers from TCP-level head-of-line blocking.

Important summary:

> HTTP/2 multiplexes many logical streams, but TCP still sees one ordered byte stream. One lost packet can temporarily block everything behind it.

---

## 25. HTTP/3 Is Different

HTTP/3 does **not** use TCP.

HTTP/3 uses **QUIC**, which runs over UDP.

```text
HTTP/1.1 → usually TCP
HTTP/2   → usually TCP
HTTP/3   → QUIC over UDP
```

QUIC still has congestion control, reliability, retransmission, and flow control, but it implements them differently in user space rather than relying on the OS TCP stack.

One major advantage:

```text
HTTP/3 avoids TCP-level head-of-line blocking between independent streams.
```

If one QUIC stream loses data, it does not necessarily block unrelated streams.

Main point:

> HTTP/1.1 and HTTP/2 depend on TCP’s congestion window. HTTP/3 does not use TCP, but QUIC has its own congestion-control mechanism with similar goals.

---

## 26. How This Affects Backend APIs

For normal API calls, the response may be small:

```text
2 KB JSON
10 KB JSON
50 KB JSON
```

For these, `cwnd` is usually less visible unless:

```text
the connection is new
RTT is high
there is packet loss
TLS handshakes happen repeatedly
many small resources are fetched separately
```

For large responses, it matters more:

```text
file downloads
large reports
large JSON exports
image/video delivery
container image pulls
large frontend bundles
```

In those cases, TCP congestion control can strongly affect throughput.

---

## 27. Example with ASP.NET API

Imagine this endpoint:

```csharp
app.MapGet("/report", async () =>
{
    var bytes = await File.ReadAllBytesAsync("large-report.pdf");
    return Results.File(bytes, "application/pdf");
});
```

Your application gives bytes to Kestrel.

Kestrel writes those bytes to the socket.

Then TCP decides:

```text
Can I send all bytes now?
No.

How much can I send?
min(cwnd, rwnd, socket buffer constraints)
```

So even if your C# code instantly produces a 100 MB file, the actual transfer speed depends on the network path.

Your app may finish generating the response quickly, but the client still receives it gradually.

---

## 28. Why Connection Pooling Matters

For backend-to-backend HTTP calls, this is very relevant.

Bad design:

```text
Create new HttpClient for every request
```

That can cause:

```text
new TCP connections
new TLS handshakes
cold cwnd
port exhaustion
poor performance
```

Better design in .NET:

```text
Use IHttpClientFactory
Reuse connections
Allow connection pooling
```

Because reused TCP connections avoid repeated setup and may benefit from an already-established congestion state.

In .NET, this is one practical reason you should not casually create and dispose `HttpClient` per request.

Example:

```csharp
services.AddHttpClient("orders", client =>
{
    client.BaseAddress = new Uri("https://orders.internal");
});
```

---

## 29. HTTP Request Body Uploads Also Use `cwnd`

It is not only responses.

If a client uploads a large file:

```text
POST /upload
Content-Length: 500 MB
```

Then the client’s TCP congestion window controls how fast the upload body is sent.

So:

```text
server response download → server's sending cwnd matters
client upload → client's sending cwnd matters
```

The sender always has the congestion window.

For download:

```text
server is sender
server cwnd matters
```

For upload:

```text
client is sender
client cwnd matters
```

For bidirectional protocols like WebSockets or gRPC streaming:

```text
both sides send data
both sides maintain their own cwnd
```

---

## 30. HTTP Timeout Confusion

This matters in real systems.

Suppose your API generates a large report and streams it to the client.

The backend may be fine. The database may be fine. But the transfer may be slow because of:

```text
high RTT
packet loss
small cwnd
client network issues
proxy buffering
load balancer idle timeout
server timeout
client timeout
```

So when someone says:

> “The API is slow.”

You should ask:

```text
Is server processing slow?
Is network transfer slow?
Is the response too large?
Is the connection reused?
Is there packet loss?
Is the client far away?
Is the load balancer/proxy timing out?
```

Do not blindly blame the database.

---

## 31. HTTP/1.1 Multiple Connections vs HTTP/2 Single Connection

Browsers historically opened multiple TCP connections per domain to download resources in parallel.

Example HTTP/1.1:

```text
Connection 1: index.html
Connection 2: app.js
Connection 3: style.css
Connection 4: image1.png
Connection 5: image2.png
```

Each connection has its own congestion window.

This can increase parallelism but also causes more handshakes and more congestion-control competition.

HTTP/2 instead tries:

```text
One TCP connection
Many streams
One congestion window
```

This is cleaner, but packet loss can affect all streams because TCP delivers one ordered stream.

So there is no free lunch.

---

## 32. Important Mental Model

For HTTP over TCP:

```text
HTTP message size affects how much data needs to be transferred.
TCP cwnd affects how quickly that data can be transferred.
RTT affects how quickly cwnd can grow.
Packet loss causes cwnd to shrink.
Connection reuse avoids starting from cold cwnd repeatedly.
```

That is the relationship.

---

## 33. Example: Small JSON API

```text
Client → GET /api/products/123
Server → 2 KB JSON response
```

The response fits in the initial congestion window.

Bottlenecks are probably:

```text
DNS
TLS
server processing
database query
application latency
cold start
auth middleware
serialization
```

`cwnd` is probably not your main issue.

---

## 34. Example: Large File Download

```text
Client → GET /downloads/report.zip
Server → 500 MB response
```

Now the bottlenecks may include:

```text
TCP cwnd growth
RTT
packet loss
bandwidth
CDN location
receiver window
server socket buffer
proxy buffering
```

Here TCP congestion behavior matters a lot.

---

## 35. Example: Microservice-to-Microservice Calls

Suppose Service A calls Service B inside the same AWS region/VPC:

```text
Service A → Service B
```

RTT is very low.

This means:

```text
ACKs return quickly
cwnd grows quickly
latency is low
throughput is usually good
```

Now compare cross-region:

```text
Service A in eu-north-1 → Service B in us-east-1
```

RTT is much higher.

This affects:

```text
latency per request
TLS setup cost
cwnd ramp-up speed
large response throughput
timeout risk
```

That is why cross-region synchronous HTTP calls are dangerous in system design.

For high-performance systems, you usually prefer:

```text
same-region service calls
async messaging across regions
CDN for static content
replication/local reads when needed
```

---

# 36. Practical Advice for Backend and System Design

## 36.1 Reuse Connections

Use connection pooling.

In .NET:

```csharp
services.AddHttpClient("orders", client =>
{
    client.BaseAddress = new Uri("https://orders.internal");
});
```

Avoid creating a new `HttpClient` per request.

---

## 36.2 Keep API Responses Reasonably Small

Do not return huge JSON payloads casually.

Bad:

```text
GET /orders
returns 50 MB JSON
```

Better:

```text
pagination
filtering
compression
streaming
separate file export flow
```

---

## 36.3 Use CDN for Static and Large Content

For:

```text
images
videos
CSS
JavaScript
downloads
public reports
```

Use S3 + CloudFront rather than serving everything from your application containers.

---

## 36.4 Compress Text Responses

Compression helps for:

```text
JSON
HTML
CSS
JavaScript
XML
```

Smaller response means fewer bytes over TCP, less pressure on `cwnd`, and faster transfer.

But do not blindly compress everything.

Already-compressed files usually do not benefit much:

```text
JPEG
PNG
MP4
ZIP
```

---

## 36.5 Avoid Chatty APIs Over High-Latency Links

Bad:

```text
Client makes 50 sequential HTTP calls
Each call waits for the previous one
```

If RTT is 100 ms, even before processing time:

```text
50 × 100 ms = 5 seconds minimum
```

Better:

```text
batching
parallel calls
GraphQL/DataLoader style aggregation
BFF pattern
server-side composition
```

---

## 36.6 Understand HTTP/2 and HTTP/3

HTTP/2 helps by multiplexing requests over fewer connections.

HTTP/3 can help especially on lossy networks because it avoids TCP-level head-of-line blocking between streams.

But backend architecture still matters more than blindly saying:

```text
Use HTTP/3
```

---

# 37. Final Summary

HTTP does not control the congestion window.

HTTP produces messages:

```text
request headers
request body
response headers
response body
```

TCP transports those bytes and controls how fast they move using `cwnd`.

For HTTP over TCP:

```text
Small response:
cwnd usually not a big issue

Large response:
cwnd, RTT, and packet loss matter a lot

New connection:
starts with cold/small cwnd

Reused connection:
can perform better

HTTP/2:
many HTTP streams share one TCP connection and one TCP congestion behavior

HTTP/3:
does not use TCP; uses QUIC with its own congestion control
```

The system-design takeaway:

> **HTTP performance is not only your API code. It is also connection reuse, payload size, RTT, packet loss, TCP congestion control, TLS setup, proxies, load balancers, and CDN placement.**

---

# 38. Brutal Interview-Level Takeaway

If someone asks you how TCP congestion window affects HTTP performance, say this:

> HTTP sends application messages, but TCP decides how fast the bytes can actually move. For HTTP/1.1 and HTTP/2, the response body is transmitted over TCP, so the sender can only have up to `cwnd` bytes in flight before waiting for ACKs. Small JSON responses usually fit inside the initial congestion window, so `cwnd` is rarely the main bottleneck. Large downloads, high RTT, packet loss, and new TCP connections make `cwnd` very important. Connection reuse, HTTP keep-alive, HTTP/2 multiplexing, CDNs, compression, and smaller payloads all help because they reduce setup cost, reduce bytes transferred, or allow the TCP connection to perform more efficiently. HTTP/3 is different because it uses QUIC over UDP instead of TCP, but QUIC still has its own congestion control.
