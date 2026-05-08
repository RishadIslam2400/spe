# Network Systems Optimizations 

## Lecture 1: Traditional Networking Optimizations

Background and Motivation

TODO

## Lecture 2: Remote Direct Memory Access (RDMA)

*CSE 498 — Alex Clevenger, Rishad Islam, Reilly Yankovich*

---

## What is RDMA?

RDMA (Remote Direct Memory Access) enables direct memory access between machines over a network, bypassing the OS and CPU on the remote side. Setting up RDMA connections is non-trivial and requires careful orchestration of multiple steps:

- Establish a protection domain
- Register memory regions
- Allocate Completion Queues (CQs)
- Create Queue Pairs (QPs)
- Exchange addresses and rkeys
- Transition QP states

Hardware-enforced ordering exists within a single QP, but not across different QPs. RDMA memory must be **pinned** (registered with the NIC) so that the NIC can safely DMA to/from it without the OS paging it out.

This complexity motivates RDMA libraries such as **Remus**, which streamline the setup process.

---

## Transport Types: RC vs. UD

### Reliable Connection (RC)

Analogous to TCP. RC provides:

- **One-to-one** connection semantics
- Support for one-sided operations (READ, WRITE, CAS)
- Per-connection state tracked by the rNIC, which creates a practical constraint on available NIC resources:
  - **ICM (Interconnect Context Memory):** stores QP context and other control structures
  - **MTT (Memory Translation Table) cache:** holds address translations for registered RDMA memory regions

### Unreliable Datagram (UD)

Analogous to UDP. UD provides:

- **One-to-many** semantics
- Lighter connection overhead
- Best-effort delivery with no ordering guarantees
- Limited to Send/Recv — **does not support one-sided operations**

---

## Operation Types: One-sided vs. Two-sided

### One-sided Operations

The remote CPU is entirely uninvolved — zero CPU cycles are consumed on the remote side.

- Supports: **Read, Write, CAS**
- Requires **Reliable Connected (RC)** transport
- Remote memory must be pre-registered with known addresses and rkeys

### Two-sided Operations

The remote CPU is actively involved.

- The receiver must post a Receive Work Request (RWR) to its Receive Queue (RQ) ahead of time
- The receiver must also poll its CQ to know when a SEND has arrived
- Works with both RC and UD
- The receiver does not need to know the sender's memory layout — no remote address specification required

---

## Zero-Copy Operations

A key consideration in RDMA design is avoiding unnecessary data copies. Consider the following two approaches:

```cpp
// Approach 1: Copy into RDMA buffer inside Write()
T obj(some_data);
compute_thread->Write(laddr, raddr, obj, wr_id);

// Approach 2: Write directly into registered buffer, then issue RDMA
*reinterpret_cast<T *>(laddr.addr + laddr.offset) = obj;
compute_thread->Write(laddr, raddr, wr_id);
```

Approach 2 is a **zero-copy** pattern — the object is written directly into the pre-registered memory region, avoiding an intermediate copy. This reduces memory bandwidth and latency.

---

## Thread-to-QP Relationships

Since ordering is enforced per-QP, the mapping between threads and QPs is an important design decision. Common patterns include:

- **One-to-One:** each thread owns a dedicated QP
- **Round-robin:** threads distribute work across QPs
- **Random:** threads pick QPs randomly
- **QP-sharing:** multiple threads share a QP — requires explicit coordination and is likely to introduce synchronization overhead

---

## System Design & Tunable Parameters

### Memory Disaggregation

Disaggregating memory is particularly well-suited for one-sided operations. Since one-sided RDMA does not involve the remote CPU, a memory node can use a cheap CPU without sacrificing performance.

### Staging Buffers

The size of staging buffers and landing space is a key tunable. These buffers must be RDMA-registered because the NIC needs stable, pinned physical addresses for DMA.

---

## RDMA Optimizations

### Completion Batching

Rather than polling for completions after every individual operation, completions can be batched — reducing the overhead of CQ polling and increasing throughput.

### Doorbell Batching

Multiple work requests can be posted before ringing the NIC's doorbell, reducing the number of costly MMIO writes to the NIC.

### Shared Completion Queue (CQ)

Multiple QPs can share a single CQ, reducing resource consumption and simplifying polling. A potential adverse effect is that a single slow QP can delay processing of completions from faster QPs.

### Memory & Layout Optimizations

- **Huge pages:** drastically reduce TLB misses for registered memory regions
- **Cache-aligned staging buffers:** improve CPU cache efficiency
- **Prefetching:** reduce cache miss latency in the data path
- **Inlining payloads:** small payloads can be inlined into the WQE, avoiding an additional DMA fetch from host memory

### Other Optimizations

- **Relax ordering:** eliminate `IBV_SEND_FENCE` where strict ordering is not required
- **Eliminate signaling for WRITEs:** suppress completion events for operations that don't need them, reducing CQ traffic
- **Arm the CQ (`ibv_req_notify_cq`):** switch to an event-driven model instead of busy-polling, freeing CPU cycles

---

## Remus

Remus is an RDMA library developed and maintained by the **Scalable Systems and Software (SSS) group at Lehigh University**. Its objectives are:

- **Streamline connection setup** — abstracting the multi-step RDMA initialization process
- **Programmability** — providing a clean, usable API for RDMA-based systems
- **Performance** — enabling low-latency, high-throughput distributed applications

---

## Ongoing Work: Fibers

A fiber is a lightweight unit of execution that runs entirely in user space, without OS involvement. Compared to traditional threads:

| | Threads | Fibers |
|---|---|---|
| Context switch cost | 1–5 µs | 10–100 ns |
| Scheduling | OS-managed | User-managed |
| Overhead | Higher | Much lower |

Fibers are being explored in ongoing work to further reduce per-operation overhead in RDMA-based systems by replacing OS-scheduled threads with cooperative, user-space switching.

---

## Citations

- Slides 3–6, 19–20 by Amanda Baran, SPAA '25
- Slide 29: [Demystifying RDMA — LinkedIn post by Ravichandran Paramasivam](https://www.linkedin.com/posts/ravichandran-paramasivam-a12b3438_demystifying-rdma-from-sockets-to-zero-copy-share-7394351876416143360-UnQK/)