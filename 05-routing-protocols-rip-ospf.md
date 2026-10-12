# 05 — Routing Protocols: RIP & OSPF

> Scope: the two protocols in detail, then a practical comparison of distance-vector vs
> link-state in the field.

---

## 5.1 Classification of Routing Protocols

```
  Routing protocols
   ├── IGP (Interior Gateway Protocol)   -- inside one administrative domain (autonomous system)
   |     ├── Distance-vector:  RIP, (EIGRP)
   |     ├── Link-state:       OSPF, IS-IS
   |     └── Special:          EIGRP (advanced distance-vector / DUAL)
   └── EGP (Exterior Gateway Protocol)   -- between autonomous systems
         └── Path-vector:       BGP (the only real EGP in use today)
```

| | **IGP** | **EGP** |
|---|---|---|
| Scope | Inside a single AS | Between ASes |
| Knows | The internal topology | Only other ASes' reachability |
| Trust | Full trust (same admin) | **No trust** — policies needed |
| Metrics | Cost / hop count | **AS path, policies** — not "shortest path" |
| Examples | OSPF, IS-IS, RIP, EIGRP | **BGP** |

---

## 5.2 RIP — Routing Information Protocol

### 5.2.1 Specification summary

| Property | Value |
|---|---|
| Type | **Distance-vector** (IGP) |
| Metric | **Hop count only** |
| Maximum valid hop count | **15** |
| "Infinity" | **16** |
| Algorithm | Bellman-Ford |
| Transport | **UDP, port 520** |
| Multicast group | **224.0.0.9** (RIPv2) |
| Update interval | **30 seconds** (periodic) |
| Invalid timer | **180 seconds** (6 missed updates) |
| Flush timer | **240 seconds** (route removed from the table) |
| Triggered updates | Yes |
| Split horizon | **By default ON** (with poisoned reverse) |
| Max prefix length | 24 bits (a /16) per update |
| Authentication | None (plain text password in v2, weak) |
| Standard | RFC 1058 (v1), RFC 2453 (v2) |
| Status | **Deprecated / obsolete for general use** (RFC 6298) |

### 5.2.2 How RIP works

1. Each router sends its **entire routing table** to each neighbour every **30 seconds** in a
   RIP **response** message (UDP 520, destination multicast 224.0.0.9).
2. Neighbours compare received costs with their own; the better one wins.
3. If a route is lost, the router sends a **triggered update** immediately with that entry set
   to **16 (infinity)** — also called a **poisoned route**.
4. If no updates are heard for a destination for **180 seconds**, the entry is declared
   **invalid**; at **240 seconds** it is **flushed** (removed from the table).

### 5.2.3 RIP message format (RIPv2)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Command       | Version       | Address Family Identifier     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Route Tag                                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Destination Network Prefix                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Subnet Mask               |      Next Hop                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Metric                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* **Command**: 1 = Request, 2 = Response.
* **AFI** (Address Family Identifier): 2 = IP.
* **Route Tag**: used for route filtering / redistribution.
* **Metric**: 1–16, and **16 = unreachable**.
* Each update message carries up to **25 routes**; more routes means more messages.

### 5.2.4 RIP's weaknesses (this is what you must remember)

1. **Hop count is a terrible metric.** It treats a 100 m LAN link the same as a 5000 km
   satellite link. A 1-hop satellite link will always beat a 5-hop fibre path.
2. **Maximum 15 hops** (16 = infinity). Modern networks with multi-layer designs (leaf/spine,
   overlay, SD-WAN) can exceed 15 hops easily, so RIP will simply declare those networks
   unreachable.
3. **Full-table periodic updates** — every router sends its whole table to every neighbour every
   30 s. In a large network this is a constant bandwidth cost.
4. **Slow convergence.** Count-to-infinity and one-round-per-hop propagation.
5. **Bad news travels slowly.** A 30-second detection time at minimum; up to 30 s before a
   triggered update even fires.
6. **Periodic updates** mask link failures: a router keeps advertising a route for up to 30 s
   after the link died.
7. **No authentication**, **no hierarchy**, **no VLSM support in v1** (v2 added masks).
8. **Cannot support large networks** — the /16 limit and the flat design mean poor aggregation.

### 5.2.5 RIP variants and tools

* **RIPv1** — classful, broadcast (255.255.255.255), no masks, no authentication.
* **RIPv2** — classless (carries a subnet mask), multicast, supports authentication.
* **`distance` command** — artificially inflates the metric of a route so RIP will prefer a
  longer backup path. A deliberate hack to work around RIP's terrible metric.
* **`offset-list`** — filters or offsets routes to influence which are preferred.

### 5.2.6 When is RIP still used?

* As a **lab/exam protocol** — it is trivially simple to demonstrate.
* In **tiny, very stable** stub networks where a static route would be more effort.
* Occasionally by vendors of consumer gear that need absolute simplicity.
* **Never** in an enterprise core.

---

## 5.3 OSPF — Open Shortest Path First

### 5.3.1 Specification summary

| Property | Value |
|---|---|
| Type | **Link-state** (IGP) |
| Algorithm | **Dijkstra (SPF)** |
| Metric | **Cost**, inversely proportional to interface bandwidth |
| Transport | **Directly over IP, protocol number 89** (no UDP/TCP) |
| Multicast groups | **224.0.0.5** (all OSPF routers), **224.0.0.6** (all DR/BDR) |
| Hello interval | **10 s** (default, on broadcast/p2p networks) |
| Dead interval | **40 s** (4 × hello) |
| Dead route hold-time | **~40 s** (varies) |
| Refresh (LSRefreshTime) | **1800 s** (30 min) |
| MaxAge | **3600 s** (1 hour) — LSA is flushed |
| Database sync | Every **30 min** (and immediately on topology change) |
| SPF delay | **200 ms** hold time between LSA arrivals before an SPF run |
| SPF throttling | Backoff to prevent CPU storms |
| Area ID | 32-bit dotted decimal |
| Common areas | Area 0 = **backbone**; **totally stubby** is very common |
| Authentication | **Plain text, MD5, or SHA (HMAC)** |
| Versions | **OSPFv2** (IPv4), **OSPFv3** (IPv6) |
| Standard | RFC 2328 (v2), RFC 5340 (v3) |

### 5.3.2 OSPF message types

| Type | Name | Purpose |
|---|---|---|
| **1** | **Hello** | Discover neighbours, negotiate parameters, elect DR/BDR |
| **2** | **Database Description (DBD)** | Negotiate the master/slave relationship and summarise the LSDB |
| **3** | **Link State Request (LSR)** | Ask for specific LSAs the neighbour is missing |
| **4** | **Link State Update (LSU)** | Flood new or updated LSAs |
| **5** | **Link State Acknowledgment (LSAck)** | Confirms receipt of LSAs |
| 6–11 | (unused) | — |

All five go to **224.0.0.5** (or **224.0.0.6** for DR/BDR); Hello packets go to **224.0.0.5**
and unicast.

### 5.3.3 OSPF LSA types

| Type | Name | Scope | Generated by |
|---|---|---|---|
| **1** | **Router-LSA** | Area | Every router; describes its own links |
| **2** | **Network-LSA** | Area | The **DR/BDR**; describes the attached network |
| **3** | **Summary-LSA** | **Between areas** | An **ABR**; how area borders exchange summaries |
| **4** | **ASBR-Summary** | Between areas | An **ABR**; routes to external (redistributed) routes |
| **5** | **External-LSA** | **AS-wide** | An **ASBR**; redistributed routes from BGP/static/etc. |
| 7 | NSSA-LSA | Within a NSSA | An **ASBR** in a Not-So-Stubby Area |

Router-LSA, Network-LSA, Summary-LSA and ASBR-Summary are **masked** when they cross an area
boundary. External-LSAs are **not** flooded into stub areas.

### 5.3.4 OSPF area types

| Area type | External LSAs (type 5) | Inter-area routes | Use case |
|---|---|---|---|
| **Normal / stub** | Allowed | Allowed | Default area |
| **Stub** | **Blocked** (replaced by a default route) | Allowed | Small branch office with a single exit |
| **Totally stubby** | **Blocked** | **Blocked** (only a default route out) | **Cisco-proprietary**, extremely common for branch routers — minimal LSDB |
| **NSSA** | **Allowed but translated to type 7** | Allowed | Need to redistribute a non-OSPF protocol at the branch |

```
  Stub / Totally-stubby = "I will never be an exit point for external routes."
  NSSA = "I am an ASBR, but I won't carry type 5 LSAs."
```

### 5.3.5 OSPF areas and the backbone

```
                      +-----------+
                      |   Area 0  |   <-- the backbone, all Area 0s are logically one
                      |  backbone |
                      +-----+-----+
              +-------------+-------------+
        +-----+-----+                 +-----+-----+
        |  Area 1   |                 |  Area 2   |
        |  (normal) |                 | (totally |
        |           |                 |  stubby) |
        +-----+-----+                 +-----+-----+
              |                             |
         ABR (router between area 0 and 1) is an Area Border Router
```

* All areas must connect to **Area 0**. You cannot have an "island" area with no path to Area 0.
* **Type 3 (summary) LSAs are injected into every area** to summarise the other areas.
* A **virtual link** can extend Area 0 across a non-backbone area when a physical backbone is
  impossible (a WAN that is not Area 0).
* **ABR (Area Border Router)** — connected to multiple areas, translates LSAs.
* **ASBR (Autonomous System Boundary Router)** — redistributes routes from BGP, static, or another
  IGP into OSPF.

### 5.3.6 OSPF network types (crucial for exams and for configuration)

| Network type | DR/BDR election | Neighbours | Multicast | Use |
|---|---|---|---|---|
| **Broadcast** | **Yes** | Full mesh on a LAN | 224.0.0.5 | Ethernet LANs |
| **Non-broadcast multi-access (NBMA)** | **Yes** (manual `neighbor` statement needed) | Manual | No (unicast) | Frame Relay, ATM |
| **Point-to-multipoint** | No | Full mesh | 224.0.0.5 | Multiple PSTN circuits to the same hub |
| **Point-to-point** | **No** | Two | 224.0.0.5 | Serial links, /30 Ethernet, tunnels |

### 5.3.7 DR/BDR election — why it exists

On a **broadcast** network (a LAN), *every* router forms an adjacency with every other router.
With 20 routers that is 380 adjacencies, each exchanging full LSDB copies — unacceptable.

OSPF solves this with the **Designated Router (DR)** and **Backup DR (BDR)**:

* All other routers (**DROthers**) send their LSAs to the **DR/BDR only**.
* The **DR** is responsible for generating the **type 2 Network-LSA** for that network and for
  ensuring all LSAs are flooded.
* If the DR fails, the BDR takes over, and a new election happens.
* All routers on the LAN form **adjacencies with each other only** through the DR/BDR.

**Election rules (highest wins):**

1. Highest **interface priority** on that network (default 1; set to 0 to decline DR).
2. Highest **router ID**.

> Rule of thumb: set the priority of the two **routers you want to be DR/BDR** to something
> high (e.g. 10 and 5) and leave everything else at 1.

**The corollary that trips people up:** on a broadcast network with 3 routers where all
priorities are equal, R2 wins (highest router ID). R1 and R3 are DROthers and form a
**point-to-point adjacency with the DR only**, not with each other.

### 5.3.8 OSPF router ID

A **32-bit identifier**, chosen as:

1. Highest IP address on a **loopback** interface, if one exists; otherwise
2. Highest IP address on a **physical** interface at the moment OSPF starts.

It is a **dotted-decimal number, not an IP address** — it does not need to be routable.

### 5.3.9 OSPF operation step-by-step

```
1. HELLO DISCOVERY
   Routers send Hellos to 224.0.0.5 every 10 s on the link.
   Fields: Router ID, area ID, hello/dead intervals, priority, DR, BDR, neighbours seen.

2. DR/BDR ELECTION
   On a broadcast network, the highest priority / highest router ID becomes DR.
   Routers with lower priority become DROthers.

3. ADJACENCY FORMATION
   DROthers must form a FULL adjacency with the DR and BDR only.
   Point-to-point networks form full adjacency directly.

4. DATABASE EXCHANGE (2-WAY -> EXSTART -> EXCHANGE -> LOADING -> FULL)
   The neighbours exchange DBD packets, compare LSDB summaries, then request and flood
   the LSAs they are missing (LSR / LSU / LSAck).
   Result: an identical LSDB on all routers in the area.

5. SPF ALGORITHM
   Each router runs Dijkstra with itself as the root, producing a shortest-path tree.
   The tree is translated into the routing table, with type 1-4 next hops.

6. MAINTENANCE
   LSAs refresh every 1800 s; expire at 3600 s.
   A change -> immediate LSR/LSU -> re-run SPF.
```

### 5.3.10 The cost formula in practice

```
  cost = (reference bandwidth / interface bandwidth)

  Reference bandwidth default = 100 Mbps
  Fast Ethernet 100 Mbps  -> cost 1
  Serial 1.544 Mbps       -> cost 64
  Serial 256 kbps         -> cost 390
```

On a network with Gigabit links, **all interfaces have cost 1**, so OSPF cannot distinguish
them. Raise the reference bandwidth on Gigabit networks.

---

## 5.4 BGP — a brief comparison (for context)

BGP is not on the syllabus but appears in almost every comparison, so a short section:

| | RIP | OSPF | BGP |
|---|---|---|---|
| Type | DV | LS | **Path-vector** |
| Scope | IGP | IGP | **EGP** |
| Algorithm | Bellman-Ford | Dijkstra | Best path by **policy** |
| Transport | UDP 520 | IP 89 | **TCP 179** (reliable) |
| Metric | Hop count | Cost | **AS path, MED, local pref, policies** |
| Updates | Periodic, 30 s | Event-driven + 1800 s refresh | **Persistent (until withdrawn or session drops)** |
| Full table sent? | Yes, every 30 s | No (only changes) | **Yes, once at session start** |
| Scales? | No | Yes (within an AS) | Yes (**globally**, ~1 M prefixes) |
| Convergence | Slow | Fast | **Slow / non-deterministic** (by design) |
| Trust model | Trusted (IGP) | Trusted (IGP) | **Untrusted (EGP)** |
| Failure impact | Drops routes | Recomputes SPF | Drops the entire session; **keeps using the old table** |

> **Important exam point:** BGP's "floppy" convergence is intentional. Dropping all routes on a
> failure would cause a much larger blackhole. BGP prefers a stale-but-mostly-working table.

---

## 5.5 Distance-Vector vs Link-State in Practice

### 5.5.1 Side-by-side summary

| Feature | RIP (distance-vector) | OSPF (link-state) |
|---|---|---|
| Algorithm | Bellman-Ford | Dijkstra |
| Sends | Its **entire routing table** periodically | Only **its own links** (LSAs), on change |
| Update timing | Every **30 s** | Event-driven + 1800 s refresh |
| Knows about | Neighbours only | Entire area |
| Metric | Hop count (max 15) | **Cost** (bandwidth-based) |
| Converges | **Minutes** (or worse) | **Seconds** |
| Count-to-infinity | **Yes** | **No** |
| Loop-prone? | **Yes** | **No** |
| CPU on the router | Low | Higher (runs SPF), but throttled |
| Bandwidth | Higher (full tables, periodic) | Lower (only deltas) |
| Memory | Low | Higher (LSDB + routing table) |
| Scalability | **Very poor** — small, flat networks only | **Excellent** — hierarchical via areas |
| Robustness to bad data | **Good** (small tables, local) | **Weaker** (one bad LSA can hurt) |
| Aggregation | Basic (summary route) | **Excellent** (auto-summarisation + areas) |
| Authentication | Weak / none | **MD5, SHA** |
| Vendor neutrality | Yes | Yes (standard, though Cisco extensions like totally stubby) |
| Complexity to configure | **Very easy** | **Moderate** (areas, network types, DR/BDR) |
| IPv6 support | **None** | **OSPFv3** |
| Typical deployment | Labs, tiny legacy nets | **Enterprise cores** |

### 5.5.2 Real-world deployment picture

```
   INTERNET (between organisations)
   +------------------------------------------------------------------+
   |  eBGP between ISPs and large enterprises. Path-vector.           |
   |  Full internet table (~1 M IPv4 prefixes). TCP 179.              |
   +------------------------------------------------------------------+
                     |
   ENTERPRISE CORE (inside one AS)
   +------------------------------------------------------------------+
   |  OSPF underlay / IS-IS. Fast convergence. Areas = OSPF areas,   |
   |  Area 0 backbone. Cost = bandwidth. Full LSDB.                   |
   +------------------------------------------------------------------+
                     |
   BRANCH / ACCESS (small remote sites)
   +------------------------------------------------------------------+
   |  Mostly STATIC routes (a default route to the core), or OSPF      |
   |  totally-stubby area. Rarely RIP.                                 |
   +------------------------------------------------------------------+
```

**Why RIP is essentially extinct in enterprises:** any modern campus has leaf-spine layers,
redundant paths, and possibly SD-WAN overlays. RIP's 15-hop limit and 30-second updates are
simply incompatible with that. RIP survives only in course labs, in a handful of legacy embedded
devices, and occasionally as an emergency "get something up fast" protocol.

**Why OSPF dominates inside enterprises:** it converges in seconds, scales to thousands of
routers via areas, supports VLSM and authentication, and is a true open standard (vendor-neutral,
unlike EIGRP which was Cisco-proprietary until 2013).

---

## 5.6 Choosing a routing protocol

```
  Small, flat, static network (< 5 routers)?       ->  STATIC ROUTES.
  Small but needing dynamic behaviour?              ->  RIP (as a lab exercise) or OSPF.
  Enterprise campus/core?                          ->  OSPF (or IS-IS) with areas.
  Between organisations?                           ->  BGP (eBGP).
  Vendor-proprietary campus (Cisco only)?          ->  EIGRP (now an open standard via RFC 7868).
  Service provider?                               ->  IS-IS (better scaling than OSPF) + BGP.
```

---

## 5.7 Self-check questions

1. State RIP's maximum hop count and the value that represents "unreachable", and explain why.
2. What happens when a link fails directly attached to a router running RIP? How long until all
   routers converge, and why is that slow?
3. What are the RIP timers (update, invalid, flush) and what is the purpose of each?
4. What is the difference between RIP's "invalid" and "flush" timers?
5. Explain OSPF's DR/BDR election and why it is needed on a broadcast network.
6. List all five OSPF packet types and their purpose.
7. What is an area, why must all areas touch Area 0, and what is an ABR / ASBR?
8. What is the difference between a stub, a totally stubby area, and an NSSA?
9. How does OSPF compute its cost, and what happens to cost when all links are Gigabit?
10. How is the OSPF router ID chosen?
11. Why does OSPF send updates on IP protocol 89 while RIP uses UDP 520?
12. Explain why BGP convergence is "slow on purpose".
13. Summarise five practical scenarios where OSPF beats RIP and one where RIP would be simpler.

---

**Next → [06 — Practical Exercise](06-practical-exercise-configuration.md)**