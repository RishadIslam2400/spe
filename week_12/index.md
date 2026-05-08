# Network Systems Optimizations 

## Lecture 1: Traditional Networking Optimizations
*CSE 498 — Alex Clevenger, Rishad Islam, Reilly Yankovich*

## Definition: Network protocols
Network protocols are an established set of rules for computers to communicate and transfer data with each other. Different layers of a network use different protocols to communicate; for example, the protocol for communication between computers over the internet is different than within a server farm. Here, we will focus on two specific protocols: User Datagram Protocol, or UDP, and Transmission Control Protocol, or TCP. We will also briefly discuss some general optimizations when networking.

## General optimizations
In general, when programming over a network, you want to *properly saturate* the network cards. As in, you want to be sending enough messages that you’re efficiently using the networking capabilities of your network, but not sending too many messages such that you’re exceeding the capabilities. If you send too few messages, not efficiently using the capabilities of your network, that’s called *under-saturating*. If you send too many messages and overwhelm your network, that’s called *over-saturating*. Typically, in any properly large network, the bigger problem is over-saturating, because you can always send more messages if you’re under-saturating, so we will focus on that side of general optimizations.

* **Send fewer messages:** the more messages that have to be exchanged between systems, the more time is spent both sending and processing messages, and the less time is spent actually doing work. By batching numerous smaller messages together into one larger message, you are able to transfer the same amount of data in fewer messages, thereby spending less time networking. All networks have a *Maximum Transmission Unit,* or MTU, which is the maximum number of bytes that can be transmitted with a single message. Over Ethernet, this is typically **1500 bytes**, and over IPV6 it’s typically **1280 bytes.** If you have a message that’s only 20 bytes long, that means you can fit several dozen into a single MTU by batching, without any modifications made to the network!
* **Think about your topology:** Networks are generally organized into a *topology,* which is how connections are made and organized between systems. One example of this is a *ring topology:* every system has one receiving connection and one sending connection. The final network connects to the first network, causing a “ring” to form: system 1 sends to system 2, which sends to system 3, which sends back to system 1.
There are several different topologies commonly used, sometimes in conjunction with each other to form a hybrid topology, and they all have their pros and cons; continuing with the ring topology example, there are very few connections that have to be formed, but this results in messages having to go around the ring. If system 2 only sends to system 3, but it needs a message to get to system 1, it has to go *all the way around the ring* to get back to system 1.
All this to say, if you don’t plan your message passing around your topology, you could be doing a lot more work than necessary. If you have a ring topology, it may be far more efficient to batch several messages going to several different systems to avoid passing through the ring multiple times. In a *mesh topology,* where every system is connected to every other system, it makes more sense to batch several messages going to the same system, because any system is able to transmit to any other system.
* **Do load balancing:** If one specific system is overloaded in a network, it could slow down every other system in the network. This is because the longer that one system takes to perform tasks, the longer it takes for all other systems to receive confirmation from that system for *their* remote tasks. This is especially true in a ring or tree topology, for example: if one system is overloaded, no messages are getting past that system until it can catch up.
To avoid this, split up the network traffic between systems. If you have a database, replicating data across different systems means that you can route traffic to different replicas, alleviating pressure. Forming a hybrid topology could reduce network traffic by allowing a second path to reach a specific destination (in a ring topology, for example, just having connections going the other direction can allow routing around a stalled/overloaded system). This can be taken further by applying *network segmentation:* have several sub-networks that are connected with one topology, and then those sub-networks are connected to other sub-networks with a separate, possibly identical, topology.  Rings within rings, rings within mesh, and so on. This allows a network to control traffic further than a simple topology.
Finally, analyzing your network traffic will show which systems need to be further alleviated, and if they always need to be alleviated. For example, in a multiplayer video game, you may expect less traffic during the average workday and more traffic during the evening. The servers for that game would need more resources dedicated to it during the evening, but during the day they might be used for some other purposes.

## UDP
User Datagram Protocol, or UDP, is a lightweight network protocol designed to have minimal overhead. There are no headers attached to messages, which are used by TCP to establish the order of packets; if a message in UDP spans multiple packets, those packets can be delivered out of order, delivered partially, or not delivered at all. This is especially true if two messages are sent at the same time. No connections are formally established when using UDP, meaning that no acknowledgements are sent back to you when sending a UDP packet. When using UDP, you don’t actually know when, or even if, a message gets delivered to the remote machine.
With these drawbacks, why would you ever want to use UDP? Well, since UDP has minimal overhead, it is ideal for cases where **speed is more important than correctness or reliability.** UDP is used for things such as video streaming, VPNs, or unreliable broadcasts in distributed systems. In all of these cases, a single dropped or corrupted packet here or there won’t affect the overall system; either that packet can be completely ignored, or it can be retrieved faster if it’s really important to include.

## UDP Example
Now, we will show a very simple example of using UDP. This example will open a server and client, and allow you to send messages from the client to the server.
For this example, you will want a terminal with netcat installed. Netcat is installed by default on all linux systems, so WSL and MACs should also have them installed by default.
If using two separate systems, then you will need to know the IP of at least one of the systems. Otherwise, you can use two terminals on one system, and use a *loopback IP.* A loopback IP is simply an IP that is used for a machine to talk to itself.
* **Step 1:** if you need the IP of the second machine, open a terminal and use `ifconfig` to find the IP, under “inet.” You can also do this to find a loopback IP if needed. **All IPs between 127.0.0.1 and 128.0.0.0 are enabled loopback IPs by default,** and these should work.

<a id="ifconfig_example"></a>
<p align="center">
  <img src="udp_example_1.png" width="48%" alt="An example of using ifconfig">
  <br>
  <em>Using ifconfig to find an IP. "inet" is what you're looking for.</em>
</p>

* **Step 2:** enter `nc` to ensure you have netcat installed. If you get a “usage” prompt like what is pictured below, you’re good.

<a id="nc_example"></a>
<p align="center">
  <img src="udp_example_2.png" width="48%" alt="seeing if netcat is installed">
  <br>
  <em>Seeing if netcat is installed.</em>
</p

* **Step 3:** open a second terminal. If using 2 separate systems, you will want a terminal on each; if using loopback, then 2 terminals on the same machine will work.

* **Step 4:** pick one of the terminals to be the server. On this server side, enter:

```bash
$ nc -u -l <port> # choose a port
```

For port, any number should ideally work, but you might want to stick to a 4 digit number just in case. For example, 1234.
The flag `-u` is telling netcat to use UDP protocols, and the flag `-l` is telling netcat to listen for anything happening on the port we enter. So, all together, `start netcat using UDP, and listen on port <port>.`

* **Step 5:** On the other terminal, enter:

```bash
$ nc -u <ip> <port> # same port
```

Now, we’re telling our client to `start netcat using UDP, and send any following messages to <ip> over port <port>.`

* **Step 6:** On the client side, type whatever you want and hit enter. You should see that mesasge pop up on the server side.

**Congratulations!** You've now used UDP! Try sending extremely long messages, and seeing if everything actually gets sent. Over loopback this is pretty likely, but over an actual network packets stand a greater chance of dropping.

## UDP Optimizations
So, now that we’ve used UDP and know what to use it for, how can we improve it? The two broad categories of optimizations we can make to UDP are OS tweaks and smarter programming. If you’re using someone else’s network, you may be unable to perform the OS tweaks, but you can always program smarter!

* **OS tweaks:** Here, there are three optimizations we wish to highlight. More exist, but since they are not OS-specific, we only want to mention a few. These tweaks are all fitting into our general goal of properly saturating the network card.
  * There are two OS buffers that are used by UDP in the linux kernel: Receiving Memory, or RMEM, and Writing Memory, or WMEM. The RMEM buffer stores packets that have been received by the system, but have not yet been used by any application. WMEM is the opposite; it stores packets that have been written by an application, but have not yet been sent over a network. By adjusting these buffer sizes, the OS has more room to store packets before it has to drop packets.
  * When opening a socket for networking applications, you can set the socket to be *non-blocking.* This allows for packets to be transferred faster. If a system is already experiencing too much traffic, it might be beneficial to not do this, but having non-blocking packets is generally a good thing for UDP.
  * You can enable *packet aggregation,* which allows the kernel to automatically join several smaller packets into one transmission unit. This is like batching mentioned before, but done automatically by your OS.
* **Smarter Programming:** Smarter programming, in this case, means to remember the MTU, and to properly saturate the network card.
  * Since UDP doesn’t guarantee that all packets will be sent in the correct order, we want to minimize the chance that there’s an error. If we send a message smaller than the MTU, then the message will either be fully delivered or not delivered at all; packets won’t be sent in the wrong order, nor will a message be partially sent.
  * If you are unable to enable packet aggregation in your network, but you’re sending several small messages, you can just batch messages yourself. This has been mentioned before for the purpose of not over-saturating the network card, but in UDP, where messages aren’t guaranteed to all be sent, this is especially important. While this means that losing that single message is more costly, sending a single, batched message lowers the chance of losing anything compared to several unbatched messages.
  * On the receiving end of UDP messages (such as a server), parallelizing your message handling is a great way to improve your system. Parallelizing allows you to process messages faster, meaning there’s a lower chance of over-saturating the network card.

## TCP
TCP is the more reliable sibling of UDP. When using TCP, a connection between the two servers is formally established. This connection, and all following messages, use the 3-way handshake of TCP: Send, acknowledge, and second acknowledge. With this 3-way handshake, when a message is sent, an acknowledgement is sent back to let the sender know that the message is properly received, and the second acknowledgement means that both sides now know that a message was properly transferred. If either side does not receive its acknowledgement, it will re-send its piece of the puzzle; if the sender doesn't receive the first acknowledgement, it will *re-send the message,* and if the receiver doesn't receive the second acknowledgement, it will *re-send its acknowledgement.*
**This is the key difference between TCP and UDP:** with UDP, we don't know if a message was ever sent, much less if it was sent correctly; with TCP, *we make sure it was sent correctly.* In order for the receiver to know that a message was fully and correctly received, a message's packets include a header, which shows which message it is a part of, which "number" packet it is (for example, 1 of 4), and some other flags and checksums to ensure that the packet was not corrupted. With these, the receiver is able to see if any packets are missing (for example, if you have a message spanning 4 packets, and you have packets 1, 2, and 4, you know you're missing 3), use the checksum to check for corruption, and separate two message's packets in order to not mix them up.
In a linux-based system, using TCP is treated the same as reading and writing locally to a file on disk, except the file is a *socket.* Otherwise, it is basically identical from the user's side, including an expensive context switch to kernel space; the only difference is the operating system writes to the buffers in the network card instead of to an actual file. Although opening a socket is more complex than opening a file, this still allows for a very programmable, intuitive way to network, without having to deal with all the messiness of *actually* dealing with the network card.
With the 3-way handshake causing any message to be tripled at the minimum (send the message, receive the acknowledgement of the message, and send a second acknowledgement), TCP is, understandably, *slower than UDP.* Therefore, it is commonly used in cases where **correctness or reliability are more important than speed.** Some examples of these cases include SSH, file transfers, and web browsing.

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
  <em>Figure 4: 100K RPC Workload Using TCP</em>
</p>

The benchmark executed 100,000 transactions in 17.2006 seconds. The system achieved a throughput of 5813.75 Transactions Per Second (TPS) with an average latency of 0.172006 ms per round-trip. Because each transaction requires a full round-trip across the network before the next can begin, this workload is heavily bottlenecked by the inherent latency of the TCP protocol and OS kernel processing, rather than raw bandwidth.

## TCP Optimizations
While TCP provides reliable delivery, its default configuration is tuned for general-purpose internet traffic, prioritizing bandwidth conservation and fairness over extreme low latency or maximum throughput. By adjusting specific socket options and application-level behaviors, we can optimize TCP for high-performance network applications.

### Socket Options for Low Latency
* **`TCP_NODELAY` (Disabling Nagle's Algorithm):** Nagle's algorithm is a congestion control mechanism which bundles multiple small, outgoing data packets into a single, larger packet before sending. While efficient for bandwidth, it introduces latency by delaying transmission  until a full packet is formed or an acknowledgment is received. For real-world applications requiring immediate data dispatch this delay is counterproductive. We can disable Nagle's algorithm for the client using the TCP_NODELAY socket option to ensure packets are transmitted immediately.

```cpp
// Optimization: Disable Nagle's algorithm on the sending socket
int opt_nodelay = 1;
if (setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &opt_nodelay, sizeof(opt_nodelay)) < 0) {
  log_error("setsockopt(TCP_NODELAY) failed:", errno);
}
```

* **`TCP_QUICKACK` (Disabling Delayed Acknowledgments):** By default, TCP delays sending an acknowledgment (ACK) for up to 40-500 milliseconds, attempting to piggyback the ACK onto an outgoing data packet. To combat this, we disable delayed acknowledgment for the server, ensuring ACKs are sent immediately. This directly reduces response time  in request-response loops. Note that on Linux systems, `TCP_QUICKACK` is not permanent and must be re-applied to the socket after subsequent read operations.

```cpp
// Optimization: Disable delayed ACKs for the incoming packet
// This must be set on the active client_socket, and reused after every read
int quickack = 1;
setsockopt(client_socket, IPPROTO_TCP, TCP_QUICKACK, &quickack, sizeof(quickack));
```

* **Buffer Sizes (`SO_RCVBUF` / `SO_SNDBUF`):** The operating system maintains memory buffers for unacknowledged outgoing data and unprocessed incoming data. Modifying these properties configures socket buffer sizes to optimize for specific network conditions and workload patterns. For bulk data transfers over high-speed links, default OS buffers are often too small, which prevents TCP from effectively scaling its window size.

```cpp
// Optimization: Increase Receive Buffer Size (4 MB)
// Must be done before listen() so Window Scaling is negotiated correctly
int rcvbuf = 4 * 1024 * 1024; 
if (setsockopt(server_fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf)) < 0) {
  log_error("setsockopt(SO_RCVBUF) failed:", errno);
}
```

We applied this optimizations to our TCP bulk transfer and RPC benchmarks.

<a id="bulk_transfer_2"></a>
<p align="center">
  <img src="tcp_bulk_transfer_server_2.png" width="48%" alt="TCP Server Output">
  <img src="tcp_bulk_transfer_client_2.png" width="48%" alt="TCP Client Output">
  <br>
  <em>Figure 5: Bulk Transfer of 1 GB Data Using TCP with Socket Optmizations</em>
</p>

<a id="rpc_2"></a>
<p align="center">
  <img src="tcp_rpc_client_2.png" width="80%" alt="TCP Client Output">
  <br>
  <em>Figure 6: 100K RPC Workload Using TCP with Socket Optmizations</em>
</p>

The unoptimized bulk transfer successfully saturated the network at ~934 Mbps. After applying the buffer size optimizations (`SO_RCVBUF` and `SO_SNDBUF` set to 4 MB), the transfer completed in 9.20524 seconds with a throughput of 933.157 Mbps. The lack of performance improvement indicates we already satuarted the network with the default buffer sizes. The Sunlab cluster is very fast so the impact of any system level optmizations will be very minimal. Also Linux TCP optmization already employs smart algorithms to automatically configure the buffer sizes to fit the workload.

The optimized RPC workload processed 100,000 transactions in 17.1002 seconds, yielding 5847.89 Transactions Per Second (TPS) with an average latency of 0.171002 ms per round-trip. This is only a marginal improvement over the unoptimized baseline of ~5813 TPS. While applying `TCP_NODELAY` and `TCP_QUICKACK` successfully removes delays by forcing immediate packet dispatch and acknowledgment, the performance remained flat. In a fast network with low latency, the delays introduced by Nagle's algorithm and delayed ACKs are not the primary bottleneck. Instead, the performance is bottlenecked by system calls at the application level.

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
