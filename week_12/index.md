# Network Systems Optimizations 

## Lecture 1: Traditional Networking Optimizations
*CSE 498 — Alex Clevenger, Rishad Islam, Reilly Yankovich*

## Comparison between TCP and UDP
We can use the `iperf3` tool, which operates similarly to the `perf` utility in Linux, to test and monitor the performance of different network protocols and configurations. This tool enables us to compare crucial performance metrics—such as network bandwidth, latency, and jitter—between TCP and UDP. To execute this comparison, we follow these steps:

```bash
$ iperf3 -s # Start the server
$ iperf3 -c <server_ip> # Benchmark TCP from client
$ iperf3 -c <server_ip> -u -b <bandwidth_limit> # Benchmarking UDP from client
```

The TCP benchmark is designed to determine the maximum available throughput (goodput) for that specific network path. For the UDP benchmark, we must explicitly set a bandwidth limit (`-b`), which forces the client to send network packets at the user-specified rate regardless of prevailing network conditions. The server, however, will only process incoming network packets up to its maximum capacity. This allows the benchmark to accurately measure packet loss. In our setup, we performed the benchmark from a local machine to a Lehigh Campus machine, routed over the Lehigh VPN.

<a id="tcp_benchmark"></a>
<p align="center">
  <img src="tcp_benchmark_1.png" width="48%" alt="TCP Server Output">
  <img src="tcp_benchmark_2.png" width="48%" alt="TCP Client Output">
  <br>
  <em>Figure 1: TCP benchmark with iperf3</em>
</p>

Reviewing the TCP benchmark results ([Figure 1](#tcp_benchmark)) shows several expected behaviors of the TCP protocol. The recorded network throughput for the sender was 25.8 Mbps, while the receiver had 24.4 Mbps. This variance between the sending and receiving bitrates occurs because TCP actively probes the network's capacity using congestion control algorithms and occasionally encounters dropped packets. As the output indicates, there were packet losses during transmission; however, TCP's built-in retransmission mechanisms successfully identified these errors and guaranteed reliable, in-order delivery. Consequently, TCP naturally throttled itself to match the Network bandwidth.

<a id="udp_benchmark"></a>
<p align="center">
  <img src="udp_benchmark_1.png" width="48%" alt="TCP Server Output">
  <img src="udp_benchmark_2.png" width="48%" alt="TCP Client Output">
  <br>
  <em>Figure 2: UDP benchmark with iperf3</em>
</p>

For the UDP benchmark ([Figure 2](#udp_benchmark)), our objective was to stress-test the network system to determine if higher throughput could be forcibly achieved. To do this, we aggressively set the bandwidth limit to 64 Mbps. As a result, the client continuously transmitted at a rate of 64 Mbps, even though the established capacity of the network and the server was significantly lower (approximately 25 Mbps, as established by the prior TCP benchmark).

Because UDP lacks flow control and congestion control mechanisms, the client did not scale back its transmission rate. Consequently, the server's receiving rate saturated at 25.3 Mbps. This difference between the sending rate and the link capacity caused network congestion, resulting in a high volume of packet loss during transmission. The server successfully received only 60% of the packets, while the remaining 40% were lost due to buffer overflows at the sender, intermediary VPN routers, or the receiver. Because of the inherently connectionless and unreliable nature of UDP, there is no mechanism to recover these dropped packets.

The key observation from these benchmarks is that instructing a protocol to transmit faster does not bypass physical network constraints. Even if we attempt to achieve higher throughput by flooding the network with UDP packets, the underlying system limitations will not allow us to overcome the bottleneck. TCP elegantly handles this limitation by dynamically adapting its window size to the available bandwidth, whereas UDP simply drops the excess data.

## TCP Protocol Challenges
While TCP guarantees reliable, in-order delivery of data, these strict guarantees inherently introduce performance overhead, making it slower and more resource-intensive compared to UDP. Below, we analyze the reasons why TCP experiences reduced performance:
* **Protocol Overhead and Larger Headers:** To maintain the reliability of TCP, the protocol requires significantly larger packet headers compared to UDP. This additional metadata—which includes sequence numbers, acknowledgment numbers, and checksums—helps the receiver check for transmission errors and coordinate retransmissions if any data is corrupted or lost.
* **Connection Establishment:* Unlike UDP, TCP is a connection-oriented protocol. It requires a three-way handshake (*SYN $\rightarrow$ SYN-ACK $\rightarrow$ ACK*) before any data can be exchanged. This mandatory handshake introduces a delay of at least one full Round Trip Time (RTT) just to open the socket.
* **Head-of-Line (HOL) Blocking and Retransmission:** Because TCP guarantees strict in-order delivery, the loss of a single packet halts the processing of the entire stream. If one packet is dropped, all subsequent successfully received packets are buffered and blocked in the operating system's queue until the lost packet is successfully retransmitted and acknowledged.
* **Security Vulnerabilities and Mitigation Costs:** The stateful nature of TCP and mechanisms like HOL blocking introduce security risks. Malicious actors can exploit these features to launch Distributed Denial of Service (DDoS) attacks or exploit the three-way handshake to launch SYN flood attacks. Implementing mitigations against these attacks consumes additional computational resources and impacts overall network performance.
* **Congestion Control Mechanisms:** TCP continuously monitors the network to avoid collapse. A new connection starts sending data slowly (known as "Slow Start") to probe the network's capacity rather than overwhelming it. If a dropped packet occurs, TCP interprets this as network congestion and aggressively reduces its sending rate, which temporarily reduces throughput.
* **Bandwidth-Delay Product (BDP) Bottlenecks:** Maximum throughput over high-latency, long-distance links is often bottlenecked by the receiver's window size. Unless TCP Window Scaling is properly configured, the sender may be forced to sit idle waiting for acknowledgments before it can transmit more data, preventing the connection from fully utilizing the available bandwidth.
* **Nagle's Algorithm Latency:** To reduce the overhead of sending numerous tiny packets, TCP implements Nagle's algorithm by default. This algorithm buffers small chunks of outgoing data until a larger maximum-sized packet can be formed or an acknowledgment is received. While this improves bandwidth efficiency, it introduces artificial latency that harms real-time applications.
* **Bufferbloat:** In modern networks, excessively large, unmanaged buffers in intermediary routers can trap TCP packets. This phenomenon, known as bufferbloat, causes queuing delays and disrupts TCP's congestion control algorithms, leading to high latency spikes.

## Examples
We have two programs to show the use of TCP. While subsequent sections will detail specific optimizations to improve their performance, these examples establish our baseline. All the experiments were performed in the CSE Sunlab Machines.

Both of our examples use the POSIX socket API to establish reliable TCP communication. The general lifecycle of these TCP applications follows a pattern:
* **Server Setup:** The server creates a socket using the `socket()` system call, binds it to a specific port using `bind()`, and opens it for incoming connections using `listen()`. It then blocks execution at the `accept()` call until a client attempts to connect.
* **Client Setup:** The client similarly creates a `socket()`, but instead of binding, it uses the `connect()` system call to initiate the three-way handshake with the server's IP address and port.
* **Data Exchange:** Once connected, both applications use standard `send()` and `recv()` system calls to push and pull byte streams across the network.

### Bulk Transfer
The objective of a bulk transfer is to push a continuous stream of data from the client to the server. To achieve this, the client allocates a 1 KB buffer and enters a loop, repeatedly calling `send()` until the entire 1 GB target payload is transmitted. Conversely, the server sits in a `recv()` loop, continuously reading the incoming byte stream from the socket buffer until the client gracefully closes the connection.

```cpp
// Basic TCP Bulk Send Loop (Client)
size_t total_chunks = TARGET_DATA_BYTES / TCP_CHUNK_SIZE;

for (size_t i = 0; i < total_chunks; i++) {
  size_t bytes_sent = 0;
  while (bytes_sent < TCP_CHUNK_SIZE) {
    ssize_t result = send(sock, buffer.data() + bytes_sent, TCP_CHUNK_SIZE - bytes_sent, 0);
    if (result <= 0) {
      log_error("Send failed:", errno);
      return;
    }
    bytes_sent += result;
  }
}
```
```cpp
// Basic TCP Bulk Receive Loop (Server)
long total_bytes = 0;
while (true) {
  int bytes_read = recv(client_socket, buffer, sizeof(buffer), 0);
  if (bytes_read == 0) {
    // Client closed connection gracefully
    break;
  } else if (bytes_read < 0) {
    std::cerr << "Receive failed\n";
    break;
  }
  total_bytes += bytes_read;
}
```
<a id="bulk_transfer_1"></a>
<p align="center">
  <img src="tcp_bulk_transfer_server_1.png" width="48%" alt="TCP Server Output">
  <img src="tcp_bulk_transfer_client_1.png" width="48%" alt="TCP Client Output">
  <br>
  <em>Figure 3: Bulk Transfer of 1 GB Data Using TCP</em>
</p>

The initial benchmark successfully transferred 1024 MB of data in approximately 9.19 seconds. This shows a throughput of 934.249 Mbps, which indicates TCP saturates the network bandwidth once the connection is established when transferring a continuous stream of data.

### RPC Workload
Remote Procedure Call (RPC) workload tests the network's latency through a synchronous request-response model. The client sends a small 32-byte struct containing a sequence ID and payload. It then immediately blocks on a recv() call, waiting for the server to process the message and echo a response back. The server reads the incoming struct, modifies it, and uses send() to return it to the client. This interaction happens strictly sequentially.

```cpp
// Basic TCP RPC Loop (Client)
for (uint32_t i = 0; i < TOTAL_TRANSACTIONS; i++) {
  request.sequence_id = i;
  
  // Send the request
  send_request(sock, &request, sizeof(SmallMessage));
  
  // Block and wait for the synchronous response
  recv_request(sock, &response, sizeof(SmallMessage));
  
  // Validate
  if (response.sequence_id != i) {
    std::cerr << "Sequence mismatch!\n";
    return;
  }
}
```
```cpp
// Basic TCP RPC Loop (Server)
while (true) {
  // Block until a request is received
  if (!recv_request(client_socket, &request, sizeof(SmallMessage))) {
    break; // Connection closed or error
  }

  // Process the data
  response.sequence_id = request.sequence_id;
  memset(response.payload_data, 1, sizeof(response.payload_data));
  
  // Send the response back immediately
  if (!send_request(client_socket, &response, sizeof(SmallMessage))) {
    break;
  }
}
```
<a id="rpc_1"></a>
<p align="center">
  <img src="tcp_rpc_client_1.png" width="80%" alt="TCP Client Output">
  <br>
  <em>Figure 3: Bulk Transfer of 1 GB Data Using TCP</em>
</p>

The benchmark executed 100,000 transactions in 17.2006 seconds. The system achieved a throughput of 5813.75 Transactions Per Second (TPS) with an average latency of 0.172006 ms per round-trip. Because each transaction requires a full round-trip across the network before the next can begin, this workload is heavily bottlenecked by the inherent latency of the TCP protocol and OS kernel processing, rather than raw bandwidth.

## Background and Motivation

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
