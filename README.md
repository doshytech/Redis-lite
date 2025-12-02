## 🚀 Project: Redis-Lite

### Introduction

This project is a high-performance, multithreaded in-memory key-value store built from scratch in C++. It is designed to be protocol-compatible with Redis clients (RESP), making it a drop-in replacement for specific caching and queuing operations.

The primary goal of this application is to demonstrate advanced systems programming concepts, including socket networking, multithreading, thread synchronization, and protocol parsing, while providing a fast, persistent storage engine for low-latency data access.

---

### Table of Contents

* Introduction
* Architecture Overview
* Use Cases
* Technical Implementation
* Setup & Usage
* Future Enhancements

---

### Architecture Overview

The system is architected as a concurrent TCP server that decouples connection handling, command parsing, and data manipulation.

#### High-Level Design (HLD)

The architecture follows a Client-Server model with a thread-per-client approach for handling concurrent connections.

```
graph TD
    Client[Client Application / redis-cli] -->|TCP Connection| NetLayer[Networking Layer]
    NetLayer -->|Spawn Thread| Handler[Command Handler]
    Handler -->|Parse RESP| Parser[Protocol Parser]
    Parser -->|Execute| DB[(In-Memory Database Singleton)]
    
    subgraph Core Engine
    DB -->|Read/Write| KV[KV Store]
    DB -->|Read/Write| List[List Store]
    DB -->|Read/Write| Hash[Hash Store]
    DB -->|Auto-Evict| TTL[Expiry Manager]
    end
    
    Background[Background Thread] -->|Snapshot every 5m| Disk[Persistence File .rdb]
    DB -.->|Load on Start| Disk
```

---

### Low-Level Design (LLD)

The application is modularized into three core components:

**Server Module (RedisServer):**

* Responsibility: Manages the lifecycle of the TCP socket server. It listens on a configurable port (default 6379) and accepts incoming connections.
* Concurrency: Utilizes `std::thread` to spawn a dedicated execution context for every connected client, ensuring that one slow client does not block the acceptance of others.
* Signal Handling: Implements graceful shutdown procedures (SIGINT) to ensure data is saved to disk before the process exits.

**Command Handling (RedisCommandHandler):**

* Responsibility: Acts as the controller. It receives raw byte streams from the socket, parses them using a custom RESP parser (handling both inline commands and array-based protocols), and routes them to the appropriate database function.

**Database Engine (RedisDatabase):**

* Pattern: Implemented as a thread-safe Singleton.
* Storage: Uses `std::unordered_map` for O(1) average time complexity lookups for strings, lists, and hashes.
* Synchronization: A global `std::mutex` (specifically `db_mutex`) protects shared memory resources, preventing race conditions during concurrent reads/writes.
* Persistence: A background thread triggers a snapshot of the memory state to a file (`dump.my_rdb`) at fixed intervals, ensuring durability.

---

### Use Cases

This system is designed for scenarios requiring sub-millisecond response times:

* **Session Management:**

  * *Scenario:* A web application needs to track user login states.
  * *Implementation:* Use `SET key value` with `EXPIRE` to store session tokens that automatically invalidate after a specific time.

* **Job Queues:**

  * *Scenario:* A background worker system processing video uploads.
  * *Implementation:* Use `LPUSH` to add jobs to a queue and `RPOP` to consume them, allowing for decoupled producer-consumer architectures.

* **Real-time Analytics:**

  * *Scenario:* Tracking live page views or API usage limits.
  * *Implementation:* Use atomic `INCR` (simulated via set/get) or Hash Maps to aggregate counters in real-time.

* **User Object Caching:**

  * *Scenario:* Storing user profiles to avoid hitting a slow SQL database.
  * *Implementation:* Use `HSET` and `HGETALL` to store structured data like Name, Email, and Age grouped by UserID.

---

### Technical Implementation

**Tech Stack**

* Language: C++ (C++17 Standard)
* Build System: GNU Make
* Networking: BSD Sockets (`sys/socket.h`, `netinet/in.h`)
* Protocol: Redis Serialization Protocol (RESP)

**Key Engineering Decisions**

* **Thread-Safe Singleton:** The database instance is a Singleton, ensuring a single source of truth for the data. Access is guarded by `std::lock_guard` and `std::mutex`, ensuring that despite multi-threaded client handling, data integrity is never compromised.
* **Lazy Expiration Strategy:** Instead of running an expensive timer for every key, the system employs a "lazy" strategy. Keys are checked for expiration only when they are accessed. (Note: A dedicated active purger could be added for optimization).
* **Custom Protocol Parser:** A robust parser was written to handle the RESP protocol manually. This allows the server to interface with any standard Redis client (like redis-cli, Python redis, or Node.js ioredis) natively.
* **Persistence Mechanism:** The system implements snapshotting. It serializes the in-memory STL containers into a custom binary-safe text format. This provides crash recovery capabilities by reloading the dump file upon restart.

---

### Setup & Usage

**Prerequisites**

* GCC/Clang compiler supporting C++17.
* Make.
* (Optional) `redis-cli` for testing.

**Installation**

Build the project:

```bash
make
```

Clean build (if necessary):

```bash
make clean
```

**Running the Server**

Start the server on the default port (6379):

```bash
./my_redis_server
```

Or specify a custom port:

```bash
./my_redis_server 6380
```

**Connecting via Client**

You can interact with the server using the standard `redis-cli`:

```bash
redis-cli -p 6379
PING
# Output: PONG

SET mykey "Hello C++"
# Output: OK

GET mykey
# Output: "Hello C++"
```

---

### Future Enhancements

To scale this project for production-grade loads (C10K problem), the following improvements are planned:

* **Event Loop (Epoll/Kqueue):** Moving from a "thread-per-client" model to a non-blocking I/O model using epoll (Linux) or kqueue (macOS). This would significantly reduce context-switching overhead and allow handling thousands of concurrent connections with a small thread pool.
* **Granular Locking:** Replacing the global mutex with Segmented Locking (sharding the key space). This would allow multiple threads to write to different parts of the database simultaneously, increasing write throughput.
* **Append Only File (AOF):** Implementing an AOF log to record every write operation. This offers better durability guarantees compared to the current snapshotting mechanism, which can lose data between dump intervals.
* **LRU Eviction Policy:** Implementing a Least Recently Used eviction policy to cap memory usage, ensuring the server doesn't crash when RAM is full.

---
