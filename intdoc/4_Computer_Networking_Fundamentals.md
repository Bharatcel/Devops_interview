# 4. The Definitive Guide to Computer Networking

## Abstract

Computer networking is the bedrock upon which all distributed systems are built. For a FAANG-level engineer, a surface-level knowledge of "IP addresses and ports" is wholly inadequate. You are expected to have a first-principles understanding of the entire network stack, from the physical transmission of bits to the application layer. You must be able to debug complex network issues using low-level tools, design resilient and scalable network architectures, and articulate the trade-offs between different protocols and designs. This document provides that definitive, expert-level knowledge.

---

### Part 1: The OSI and TCP/IP Models Revisited

These models are not just academic concepts; they are mental frameworks for debugging. When something is broken, you should instinctively think: "What layer is the problem at?"

| OSI Layer | TCP/IP Layer | Purpose | Protocols | PDU | FAANG-Level Concerns & Debugging |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 7. Application | **Application** | Human-computer interaction | HTTP, HTTPS, DNS, SMTP, SSH | Data | Is the application code correct? Is the DNS name resolving? `curl -v`, `dig`, `openssl s_client` |
| 6. Presentation | (in App) | Data formatting, encryption | TLS/SSL, MIME | Data | Is there a TLS handshake failure? Is the character encoding correct? `openssl s_client` |
| 5. Session | (in App) | Manages sessions between apps | - | Data | Is the application maintaining state correctly? |
| 4. Transport | **Transport** | Host-to-host, process-to-process communication | **TCP**, **UDP** | **Segment (TCP)**, **Datagram (UDP)** | Is the port open? Is there a firewall blocking it? Is TCP state correct? `netstat`, `ss`, `nmap`, `telnet` |
| 3. Network | **Internet** | End-to-end packet delivery across networks | **IP** (IPv4, IPv6), ICMP, ARP | **Packet** | Can I ping the host? Is routing correct? Is there a NAT issue? `ping`, `traceroute`, `ip route` |
| 2. Data Link | **Link** | Node-to-node delivery on the same local network | Ethernet, Wi-Fi | **Frame** | Is the MAC address correct? Is the switch port configured correctly? `ip link`, `arp` |
| 1. Physical | **Physical** | Transmitting raw bits | Copper, Fiber, Radio | Bits | Is the cable plugged in? Is the fiber good? (The classic "Layer 1 issue") |

**Key Takeaway:** The TCP/IP model is the practical, simplified version you will use 99% of the time.

---

### Part 2: The Link Layer (Layer 2) Deep Dive

*   **MAC Addresses:** A 48-bit, globally unique "hardware" address burned into every Network Interface Card (NIC). Used for communication *within the same local network segment*.
*   **ARP (Address Resolution Protocol):** The glue between Layer 2 and Layer 3.
    *   **The Problem:** Your computer wants to send an IP packet to `192.168.1.100`. It knows the IP address (Layer 3), but to create the Ethernet frame (Layer 2), it needs the destination's MAC address.
    *   **How it Works:**
        1.  Your computer shouts an ARP Request to the broadcast MAC address (`ff:ff:ff:ff:ff:ff`) on the local network: "Who has IP address `192.168.1.100`? Tell `192.168.1.50` (me)."
        2.  All devices on the local network receive this broadcast.
        3.  The device that owns `192.168.1.100` replies with an ARP Reply directly to your computer's MAC address: "I have `192.168.1.100`. My MAC address is `aa:bb:cc:dd:ee:ff`."
        4.  Your computer stores this mapping in its ARP cache (`arp -a`) and can now send the Ethernet frame.

---

### Part 3: The Internet Layer (Layer 3) Deep Dive

*   **IP (Internet Protocol):** The protocol that provides the global addressing scheme and routes packets across different networks. It is **unreliable** and **connectionless**. It makes a "best effort" to deliver packets, but provides no guarantee of delivery, no guarantee of order, and no error correction.
*   **IP Packet Header:** Contains source IP, destination IP, TTL (Time to Live - decremented by each router, packet is dropped if it hits 0 to prevent infinite loops), and the protocol of the payload (e.g., TCP is protocol 6, UDP is 17).
*   **Routing:**
    *   Every computer and router has a **routing table** (`ip route` or `netstat -r`).
    *   When a packet needs to be sent, the kernel consults this table.
    *   If the destination IP is on a local subnet, it sends it directly (using ARP to find the MAC).
    *   If the destination is on a remote network, it sends the packet to the MAC address of its configured **default gateway** (the local router). The router then repeats this process.
*   **`traceroute` (or `tracert`):** A brilliant diagnostic tool that maps the path a packet takes. It works by sending IP packets with increasing TTL values. The first packet has TTL=1, the first router decrements it to 0, drops it, and sends back an ICMP "Time Exceeded" message. `traceroute` records the router's IP. Then it sends a packet with TTL=2, which gets dropped by the *second* router, and so on.

---

### Part 4: The Transport Layer (Layer 4) Masterclass: TCP vs. UDP

This is one of the most critical topics for any software engineer.

#### UDP (User Datagram Protocol)
*   **Philosophy:** "Fire and forget."
*   **Characteristics:**
    *   **Connectionless:** No handshake. You just send the datagram.
    *   **Unreliable:** No acknowledgements, no retransmissions. If a packet is lost, it's gone.
    *   **Unordered:** Packets may arrive out of order.
    *   **Lightweight:** Very small header (8 bytes).
*   **Use Cases:** Situations where speed is more important than reliability.
    *   **DNS:** A fast query/response is ideal. If a DNS request is lost, the client just asks again.
    *   **VoIP / Video Streaming:** Losing a single packet is better than pausing the entire stream to wait for a retransmission.
    *   **Monitoring/Metrics:** If a single metrics packet is lost, it's usually not a big deal.

#### TCP (Transmission Control Protocol)
*   **Philosophy:** "Reliable, ordered, stream-oriented delivery."
*   **Characteristics:**
    *   **Connection-Oriented:** Requires a **Three-Way Handshake** to establish a connection.
    *   **Reliable:** Uses **Sequence Numbers** and **Acknowledgements (ACKs)** to guarantee every segment is received. If an ACK isn't received in time, the segment is retransmitted.
    *   **Ordered:** The sequence numbers allow the receiver to reassemble the segments in the correct order, even if they arrive out of order.
    *   **Flow Control:** Uses a **Sliding Window** mechanism. The receiver advertises how much data it can currently buffer (the "receive window"). The sender will not send more data than this window size, preventing the receiver from being overwhelmed.
    *   **Congestion Control:** Algorithms (like Reno, CUBIC) that slow down the sending rate if packet loss is detected, to avoid overwhelming the network itself.

#### ★ FAANG-Level Deep Dive: The TCP State Machine ★

You must be able to draw this on a whiteboard. `netstat -an` or `ss -t` will show you these states.

1.  **CLOSED:** The starting point.
2.  **LISTEN:** (Server only) The server is waiting for an incoming connection request.
3.  **SYN-SENT:** (Client only) The client has sent a `SYN` packet and is waiting for a `SYN-ACK`.
4.  **SYN-RECEIVED:** (Server only) The server has received a `SYN`, sent a `SYN-ACK`, and is waiting for the final `ACK`.
5.  **ESTABLISHED:** The three-way handshake is complete. Data can be transferred. This is the normal state for an active connection.

**Terminating a connection (The Four-Way Handshake):**
6.  **FIN-WAIT-1:** The application on this side has closed, and it has sent a `FIN` packet.
7.  **CLOSE-WAIT:** This side has received a `FIN` from the other side and is waiting for its local application to close. A server stuck in `CLOSE-WAIT` often indicates a bug where the application isn't properly closing its sockets.
8.  **FIN-WAIT-2:** This side has received an `ACK` for its `FIN` and is now waiting for a `FIN` from the other side.
9.  **LAST-ACK:** This side has received a `FIN`, sent its own `FIN`, and is now waiting for the final `ACK`.
10. **TIME-WAIT:** This side has received the final `FIN` and sent the final `ACK`. It now waits for a period (2 * Maximum Segment Lifetime) to ensure the final `ACK` was received and to handle any stray, delayed packets. A high number of connections in `TIME-WAIT` is normal for a busy server.

---

### Part 5: The Application Layer (Layer 7) Deep Dive

*   **DNS (Domain Name System): The Internet's Phonebook**
    *   **The Problem:** Humans remember names (`google.com`), but computers need IP addresses (`172.217.14.238`).
    *   **How it Works (Recursive Query):**
        1.  Your computer asks its configured **resolving nameserver** (e.g., `8.8.8.8`).
        2.  The resolver checks its cache. If not found, it asks one of the **Root Nameservers** (`.`).
        3.  The Root server says, "I don't know, but here's the IP for the **`.com` TLD (Top-Level Domain) server**."
        4.  The resolver asks the `.com` TLD server.
        5.  The TLD server says, "I don't know, but here's the IP for the **Authoritative Nameserver for `google.com`**."
        6.  The resolver asks the `google.com` nameserver.
        7.  The `google.com` nameserver replies with the IP address.
        8.  The resolver caches the result (respecting its TTL) and returns it to you.
    *   **`dig` is your best friend:** `dig +trace google.com` will show you this entire process.
    *   **Record Types:** `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias), `MX` (mail server), `TXT` (text records, used for SPF, DKIM).

*   **HTTP (Hypertext Transfer Protocol):** The language of the web.
    *   **HTTP/1.1:** Text-based, uses persistent TCP connections. Suffers from **Head-of-Line Blocking** (a slow request blocks all subsequent requests on the same connection).
    *   **HTTPS:** Just HTTP over TLS (Transport Layer Security). Provides encryption, authentication, and integrity. The TLS handshake happens *after* the TCP handshake but *before* any HTTP data is sent.
    *   **HTTP/2:** Binary protocol. Introduces **multiplexing**, allowing multiple requests and responses to be interleaved on the same TCP connection, solving head-of-line blocking.
    *   **HTTP/3:** Runs on **QUIC**, which is a new transport protocol built on top of **UDP**. This eliminates TCP head-of-line blocking entirely and provides faster connection setup.

---

### Part 6: Modern Network Architecture & ★ FAANG-Level Scenarios ★

*   **NAT (Network Address Translation):** The technology that allows multiple devices in a private network (e.g., `192.168.x.x`) to share a single public IP address. Your home router does this. In the cloud, a NAT Gateway does this for private subnets.
*   **Load Balancers:** Distribute incoming traffic across multiple backend servers.
    *   **Layer 4 (Transport):** Operates at the TCP/UDP layer. It just forwards packets to a backend server based on IP and port, without looking at the content. Very fast.
    *   **Layer 7 (Application):** Operates at the HTTP layer. It can make routing decisions based on the content of the request (e.g., URL path, headers, cookies). It can terminate TLS, inspect traffic, and provide features like sticky sessions.
*   **VPC (Virtual Private Cloud):** Your own logically isolated section of a public cloud (like AWS). You have full control over your virtual networking environment, including your own IP address range, subnets, route tables, and network gateways.

#### ★ Interview Scenario 1: The Slow Website ★

*   **Question:** "A user reports that `www.example.com` is slow to load. How do you debug this?"
*   **Answer:** "I would debug this systematically, moving up the network stack.
    1.  **Layer 7 (DNS/Application):** Is DNS resolution slow? I'd run `time dig www.example.com`. A slow response points to a DNS issue. If DNS is fast, I'd use `curl -v -w "dns_lookup: %{time_namelookup} | tcp_connect: %{time_connect} | tls_handshake: %{time_appconnect} | start_transfer: %{time_starttransfer} | total: %{time_total}\n" https://www.example.com`. This breaks down the request lifecycle. Is the TLS handshake slow? Is the server itself taking a long time to generate the first byte (`time_starttransfer`)? This would point to an application or database problem.
    2.  **Layer 3/4 (Network/Transport):** Is there network latency or packet loss between me and the server? I'd run `ping www.example.com` to check latency and basic packet loss. Then I'd run `traceroute www.example.com` (or `mtr`) to see which hop on the path is introducing latency. High latency at a specific router could indicate network congestion.
    3.  **On the Server:** If I have access to the server, I'd check its resource utilization (`top`, `iostat`). Is it overloaded? I'd check the application logs for errors. I'd use `netstat -an | grep ESTABLISHED | wc -l` to see the number of active connections. Is it running out of sockets?"

#### ★ Interview Scenario 2: Designing a Scalable Web Architecture ★

*   **Question:** "Design a highly available and scalable network architecture for a popular e-commerce website on AWS."
*   **Answer:** "I would design a multi-tier architecture within a VPC.
    1.  **VPC and Subnets:** I'd create a VPC (e.g., `10.0.0.0/16`). Within it, I'd create **public** and **private** subnets across multiple Availability Zones (AZs) for high availability.
    2.  **Internet Gateway & Load Balancer:** An Internet Gateway would be attached to the VPC to allow public traffic. An **Application Load Balancer (ALB)** would be placed in the public subnets. This ALB would be the single entry point for all user traffic. It would terminate HTTPS, offloading that work from the web servers.
    3.  **Web Tier:** The web servers (e.g., EC2 instances in an Auto Scaling Group) would be placed in the **private subnets**. They would not have public IP addresses. They would only accept traffic from the ALB. This protects them from direct attack from the internet.
    4.  **Application/Database Tier:** The application servers and databases would be in separate private subnets, even more isolated. The web tier would communicate with the app tier, and the app tier with the database tier, using security groups as firewalls.
    5.  **Outbound Traffic (NAT):** If the private instances need to access the internet for things like software updates or calling external APIs, their traffic would be routed through a **NAT Gateway** located in the public subnet. This allows the private instances to initiate outbound connections without being publicly reachable.
    6.  **DNS:** We would use Amazon Route 53 for DNS, with an `A` record (or `ALIAS` record) pointing our domain name to the public DNS name of the Application Load Balancer."
