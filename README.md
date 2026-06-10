# 👋 Hi, I'm Rajveer Yadav

🚀 **Founding Backend Engineer @ AlgoTest (YC S22) · AI Researcher · Systems Developer**
🎓 **B.S. Chemistry, IIT Bombay**
💡 I build things at the intersection of low-latency systems, quantitative infrastructure, and applied AI — from exchange-grade HFT feed handlers to deep learning research.

---

### 💼 Experience

**Founding Backend Engineer — [AlgoTest](https://algotest.in) (YC S22)** *(Nov 2025 – Present)*

AlgoTest is a YC-backed algorithmic trading platform. I joined as a founding engineer to build the core backend from scratch — the plumbing that lets retail and prop traders wire up their brokerage accounts and execute strategies directly through the platform.

The core challenge was building a unified execution layer across multiple broker APIs, each with wildly different semantics, rate limits, and failure modes. I designed an **event-driven pub/sub pipeline** for market data ingestion and signal routing, where order placement, modification, and cancellation all flow through an async execution layer with real-time portfolio state updates. Reliability under broker failures and network partitions was a first-class design constraint.

---

### ⚡ HFT & Quant Infrastructure

A suite of components that together form a production-style exchange data pipeline — from raw multicast frames off the wire to typed messages on an internal bus, ready for a strategy engine to consume.

```
  [itch-multicast]          [itch-cpp]              [mbus]               [strategy]
  MoldUDP64 generator  →   ITCH 5.0 parser   →   Disruptor ring   →   Av-Stoikov / risk
  + gap detection           4.9 ns/msg            sub-5µs p99
```

---

**[Alta-Bus](https://github.com/ray-27/Alta-Bus) — Lock-Free Message Bus** *(Rust)*

Most pub/sub systems (Redis: ~150–300µs p99, NATS: ~50–150µs p99) are general-purpose and pay the cost of userland sockets on every message. `mbus` is purpose-built for the HFT hot path.

The design is a **LMAX Disruptor-style shared ring buffer**: the producer writes once to a single shared array; each subscriber maintains its own independent atomic read cursor and reads directly — no per-subscriber buffers, no central dispatch thread, no locks anywhere on the hot path. Backpressure is handled at the slot level: a producer checks the minimum consumer cursor before overwriting, so a slow downstream consumer stalls the producer rather than silently corrupting state.

Every slot and every cursor struct is `#[repr(C, align(64))]` — cache-line padded to prevent false sharing, which is the silent latency killer in multi-consumer ring designs. The memory ordering is deliberate: `Release` on the slot sequence commit, `Acquire` on the consumer spin — the minimum ordering that gives correct visibility without unnecessary fence overhead. The result is **sub-5µs p99** end-to-end on a single host. Getting below 1µs would require kernel-bypass (DPDK / Solarflare OpenOnload), which is on the roadmap.

The canonical slot schema (`PriceTick`, `OrderBookDelta` — fixed-point `i64` fields, never `f64`) acts as the lingua franca of the whole pipeline. Everything upstream is protocol-specific; everything downstream is protocol-agnostic.

---

**[ITCH-cpp](https://github.com/ray-27/ITCH-Parser) — NASDAQ ITCH 5.0 Feed Parser** *(C++)*

NASDAQ distributes its full order book via the TotalView-ITCH 5.0 binary protocol over multicast. `itch-cpp` is a **zero-copy, header-only C++ library** that deserializes raw ITCH binary streams into typed structs — fast enough to keep up with the full NASDAQ feed.

The design decision was library, not service: a service boundary would force a serialization round-trip on every message, adding 2–5µs of latency for zero benefit. Instead, the parser is a pure transformation kernel — no threads, no locks, no allocator, no network stack — designed to sit inline inside a feed handler process, calling directly into `mbus`.

All 23 ITCH 5.0 message types are implemented as `std::variant`-based structs with `StrongType<T,Tag>` wrappers for all numeric fields (compile-time domain safety, zero runtime cost). A `SoupBinReader` handles the SoupBinTCP framing layer on top of ITCH — including a non-obvious protocol detail: the SoupBinTCP frame includes a 1-byte packet-type prefix *before* the ITCH message type byte, which caused a subtle off-by-one that made 98% of messages parse as `OrderReplace` until caught and fixed. After the fix, message distribution matched expected NASDAQ proportions (~44% `AddOrder`, ~43% `OrderDelete`).

Benchmark on a 1 GB ITCH file: **205M msg/s at 4.9 ns/msg** (warm cache). Cold-cache on an 11 GB file drops to ~20M msg/s due to page faults — addressed with `MADV_WILLNEED` and multi-pass benchmarking that reports best/avg/worst.

---

**[ITCH-Multicast](https://github.com/ray-27/ITCH-multicaster) — MoldUDP64 Multicast Simulator** *(C++)*

NASDAQ's actual on-the-wire transport is MoldUDP64 — a UDP multicast protocol with sequenced framing and a separate TCP retransmission channel for gap recovery. `itch-multicast` implements this protocol from scratch to give the full pipeline a realistic upstream source.

The sender generates synthetic ITCH messages, wraps them in MoldUDP64 frames with big-endian sequence numbers and 2-byte message length prefixes (matching the real spec byte-for-byte), and blasts them over a multicast group. The receiver reassembles the stream and tracks sequence continuity — if a gap is detected (dropped packet or out-of-order delivery), it flags the missing range and can request retransmission over the TCP gap-fill channel. This is the detail that separates a toy from real infrastructure: in production, packet loss handling is where most feed handler bugs live.

The two SPSC rings inside the pipeline — one for raw JSON frames (~8.2 KB/slot, 1024 slots), one for encoded binary packets (~1.5 KB/slot, 4096 slots) — are sized around worst-case Coinbase WebSocket snapshot bursts, with slot counts chosen to cover burst depth without wasting L3 cache.

---

### 🔬 Research

**Visual Haystack & Retrieval** *(Jan 2025 – Jul 2025)*
*Guide: Prof. Ganesh Ramakrishnan, CS Dept., IIT Bombay*
Multi-image question answering — retrieval algorithms, similarity search, and cross-image reasoning over large visual datasets.

**Medical Image Super-Resolution** *(Aug 2023 – Nov 2023)*
*Guide: Prof. Amit Sethi, MEDAL Lab, EE Dept., IIT Bombay*
Resolution enhancement on DIV2K and BreastHis using Vision Transformer, SwinIR, SRGAN, Real-ESRGAN, and SinGAN.
### 🚀 Other Projects

---

**🔍 Raydeon — AI Data Analysis Agent** *(Closed Source)*
Natural language interface over large CSV/Excel datasets — query and visualize data without writing SQL. Built with **Go + Gin + Langraph + Gemini**; stream-ingests directly into **ClickHouse** and **PostgreSQL** without writing temp files to disk.

**🩺 AI Medical History Analyzer** *(Open Source)*
Extracts structured patient data from unorganized medical documents using OCR, then lets doctors query records in natural language. Built with **Python + Flask**, deployed on AWS.

**💬 [rayChats — Real-Time Chat Server](https://github.com/ray-27/rayChats)** *(Open Source)*
WebSocket-based chat backend in **Go + Gin** with **Redis** session management, built for high concurrency and low-latency messaging.

---

### 🛠 Skills

| | |
|---|---|
| **Languages** | Go, Rust, C++, Python, TypeScript |
| **Low-Latency / Systems** | Disruptor ring buffers, SPSC lock-free queues, atomic memory ordering, UDP multicast, ITCH 5.0, MoldUDP64, SoupBinTCP |
| **AI / ML** | PyTorch, GANs, Diffusion Models, Transformers, Langchain, Langraph |
| **Backend** | Gin, WebSockets, pub/sub, async pipelines, event-driven architecture |
| **Databases** | PostgreSQL, ClickHouse, Redis, MongoDB |
| **Cloud / DevOps** | AWS (EC2, Lambda, Route53, CodeDeploy), Cloudflare, CI/CD |

---

### 📫 Connect

[![LinkedIn](https://img.shields.io/badge/LinkedIn-blue?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/rajveeryadav27/)
[![GitHub](https://img.shields.io/badge/GitHub-black?style=for-the-badge&logo=github&logoColor=white)](https://github.com/ray-27)
[![Email](https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:rajveeryadav2711@gmail.com)
