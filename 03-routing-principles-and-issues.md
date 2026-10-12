# 03 — Routing Principles & Issues

> Scope: routing metrics, scalability, convergence, and routing loops (including TTL).

---

## 3.1 What Routing Is

**Routing** = the process by which routers and hosts decide *which path* packets should take
towards each destination, and (in the control plane) exchange information so that all routers
agree.

Recall from §01:

* **Routing** — global, per-destination, control plane, slow.
* **Forwarding** — local, per-packet, data plane, fast.

A **routing protocol** is the set of rules by which routers **exchange** information and
**compute** the routing table. It does not move packets itself — it feeds the forwarding table.

### The two things every routing protocol must do

1. **Learn** the topology / reachability of destinations (from neighbours, or statically).
2. **Compute** the best path to each destination and install it.

### Routing table entries

```
  Destination prefix   Next hop        Interface   Metric   Protocol
  10.0.0.0/8           directly conn.  Gi0/0       0        connected
  10.1.2.0/24          directly conn.  Gi0/1       0        connected
  10.1.3.0/24          172.16.0.2      Gi0/1       10       OSPF
  172.16.0.0/16        directly conn.  Gi0/1       0        connected
  0.0.0.0/0            192.168.1.1     Gi0/2       1        static
```

---

## 3.2 Static vs Dynamic Routing

| | **Static routes** | **Dynamic routes** |
|---|---|---|
| Configured by | Administrator, manually | Learned automatically by a protocol |
| CPU/memory | Almost none | Protocol process, memory for tables |
| Change detection | Manual — **easy to forget** | Automatic |
| Scale | Small networks only (or a few critical routes) | Large and changing networks |
| Security | **No protocol to attack**; easy to filter at the edge | Must be secured (OSPF auth, BGP MD5/SAFI) |
| Failure detection | Manual (or via interface down) | Automatic (hello/dead intervals) |
| Predictability | Total | Depends on the protocol |
| Examples | `ip route 10.5.0.0 255.255.0.0 172.16.1.1` | RIP, OSPF, IS-IS, EIGRP, BGP |

**Best practice:** use static routes for a default route out of a small branch, or for stub
networks; use dynamic routing in the core.

### Administrative distance

Cisco-style devices install the best route **per prefix** according to administrative distance —
the trustworthiness of the source, checked before the metric.

| Protocol / Source | AD |
|---|---|
| Connected interface | 0 |
| Static route | 1 |
| External BGP | 20 |
| Internal BGP | 200 |
| **OSPF** | **110** |
| IS-IS | 115 |
| EIGRP (internal) | 90 |
| RIP | 120 |
| ODR | 160 |
| iBGP | 200 |
| eBGP | 20 |
| Unknown / unreachable | 255 |

**Lower AD wins.** A static route to `0.0.0.0/0` (AD 1) will always beat an OSPF-learned default
(AD 110). This is how you override a dynamically-learned route.

---

## 3.3 Routing Metrics

A **metric** is a measure of the "cost" of a path. It is **protocol-specific** and **only
comparable within the same protocol**.

### Common metrics

| Metric | Description | Used by | Problem |
|---|---|---|---|
| **Hop count** | Number of routers traversed | **RIP** | Simple, but treats a 1 km link and a 10 000 km link as equal |
| **Bandwidth** | Inverse of link speed | OSPF, EIGRP, IS-IS | Ignores delay, congestion, and load; a 10 Gb link may be busy while a 1 Gb link is idle |
| **Delay** | Round-trip propagation time | EIGRP | Hard to measure; changes constantly; **can oscillate** |
| **Load** | Link utilisation | (EIGRP default metric) | Changes rapidly -> **route flapping** |
| **Reliability** | Error rate | EIGRP | Subjective |
| **Cost** | Configured value | OSPF, IS-IS | Operator-defined; **OSPF cost is inversely proportional to bandwidth** |
| **AS path length / policies** | — | **BGP** | Not a "cost"; BGP is policy-driven, not cost-driven |

### OSPF cost formula

```
  cost = reference_bandwidth (bps) / interface bandwidth (bps)
```

Cisco's classic default `reference-bandwidth = 100 Mbps (100 000 kbps)` gives:

| Interface | Bandwidth | Cost |
|---|---|---|
| Serial 56 kbps | 0.056 Mbps | 1785 |
| T1 1.544 Mbps | 1.544 | 64 |
| Fast Ethernet | 100 Mbps | **1** |
| Gigabit Ethernet | 1000 Mbps | **1** |
| 10 GbE | 10 000 Mbps | **1** |

All links ≥ 100 Mbps get cost 1, so OSPF would prefer a congested 10 Gb link over an idle
100 Mbps link. Fix: **raise the reference bandwidth** so cost stays meaningful (e.g. 10 000 Mbps
for 10 GbE). (Alternatively use OSPF-TE / link metrics, or EIGRP.)

```
  With reference-bandwidth = 10000 Mbps:
     100 Mbps  -> cost 100
     1 Gbps   -> cost 10
     10 Gbps  -> cost 1
```

**EIGRP default metric** is a weighted combination:

```
  EIGRP metric = 256 * [ (K1 * BW) + (K2 * RELIABILITY * BW) + (K3 * DELAY) ] / K0

  K1=1, K2=0, K3=1, K4=0, K5=0 by default
```

### Metrics that make convergence hard

* **Heterogeneous metrics.** You cannot compare a hop count of 3 with a bandwidth of 100 Mbps.
  Each protocol uses its own metric, so two different protocols will disagree about the best
  path — which is why **multiple protocols on one router can oscillate** ("routing feedback").
* **Non-additive metrics.** Only metrics that satisfy
  `cost(A→B→C) = cost(A→B) + cost(B→C)` compose cleanly. Delay and load do not add, which
  creates **count-to-infinity-like** instability.

---

## 3.4 Scalability

### Why routing must scale

The number of inter-domain destinations in the global BGP table:

```
  1994:   ~5 000 IPv4 prefixes
  2000:  ~150 000 prefixes
  2008:  ~330 000 prefixes
  2015:  ~600 000 prefixes
  2020:  ~850 000 prefixes
  2024:  ~1 000 000 IPv4 prefixes
```

At ~80–100 bytes per BGP entry, that is roughly **80–100 MB of IPv4 routing table** — exactly at
the edge of what older edge routers could hold in RAM. Memory exhaustion and route flapping
during updates are real operational risks.

### The three techniques for scale

**(a) Aggregation / hierarchy**

```
  Without aggregation:  a router must know routes for 10.0.1.0/24, 10.0.2.0/24, ... 10.0.500.0/24
  With aggregation:     one entry 10.0.0.0/16 covers all of them
```

* **Intra-AS**: OSPF **areas** and **IS-IS levels** partition the topology. Each router only needs
  a full picture of its own area plus a summary of the rest.
* **Inter-AS**: BGP summarises advertisements (`aggregate-address`, `network ... mask`, or
  provider/traffic engineering communities).

**(b) Split horizon / hierarchy of trust**

* **Distance-vector**: only learn routes from neighbours (one-hop knowledge) — cheap memory, but
  many messages.
* **Link-state**: full topology picture, but **only within an area** — bounded memory, few
  messages.

**(c) Address aggregation**

CIDR (§02) is what makes hierarchy possible: the address itself encodes the aggregation
structure, so routers can summarize without extra configuration.

### Complexity comparison

| | Distance-vector | Link-state |
|---|---|---|
| Knowledge | Neighbours only | Whole area |
| **Routing table size** | **O(n)** per node → **O(n²)** total | **O(n)** total |
| **Messages** | O(n²) total; every update is **sent to every neighbour** | O(n) floods, each node has the full LSDB |
| **Computation per node** | Simple Bellman-Ford: O(n × k) | **Dijkstra: O((V + E) log V)** — more CPU |
| **Memory** | Small | Larger (LSDB = copy of the topology) |
| Robustness | **Good** — views are local, no need for a complete picture | **Fragile** — one corrupted LSA can cause a **black hole** or a **routing loop** |
| Change propagation | **Slow** (periodic updates, count-to-infinity) | **Fast** (immediate, triggered) |

### The scalability myth to unlearn

> **"Distance-vector scales better than link-state."**
>
> This is *not* quite true. DV uses less memory per node but sends far more messages and converges
> more slowly. LS converges in seconds using more CPU but O(n) memory. **Inside a single AS, the
> large LS protocol (OSPF) is preferred for exactly this reason: its *area* mechanism lets it
> scale.** In the wide area between organisations, BGP (a **path vector** protocol) is used,
> which is neither DV nor LS — it adds policy to the DV idea.

---

## 3.5 Convergence

**Convergence** = the time it takes **all routers** in the network to agree on the correct
forwarding information after a change.

```
  t=0      Link R1-R2 fails.
  t=0.01   R1 and R2 notice (hello timeout, carrier loss, or triggered update).
  t=0.05   R1 removes the routes that used that link. R3, R4, R5 ... do NOT know yet.
  t=0.05..X SLOW CONVERGENCE: some routers still forward packets into the black hole.
  t=X      All routers have the same table.  -> CONVERGED
```

During non-convergence you have **transient loops, black holes, and inconsistent paths** — this
is when users actually experience an outage, even though the "failure" was over in a millisecond.

### Slow convergence in distance-vector

Two causes:

1. **Count-to-infinity** (§04) — routers keep believing a bogus route and count it up.
2. **Periodic updates** — if the protocol only sends an update every 30 s (RIP), detection alone
   can take up to 30 s.

Fixes: **triggered updates**, **hold-down timers**, **split horizon with poisoned reverse** (§04),
and small hello/hold-down values.

### Fast convergence in link-state

* **Hello every 10 s, dead after 40 s** (OSPF default) — but OSPF also detects a **down** link
  instantly, and detects a dead neighbour in ~40 s (adjustable down to ~1 s).
* **Triggered (incremental) updates**: as soon as anything changes, the change is **flooded**
  immediately.
* A full **SPF (Dijkstra) run** is triggered by every change, but it is **fast** — microseconds
  to milliseconds for a few thousand nodes.
* **LSA ageing / max-age (3600 s)** — LSAs expire, so a corrupt or lost LSA self-heals.
* **Sequence numbers** — a router with a *newer* LSA always wins, so flood storms and stale
  information resolve deterministically.

### The convergence–stability trade-off

```
  Very fast timers  -> fast convergence, but tiny blips cause flapping and CPU storms
  Very slow timers  -> stable, but slow to notice and recover from real failures

  Dynamic route damping: suppress a route for exponentially increasing periods
  after each instability, until it is "stable enough". Used in BGP (route flap damping).
  IP Prefix List + damping + hold-down are the classic Cisco tools.
```

### False positives and false negatives

```
  False positive: a link is UP but the protocol thinks it is DOWN -> unnecessary reconvergence.
  False negative: a link is DOWN but the protocol thinks it is UP -> packets lost in a black hole.
```

Both increase with **slow timers** (later detection) and with **unstable media** (wireless,
satellite, congested links). The classic fix is a **hold-down timer** to damp flapping.

---

## 3.6 Routing Loops and TTL

### 3.6.1 Routing loop (packet looping)

A **routing loop** occurs when packets circulate between routers forever because each router's
table points back toward the previous router.

```
        Loop between R1 and R2:

     R1  --->  R2                 Packet destined for 10.9.9.9 (unreachable)
     ^         |                 R1's table says "go to R2"
     |         v                 R2's table says "go to R1"
     +---------  (loops forever)
```

**Causes:**

* **Count-to-infinity** in distance-vector protocols (§04).
* **Stale / inconsistent routing tables** during convergence.
* **Misconfiguration** (a static route pointing back at the wrong interface).
* **Redistribution errors** — redistributing a route back into the protocol it came from without
  a filtering prefix-list, creating a loop between two protocols.

**Consequences:** the packet's **TTL** runs out (see below) and it is dropped; the router sends
ICMP **Time Exceeded** back. In the meantime **CPU and link bandwidth are wasted** carrying a
packet that will never arrive. A single loop can consume an entire link's capacity.

**Prevention:**

* TTL / Hop Limit (the universal, guaranteed backstop)
* Split horizon + poisoned reverse (§04)
* Triggered updates + hold-down timers
* Route tagging / filtering when redistributing between protocols
* Administrative distance ordering (so the more trustworthy source wins)
* Deterministic algorithms (OSPF, IS-IS, EIGRP) with sequence numbers

### 3.6.2 TTL (IPv4) / Hop Limit (IPv6)

**Purpose 1 — prevent packets circulating forever.**

Every router that forwards a packet decrements TTL by 1:

```
  Host sets TTL = 64 (typical Windows) or 255 (Cisco default; Linux often 64).

  64 -> 63 -> 62 -> ... -> 1 -> 0   =>  router sees TTL = 1, decrements to 0,
                                      DROPS the packet, and sends
                                      ICMP Time Exceeded (Destination Unreachable)
                                      back to the SOURCE (not to the destination)
```

> Critical detail: the ICMP error goes back to the **original source**, because the router
> cannot know the original next hop (NAT, and the path back may be different). This means the
> **source** is the one that must respond by reducing its packet size (Path MTU Discovery, §02.9).

Maximum value: **255** (8-bit field). So a packet can traverse at most 254 routers.
This bounds the loop damage: a looping packet consumes at most 255 hops of bandwidth.

**Purpose 2 — `traceroute` / `tracert`.**

```
  Traceroute exploits TTL deliberately:
   - Send a packet with TTL = 1. Router 1 decrements to 0, drops it, and returns
     ICMP Time Exceeded revealing ITSELF.  -> hop 1
   - Send TTL = 2. Router 2 does the same.                              -> hop 2
   - ...
   - When the DESTINATION receives it, it returns ICMP Port Unreachable (UDP probe)
     or an ICMP Echo Reply (ICMP probe).  -> destination reached

  Output:
     1  10.0.0.1        1 ms   0.821 ms
     2  192.168.1.1     8 ms   7.905 ms
     3  203.0.113.1    14 ms  13.771 ms
     4  * * *                  (a router filtered ICMP — no response)
    10  8.8.8.8        27 ms  26.401 ms

  A "*" means the router did not respond to ICMP. It does NOT necessarily mean the router is
  broken — many routers rate-limit ICMP (control-plane policing) and deliberately do not answer.
```

**Purpose 3 — hiding internal topology / traceroute security.**

A low TTL prevents packets from reaching internal hosts when sent externally.

**Purpose 4 — multicast scoping.**

IP multicast packets use the TTL to confine them to an administrative scope:

| TTL | Scope |
|---|---|
| 0 | Same host |
| 1 | Same subnet (link) |
| < 32 | Same site |
| < 64 | Region |
| < 128 | Organization |
| 255 | Global |

### 3.6.3 The problem with TTL as a loop-preventer

* **Control-plane loops.** TTL protects the *data plane*. It does **not** stop the *routing
  protocols* from continuing to count routes to infinity in their tables. A loop in the routing
  table wastes CPU even after every packet's TTL expires.
* **Black holes are silent.** If a router drops packets for any reason other than TTL, the source
  hears nothing at all. This is harder to debug than a TTL error, which is at least *evidence*.

---

## 3.7 Routing Table Lookup — Algorithm and Tie-Breaking

When more than one route matches a destination, the **route selection algorithm** decides. The
classic Cisco sequence:

```
  1. Longest prefix match (most-specific / lowest prefix length wins)
  2. Local preference (BGP; higher is better)
  3. Administrative distance (lower is better)   -- actually decided before metric
  4. Metric (lower is better)
  5. Equal-cost multi-path (ECMP)
  6. Tie-breakers: internal BGP > external BGP; lowest router ID; shortest cluster list;
     lowest neighbour IP; oldest route
```

**ECMP (Equal-Cost Multi-Path)** — if two routes have the same prefix, AD and metric, load-share
across them by hashing (usually the **source/destination port 5-tuple**) so that a flow stays on
one path and returns in order.

---

## 3.8 Static vs Dynamic — decision summary

```
  Use STATIC when:                    Use DYNAMIC when:
  - the network is small/stable       - the network is large/changing
  - you want zero CPU                 - you need automatic failover
  - you want a backdoor               - you want load-sharing
  - there is a default route out      - the topology is multi-path
  - the route must NEVER be advertised  - you need fast convergence
```

---

## 3.9 Self-check questions

1. Explain the difference between routing and forwarding, and what device subsystem performs each.
2. What is administrative distance and how does it interact with the routing metric?
3. Give four examples of routing metrics and identify one problem with hop count and one with load.
4. Why is OSPF's default reference bandwidth of 100 Mbps a problem in a 10 Gb environment?
5. Why did routing protocols need to aggregate routes? Give a number for the size of the global
   BGP routing table.
6. Explain how OSPF areas make link-state scale, and why DV "does not scale" is a simplification.
7. What is convergence and why does slow convergence hurt users even when the failure is brief?
8. Describe four ways a routing loop can arise.
9. What exactly does a router do when the TTL reaches 0, and where does the ICMP error go?
10. How does traceroute get the IP addresses of intermediate routers?
11. Why can a "*" in traceroute output be a false alarm?
12. What does TTL protect, and what does it *not* protect?

---

**Next → [04 — Routing Algorithms](04-routing-algorithms.md)**