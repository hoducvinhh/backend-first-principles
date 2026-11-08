# Backend From First Principles 🚀

> Understanding backend engineering and distributed systems by decomposing them into their fundamental truths and building upwards.

## 📌 Overview

This repository is dedicated to mastering backend engineering from **First Principles** — diving deep into low-level mechanics, system architecture, database internals, networking protocols, concurrency, and security, rather than merely using high-level frameworks.

---

## 🗺️ Core Pillars & Roadmap

### 1. 🛡️ Backend Security
- **Cryptography & Hashing:** Symmetric/Asymmetric encryption (AES, RSA, ECC), Salted Hashing (Argon2, bcrypt).
- **Identity & Access Management:** OAuth 2.0, OpenID Connect (OIDC), JWT, Session-based auth, RBAC/ABAC.
- **Web & API Defense:** OWASP Top 10, SQLi, XSS, CSRF, SSRF, IDOR, Rate Limiting, DDoS mitigation.
- **Data Protection:** Encryption at rest & in transit, TLS/SSL termination, Secrets management.

### 2. 🌐 Networking & Protocols
- **Transport & Sockets:** TCP/IP stack, UDP, Socket programming, Handshakes & packet loss.
- **Application Protocols:** HTTP/1.1, HTTP/2 (Multiplexing), HTTP/3 (QUIC), WebSockets, gRPC / Protobuf.
- **DNS & Routing:** DNS resolution, Anycast, Reverse Proxies (Nginx, Envoy), Load Balancers (L4 vs. L7).

### 3. ⚡ Operating Systems & Concurrency
- **Execution Units:** Processes, Kernel Threads, User Threads (Goroutines, Green Threads, Fibers).
- **Concurrency & Synchronization:** Race conditions, Deadlocks, Mutex, Semaphores, Lock-free data structures.
- **I/O Models:** Blocking vs Non-blocking, I/O Multiplexing (`select`, `poll`, `epoll`, `kqueue`, `io_uring`), Event Loops.

### 4. 💾 Storage Engines & Databases
- **Underlying Data Structures:** B-Trees, B+ Trees, Log-Structured Merge (LSM) Trees.
- **Reliability & Consistency:** Write-Ahead Log (WAL), Buffer pools, ACID guarantees, Isolation levels.
- **Scaling:** Partitioning / Sharding, Read Replicas, Multi-Master replication, Distributed Transactions (2PC).

### 5. 🏗️ Distributed Systems & High Availability
- **Foundations:** CAP Theorem, PACELC, Fallacies of Distributed Computing.
- **Consensus Algorithms:** Paxos, Raft, Vector Clocks.
- **Message Brokers & Event Streaming:** Kafka, RabbitMQ, Event-driven architectures, Idempotency.
- **Resilience Patterns:** Circuit Breaker, Bulkhead, Retry with exponential backoff, Dead Letter Queues.

---

## 📂 Repository Structure

```text
backend-first-principles/
├── Backend Security/
│   └── note.md
├── README.md
└── .gitignore
```

---

## 📝 License

Distributed under the MIT License.
