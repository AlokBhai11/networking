# 01 — Network Layer Design Issues

> Scope: what the network layer is *for*, the services it offers, the difference between
> **routing** and **forwarding**, how **store-and-forward** works, why **congestion control**
> belongs at this layer, and what **internetworking** means.

---

## 1.1 Positioning of the Network Layer

The OSI model gives each layer a job:

| Layer | Unit | Core question | Example protocol |
|---|---|---|---|
| 7 Application | message | "What does the data mean?" | HTTP, DNS, SMTP |
| 6 Presentation | — | "How is it encoded/encrypted?" | TLS, JPEG |
| 5 Session | — | "Which conversation is this?" | NetBIOS, RPC |
| 4 Transport | segment/datagram | "Which process, and is it reliable?" | TCP, UDP |
| **3 Network** | **packet / datagram** | **"Which destination host, across which links?"** | **IP, ICMP, ARP** |
| 2 Data Link | frame | "Which device on this link?" | Ethernet, PPP |
| 1 Physical | bits | "What is the voltage?" | 10GBASE-T |

The **network layer** is the first layer that is **not concerned with a single link**. Data
link layers work hop-to-hop; the network layer works **end-to-end across an entire internetwork**.

The Internet network layer is the **Internet Protocol (IP)**. It is deliberately minimal:

* **Connectionless** — no setup, no teardown, no state in the network.
* **Best-effort** — the network makes *no* guarantee of delivery, order, uniqueness, or
  delay. This is the famous **"best-effort service"**.
* **Datagram** — each packet is independent and independently routable.

Everything else (reliability, ordering, congestion feedback at the end host) is pushed up into
the **transport layer (TCP)** or down into **end-host behaviour**. IP is deliberately dumb so it
can be implemented cheaply and run over anything.

---

## 1.2 Store-and-Forward

Because a router may be forwarding between links with **different bandwidths, different frame
formats, or different maximum frame sizes**, a packet usually cannot be streamed out while it is
still arriving. Routers therefore use **store-and-forward**:

```
Packet arrives  ->  [ receive ENTIRE packet into buffer ]  ->  Forward it out next link
                         ^
                  "store"
```

### Why "forward and switch" instead of "cut-through"?

* **Correctness of service guarantees.** If a router streams a packet before knowing its final
  destination (e.g. a datagram that might need to be *copied to several outgoing interfaces* for
  multicasting), it must buffer the packet first to avoid mangling frames already on the wire.
* **Buffering prevents fragmentation problems.** If an outgoing link requires smaller frames than
  the incoming link, the router must have the **complete** packet to be able to fragment it
  correctly (fragmentation needs to know the total length and the offset of the original datagram).
* **Different link types.** A packet might be arriving on a 10 Mbps Ethernet link and leaving on a
  56 kbps leased line — an output rate far slower than input. Store-and-forward lets the output
  queue do its job (see congestion control below).

### The cost

* **Delay (latency).** Each hop adds the serialization delay:
  `transmission delay = L / R` (packet length / link rate) plus the propagation delay
  `d / S` (distance / signal speed).
* **Memory (buffers).** Every hop consumes buffer space. When arrival rate exceeds service rate,
  the buffer fills — see §1.5.

> **Exam pointer:** store-and-forward is also why end-to-end throughput can never exceed the
> **slowest** link on the path, and why a path with *n* routers accumulates *n* transmission +
> propagation delays. This is why traceroute (§03) matters operationally.

---

## 1.3 Services Offered by the Network Layer

A network layer protocol provides (at minimum) these services to the transport layer:

### 1.3.1 Connectionless, best-effort datagram service

* Packets are independent; no setup, no acknowledgement.
* Packets may be **lost, duplicated, delayed, or reordered**.
* Typical: **IP (IPv4 and IPv6)**, also IPX, AppleTalk.

### 1.3.2 Connection-oriented virtual-circuit service

A virtual circuit is a logical path established before data flows. Two flavours:

**(a) Reliable, in-order delivery** — packets arrive exactly once, in sequence. Example:
**ATM** (Axiom Virtual Circuits) with AAL classes. Rarely deployed in pure form today.

**(b) Unreliable, in-order (datagram with ordering guarantee)** — example: early frame relay.

> The Internet deliberately chose **(a) = connectionless best-effort** at layer 3, because a
> network that tries to be reliable has to keep per-flow state and must retry — and one loss
> should not stall an entire virtual circuit. It is cheaper and more robust to let the *endpoints*
> (TCP) do reliability on a stateless network.

### 1.3.3 Internetworking interface

The "network layer" also has to **interconnect different network-layer protocols**. IP is the
interconnection vehicle: it runs *over* Ethernet, PPP, Frame Relay, ATM, MPLS, 802.11, satellite,
and even over itself. That is **internetworking** (§1.6).

### Encapsulation boundary

```
Ethernet frame   [ | IP header | payload | ]  <- the "IP datagram" is the frame's payload
                   ^
                   everything above the horizontal line is layer-3 concern
```

---

## 1.4 Routing vs Forwarding

This is the single most important distinction in the entire course. Both happen inside a router,
at the same instant, but they are completely different jobs.

| | **Forwarding** | **Routing** |
|---|---|---|
| **What** | Move one packet from input port to output port | Choose the *best path* between source and destination across the whole network |
| **Scope** | **Local**, per packet, sub-millisecond | **Global**, per destination-prefix, seconds to minutes |
| **Decided by** | **Data plane** (hardware: ASIC/TCAM forwarding tables) | **Control plane** (software: routing protocol + routing table) |
| **Input** | Arriving packet + local forwarding table | Link-state info / neighbour metrics + policy |
| **Output** | Selected outgoing interface | Installed **routes** into the forwarding table |
| **Table** | **Forwarding table / FIB / CAM table** | **Routing table / RIB / control-plane table** |
| **Changes** | Very frequently (every packet) | Infrequently (on topology/prefix change) |
| **Analogy** | A receptionist reading the room number on a badge and pointing the way | The architect deciding *which corridor* people should use |

### Data plane vs control plane

```
        CONTROL PLANE  (software, slow, "brain")
   ┌──────────────────────────────────────────────┐
   │  routing protocol  (OSPF / RIP / BGP /       │
   │  static routes / policy)                     │
   │            ↓  installs                       │
   │  ROUTING TABLE (RIB)                         │
   └───────────────┬──────────────────────────────┘
                   │  copy of best routes
                   ▼
        DATA PLANE  (hardware, fast, "muscle")
   ┌──────────────────────────────────────────────┐
   │  FORWARDING TABLE (FIB / TCAM)               │
   │  lookup destination -> {next hop, interface, │
   │                         outgoing if}         │
   └──────────────────────────────────────────────┘
```

### Worked example — one lookup

Packet arrives: `src 10.0.0.5  ->  dst 8.8.8.8`

1. Read the **destination address** `8.8.8.8` out of the IP header.
2. Strip any link-layer frame (the frame is discarded; only the datagram is forwarded).
3. **Decrease TTL** by 1. If TTL becomes 0 -> drop and send ICMP *Time Exceeded* (back to §03).
4. Look up `8.8.8.8` in the **forwarding table**:
   ```
   8.8.8.8/32 via 192.168.1.1, out Gi0/1
   10.0.0.0/8   directly connected, out Gi0/0
   0.0.0.0/0    via 192.168.1.1, out Gi0/1      <-- default route
   ```
5. Longest-prefix match wins -> `0.0.0.0/0` via `192.168.1.1`, exit `Gi0/1`.
6. **Decrease the TTL** check again, recompute the **header checksum** (IPv4 only).
7. Wrap the datagram in a **new Ethernet frame** for the outgoing link, with the *next hop's*
   MAC address as the new destination MAC (and the router's own MAC as source).
8. Transmit.

Note step 7: **every hop re-encapsulates** the layer-2 frame but leaves the layer-3 IP header
essentially intact (except TTL/checksum). This is what makes IP "internetworking glue".

---

## 1.5 Congestion Control at the Network Layer

**Congestion** = the offered load to a network exceeds the resources available to carry it.
Congestion is a **network** problem; **flow control** (avoid overwhelming a single receiver) is an
**end-to-end** problem. They are frequently confused.

### Where congestion shows up

```
        server
          |  100 Mbps of offered traffic
          v
      [ core router ]  ---- link A: 1 Gbps  ----
          |
        [ edge router ]  ---- link B: 10 Mbps  ----
          |                       ^
          |              packets queue up here, then drop
          v
      [ server ]
```

Queues fill up and packets are **dropped**. Consequences:

* **Goodput** (useful data delivered) falls below throughput.
* **Delay and jitter** explode (queueing delay dominates once the queue is non-empty).
* **Retransmissions** from TCP waste even more capacity -> a **congestion collapse**.
* **Router CPU/memory** spent on routing-protocol updates instead of forwarding.

### The fundamental law (a.k.a. the congestion triangle / Efficiency equation)

```
        Goodput = Offered Load × Efficiency

        1              1
Efficiency = -------------   ≤  1
             (1 + 2 * Delay)

  where Delay = Prop delay + Queueing delay

        Throughput
Eff.   = ---------   ≥  1    at light load (no queueing delay)
        Offered Load

        Eff.  ->  0            at heavy load (queueing delay -> infinity)
```

Interpretation:

* **Low load** — throughput = offered load, efficiency = 1, delay is just propagation.
* **High load** — a small increase in load causes a large increase in queueing delay. This is the
  *congestion collapse* point. Hop-by-hop working exactly as designed produces a globally bad
  outcome.

### Classic congestion-control principles

| Principle | Meaning |
|---|---|
| **Retransmit only when necessary** | Timers must be conservative (RTO ≳ RTT). Excess retransmissions make congestion worse. |
| **Retransmit fast** | If congestion is detected, retransmit quickly — don't wait for a timer. |
| **Do not retransmit at the same congestion level** | Receiving duplicate ACKs means congestion; the second copy of a packet is useless. |
| **Abandon useless packets** | Drop packets that will be useless by the time they arrive. |
| **Resynchronize congestion to the current network** | After a long quiet period, back off from the old operating point. |
| **Conservative new flows** | An aggressive new flow can destroy the well-tuned state of existing flows (a "TCP meltdown"). |

### Who does what

| Layer | Mechanism | Scope |
|---|---|---|
| 2 (link) | **Ethernet PAUSE frames** (flow control, optional, rarely used on WAN) | Single link |
| 3 (network) | **RED/WRED/ECN** — Random Early Drop / Weighted ECN. Routers signal congestion by *dropping probabilistically as the queue grows*, and with **Explicit Congestion Notification** explicitly mark packets (DSCP ECN bits) instead of dropping them. | Single network |
| 4 (transport) | **TCP** congestion control: slow start, congestion avoidance, fast retransmit/recover | End-to-end |

### The dilemma: who controls congestion?

1. **End-to-end (TCP)** — only endpoints know whether the *user* actually needed the retransmission.
   It adapts to the bottleneck path. But it is **slow to react** and has a "TCP meltdown" risk:
   it can add more traffic to a network that is already congested.
2. **Network-assisted (ECN/RED)** — routers signal congestion explicitly; TCP reduces its rate.
   Needs per-flow state in routers; does not work for UDP.
3. **Network-only** — routers drop or shape proactively (buffering, QoS). Cannot see user intent.

In practice the Internet uses a **hybrid**: ECN/RED where endpoints cooperate, plus **UDP
traffic-control design** (e.g. DNS, real-time media) that is *not* supposed to overwhelm the net.

> **Key insight:** the Internet network layer is best-effort and *has no congestion control at
> all* as a protocol. Congestion control is an **end-to-end, cooperative** phenomenon layered on
> top. This was a deliberate design choice by Cerf & Kahn in 1974: keep the core simple and
> stateless so it scales.

---

## 1.6 Internetworking

**Internetworking** = connecting physically and logically dissimilar networks into one
inter-network (the "Internet"). A **heterogeneous network** mixes different technologies.

### The problem

* Ethernet frames (MTU 1500) are not PPP frames (MTU 1500 but different header format).
* A Layer-2 address (48-bit MAC) is **only meaningful within one link**. It has no global meaning.
* Different networks have different **addressing, error control, flow control, and QoS** models.
* Nobody wants to rewrite their protocol stack for every new link type.

### The solution: a common layer-3 protocol (IP)

IP provides the universal envelope:

```
 ┌──────────────────────────────────────┐
 │        Transport (TCP/UDP)           │  <-- application data
 ├──────────────────────────────────────┤
 │        IP  (header: addresses, TTL)  │  <-- THE INTERWORKING LAYER
 ├─────────┬──────────┬─────────────────┤
 │ Ethernet│   PPP    │  Frame Relay    │  <-- radically different link layers
 ├─────────┴──────────┴─────────────────┤
 │         physical media               │
 └──────────────────────────────────────┘
```

Consequences of this design:

* Any link layer can carry IP -> IP is **media independent**.
* **Layer-2 addresses are re-created at every hop** (see the forwarding walkthrough in §1.4), while
  **layer-3 addresses are end-to-end** (they survive the whole path).
* IP hides the physical topology from the transport layer.

### Translation, tunnelling, and multi-homing

Internetworking needs three more tools:

* **Address/NBMA translation (NAT)** — lets a host with a non-globally-unique address reach the
  public Internet by reusing a public address in a translation table. (Section 02.)
* **Tunnelling** — encapsulating one protocol inside another so an incompatible network can
  "carry" it:
  ```
  [IP | UDP | DNS] ---> [IP | UDP(1701) | GRE | IP | UDP | DNS] ---> [IP | UDP | DNS]
                         ^-- tunnelled                                  ^-- decapsulated
  ```
  Examples: **GRE**, **IPsec tunnel mode**, **PPP over Ethernet**, **VXLAN**, **6to4**.
* **Router advertisement / fragmentation differences** — IPv6 routers **never fragment**; only the
  source may do so, and only via IPv6 routers using Path MTU Discovery + ICMPv6 *Packet Too Big*.
  A router that must fragment must drop the packet and tell the source.

### The 4-Internet problem

The classic internetworking taxonomy (Tanenbaum):

| | **IPv4 internet** | **IPv6 internet** |
|---|---|---|
| Addresses | Heterogeneous: addresses from different subnets/nets | Homogeneous: all addresses are 64-bit LAN IDs |
| Connection establishment | Connectionless (datagram) | Connection-oriented (virtual circuit) |
| State at routers | Stateless (per-packet routing) | Stateful (per-flow state, QoS) |

The Internet today is largely an **IPv4 internet**; the v6 side of the table is a vision from
early work, not a description of production IPv6.

---

## 1.7 Why Layer 3 Is Where the Internet Lives

* **Global addressing.** Layers 1–2 give you a local address (MAC). Only layer 3 gives a *globally
  unique hierarchical* address, which is what makes routing tables aggregatable (Section 03).
* **Aggregation.** A hierarchical address space lets one routing table entry (a prefix) summarize
  millions of hosts. Without it, routing tables would not fit.
* **Decoupling.** A protocol that is decoupled from the lower layer can run over any of them, which
  is exactly what "internetwork" means.

---

## 1.8 Summary table

| Issue | Position | Summary |
|---|---|---|
| Unit of work | L3 | Packet / datagram, end-to-end across many links |
| Delivery | L3 | Connectionless, **best effort**; no loss/order/delay guarantees |
| Hopping mechanism | L3 | **Store-and-forward** (buffer whole packet, then transmit) |
| Per-packet job | **Forwarding** | Local, hardware FIB lookup, microseconds |
| Path-decision job | **Routing** | Global, software control plane, seconds |
| Overload | L3 | **Congestion**; queue overflow -> drops -> delay -> collapse |
| Interconnection | L3 | **Internetworking** via a common protocol (IP) + NAT/tunnelling |
| Reliability | L4 | TCP, not IP |

---

## 1.9 Self-check questions

1. Why must a router store a packet before forwarding it, rather than streaming bits straight out?
2. What is the difference between congestion and flow control? Give one example of each.
3. State the congestion efficiency equation and describe what happens to efficiency as load grows.
4. Give one concrete reason a router needs a *routing* table in addition to a *forwarding* table.
5. When a packet is forwarded, which fields of the layer-2 frame change, and which fields of the
   layer-3 header change? Which stay the same?
6. Explain what a "virtual circuit" service is and why IP chose not to provide one.
7. Why is IPv6's design rule "routers never fragment" a form of internetworking requirement?

---

**Next → [02 — IPv4 vs IPv6 Concepts](02-ipv4-vs-ipv6.md)**