# 04 — Routing Algorithms

> Scope: Distance-Vector (Bellman-Ford) and Link-State (Dijkstra), plus the pathologies of
> distance-vector (count-to-infinity) and their solutions, and the **path-vector** idea that
> BGP built on.

---

## 4.1 The Core Question

Given a topology of routers and costs, find the **best path** from source *s* to destination *t*.

Two classical families:

| | **Distance-vector (DV)** | **Link-state (LS)** |
|---|---|---|
| Each router knows | Its **neighbours' advertised costs** | The **entire topology** of its area |
| Algorithm | **Bellman-Ford** | **Dijkstra** |
| Local information | Distance + direction (next hop) per destination | Full map; computes the tree |
| Assumption | No router has the full picture | Everyone has the same picture |
| Goal | Find the best distance *through my neighbours* | Find the shortest path tree *from me* |

---

## 4.2 Distance-Vector: Bellman-Ford

### 4.2.1 The recurrence

Let `D_x(y)` = the known best cost from router **x** to destination **y**.
Then the update rule is:

```
  D_x(y) = min over neighbours v of  {  c(x,v) + D_v(y)  }
```

Read as: *"my best cost to y is whichever neighbour offers the cheapest path plus my cost to
reach that neighbour."* Because `c(x,v)` is always **positive** (cost > 0), the answer is always
found in a finite number of iterations — and if no neighbour has a route, `D_x(y) = infinity`.

Equivalent formulation (from the graph theory view):

```
  d(s,t) = 0                                  if s == t
  d(s,t) = min over edges (u,v)  { d(s,u) + cost(u,v) }   for v not yet solved
```

### 4.2.2 The algorithm (asynchronous)

```
  Initialise:   D_x(y) = c(x,y) if y is a direct neighbour,  INF otherwise
  Repeat:
     receive a distance vector from each neighbour v:  D_v(*)
     recompute:  D_x(y) = min { c(x,v) + D_v(y) }   for every destination y
     if anything changed, send the new vector to all neighbours
  Until no change.
```

Each exchange is called a **routing round** / **iteration**. With N routers in a line, the change
propagates **one hop per round** — so full convergence takes up to **N rounds**. This is the
fundamental weakness.

### 4.2.3 Worked example

Topology (costs in brackets):

```
        [Y:5]        [Z:6]        [W:8]
        /    \      /     \      /
       /      \    /       \    /
   [X:3]-----[V:2]---------[...]----
```

Let me use a cleaner classic 4-router example:

```
           2              3
      X ------- V ------- Y
      |       / |         |
      |     1   |         |
      |     /   | 3       |
      |   Z     |         |
      +---------W---------+
          4          1
```

Costs: X-V=2, V-Y=3, X-Z=1, Z-V=1, Z-W=4, V-W=3, Y-W=1.

Initial tables (direct neighbours known, everything else INF):

| Router | to X | to Y | to Z | to W |
|---|---|---|---|---|
| X | 0 | INF | 1 | INF |
| V | 2 | 3 | 1 | 3 |
| Z | 1 | INF | 0 | 4 |
| W | INF | 1 | 4 | 0 |
| Y | INF | 0 | INF | 1 |

**Round 1** (each router sends its table to all neighbours):

- **X** learns from V(3 to Y) → 2+3=5; from Z → nothing better.
  `X: to Y = 5 via V`
- **V** learns from X, Z, W. To W: direct 3, or via Z(1+4=5), or via Y(3+1=4) → keep 3.
  `V: to X = 2 direct, to Y = 3 direct, to Z = 1 direct, to W = 3 direct` (no change)
- **Z** learns from W(1 to Y) → 4+1=5; from V(3 to Y) → 1+3=4 → **4 via V**.
  `Z: to Y = 4 via V`
- **W** learns from V(2 to X) → 3+2=5 → **5 via V**.
  `W: to X = 5 via V`
- **Y** learns from V(2 to X) → 3+2=5 → **5 via V**.
  `Y: to X = 5 via V`

| Router | to X | to Y | to Z | to W |
|---|---|---|---|---|
| X | 0 | **5 via V** | 1 | INF |
| V | 2 | 3 | 1 | 3 |
| Z | 1 | **4 via V** | 0 | 4 |
| W | **5 via V** | 1 | 4 | 0 |
| Y | **5 via V** | 0 | INF | 1 |

**Round 2:**

- **W** now knows to X = 5 (via V), so to Y: direct 1, or via V = 3+2 = 5, or via **X**? W has no
  route to Z improvement… Actually W learns from Z: to Y = 4 → 4+4 = 8; from Y: to X = 5 → 1+5=6;
  from V: to Z = 1 → 3+1 = 4. So `W: to Z = 4 via V`.
- **Z** learns from W: to X = 5 → 4+5=9, no. To Y already 4.
- **X** learns from V: to W = 3 → 2+3 = 5 → `X: to W = 5 via V`.
- **Y** learns from V: to Z = 1 → 3+1 = 4 → `Y: to Z = 4 via V`.

| Router | to X | to Y | to Z | to W |
|---|---|---|---|---|
| X | 0 | 5 via V | 1 | **5 via V** |
| V | 2 | 3 | 1 | 3 |
| Z | 1 | 4 via V | 0 | 4 |
| W | 5 via V | 1 | **4 via V** | 0 |
| Y | 5 via V | 0 | **4 via V** | 1 |

**Round 3:** nothing changes. **Converged.**

| Router | to X | to Y | to Z | to W |
|---|---|---|---|---|
| X | 0 | 5 via V | 1 | 5 via V |
| V | 2 | 3 | 1 | 3 |
| Z | 1 | 4 via V | 0 | 4 |
| W | 5 via V | 1 | 4 via V | 0 |
| Y | 5 via V | 0 | 4 via V | 1 |

### 4.2.4 Path-vector (BGP)

BGP is a **fourth** category, taught alongside DV and LS.

```
  Distance-vector:  advertises the DISTANCE (cost) to each destination
  Path-vector:      advertises the actual PATH (the AS numbers it traversed)
```

This is enough to detect routing loops ("you already appear in my AS path") and to allow
**policy** (a router can prefer a route through AS 1234 even if it is longer). BGP is an
**inter-domain** protocol; its design goal is **policy correctness**, not finding the shortest
path. See §05.4.

---

## 4.3 The Problem with Distance-Vector: Count-to-Infinity

### 4.3.1 The pathology

The classic failure of Bellman-Ford when costs can change:

```
   Network:      X <---2---> Y
   Destination:  N is attached to X (cost 1)
   Initial:      X knows N=1 directly.
                 Y believes N = 3 (2 to X + 1)

   FAILURE: the link X-N goes down.

   X's own table:  N unreachable. Correct.
   Y still thinks N = 3 via X.

   Now the count starts:
     Y sends "I believe N = 3 via X" to X.
     X computes:  cost(X->Y)=2  +  Y's 3  =  5.   X believes N = 5 via Y!
     X sends that to Y.
     Y computes:  cost(Y->X)=2  +  X's 5  =  7.   Y believes N = 7 via X.
     X computes:  2 + 7 = 9 ...
     ... and so on:  3, 5, 7, 9, 11, ... until it hits the infinity threshold.

   Meanwhile, all packets for N bounce between X and Y forever
   (until the TTL expires).
```

**Why it happens:** Bellman-Ford assumes costs are **positive**, so information should converge.
But it assumes routers only *add* information. When a route is lost, the only information
available is the *neighbour's own claim*, which may itself be based on the lost route — so the two
routers reinforce each other's error.

### 4.3.2 Why the problem is *not* symmetric — bad news travels slowly

| Failure | Trigger | Message direction |
|---|---|---|
| Link goes down | **fast** (hold-down timer: one round trip) | **from the failure source outward** |
| **Bad news** | slow (count-to-infinity, 1 hop/round) | |

This asymmetry is why **bad news travels slowly** and good news travels fast — a fundamental
property of distance-vector protocols.

### 4.3.3 Impact

```
   Suppose a router reaches infinity in 250 rounds with a 1-second update interval.
   -> 250 seconds of congestion collapse.
   RIP caps the metric at 16 (infinitely far), so "counting to infinity" in RIP means
   counting to 16. But a full RIP convergence on a large network can still take minutes.
```

In a protocol with **no infinity cap** (e.g. EIGRP can theoretically reach 5,120,000), this is
even worse.

### 4.3.4 Solutions

**(a) Split horizon**

*Never advertise a route back to the neighbour you learned it from.*

```
   Split horizon rule: "If you learned a route from neighbour i, do not include it in the
                       routing table you send to i."
```

In the example: Y learned N from X, so Y never tells X "N = 3 via X". X never learns the bogus
route. **Solves the two-router case completely** (and most small cases).

**(b) Split horizon with poisoned reverse**

*Explicitly tell the neighbour the route is unreachable.*

```
   Instead of saying nothing about N to X, actively poison it:
       "N = 16 (infinity) via you"   (or 255 in RIP)
```

Advantage: **immediate and explicit.** The router explicitly denies the route instead of leaving
the neighbour in doubt. Consequence: updates are **larger**.

**(c) Hold-down timer**

*Ignore any updates about a route for *T* seconds after it is removed.*

```
   Pro: stops the router from believing flapping information.
   Con: delays legitimate re-advertisement of a recovered route.
   Mitigated by triggered updates.
```

**(d) Triggered (incremental) updates**

*Send an update immediately when the routing table changes, rather than waiting for the periodic
timer.*

```
   Pro: hugely reduces time-to-detect (a RIP failure is noticed in seconds, not up to 30).
   Con: can cause an "update storm" in a large network during a flapping event.
```

**(e) Infinite cost / metric cap**

*Bound the maximum metric* (RIP: 16). This **limits** the damage but does not eliminate the
loop; it just makes the loop end sooner.

**(f) Topology-based (hierarchical) design**

*Keep networks and routers organized in a strict hierarchy* (area borders, backbone area), so
loops are bounded. Used by OSPF and IS-IS.

**(g) Use a link-state protocol instead**

Dijkstra has **no count-to-infinity problem** because every router computes from an **identical,
complete picture**. That is why OSPF converges in seconds and RIP in minutes.

---

## 4.4 Link-State: Dijkstra's Algorithm

### 4.4.1 How it works

Each router:

1. **Discovers its neighbours** and the **cost** of the link to each (hello packets + costs).
2. **Floods its own link-state advertisements (LSAs)** to all routers in its area.
3. Every router builds an identical **link-state database (LSDB)** — a map of the whole area.
4. Runs **Dijkstra's algorithm** with **itself as the root**, producing a **shortest-path tree**.
5. Populates the **routing table** with the best next hop for every destination.
6. If anything changes, new LSAs are flooded and the tree is **recomputed incrementally**.

The result is that **every router independently computes the same tree**, with itself at the root
— so all routers agree without any further communication.

### 4.4.2 Dijkstra's algorithm

```
  Input:  adjacency list of the graph G(V,E) with non-negative costs,
          root node s
  Output: shortest distances d[] and the shortest-path tree T

  d[s]   = 0        ;  d[v] = INF  for all v != s
  T = { s }         ;  Q = V
  predecessor[v] = undefined

  while Q is not empty:
      u = extract_min(Q)              # the unvisited vertex with the smallest d[]
      for each neighbour v of u:       # scan u's adjacency list
          if d[v] > d[u] + cost(u,v):
              d[v]           = d[u] + cost(u,v)
              predecessor[v] = u
              insert/update v in the priority queue
      done
```

- `d[v]` = **distance label** (the best cost found so far to v).
- `extract_min` = pick the vertex with the smallest label.
- `predecessor[v]` = the next hop on the best path to v.

At the end, tracing `predecessor` backwards from any node to `s` gives the path.

**Correctness argument (invariant):** when a vertex `u` is extracted (removed from Q), its
distance label `d[u]` is **final**. Proof sketch: if a shorter path to `u` existed, the predecessor
`w` of `u` on that shorter path would have `d[w] < d[u]`, so `w` would have been extracted first,
and `u` would have been relaxed when `w` was scanned. Contradiction.

**Complexity:** `O((V + E) · log V)` with a binary heap. For 100,000 nodes this is very fast in
modern hardware — typically a few milliseconds.

> **Dijkstra requires non-negative costs.** This is why link-state protocols assign link *costs*
> (non-negative integers) rather than raw delay, which could be noisy or negative. RIP's hop
> count is also non-negative but cannot represent the topology — it discards the graph.

### 4.4.3 Why Dijkstra has no count-to-infinity

Because **every router sees the same complete, current graph** and computes from it locally,
there is no possibility of routers reinforcing each other's errors. If a link fails, every router
sees the same LSA withdrawal and recomputes identically. There is no path by which an error can be
amplified.

Also, Dijkstra's algorithm uses a **priority queue**, so information propagates according to cost
order rather than one hop per round.

---

## 4.5 Distance-Vector vs Link-State

| | **Distance-vector** | **Link-state** |
|---|---|---|
| Knowledge required | Neighbours only | Whole topology of the area |
| Algorithm | **Bellman-Ford** | **Dijkstra** |
| Metric | Any additive metric (hop count, cost) | Non-negative link **cost** only |
| Information sent | **The routing table** (a full list of distances) | **Only the router's own links** (link-state advertisements) |
| Message size | **O(n)** per update (grows) | **O(1)** per LSA (fixed) |
| Messages sent | **O(n)** per router per period | **O(n)** floods total (each LSA flooded once) |
| Table size per router | **O(n)** (neighbours × destinations) | **O(n)** LSDB + routing table |
| **Total state** | **O(n²)** | **O(n)** (per area) |
| Time to converge | Slow (rounds; count-to-infinity) | **Fast** (SPF recomputation) |
| **Loops** | Possible (count-to-infinity) | **None** (consistent database) |
| Updates are | **Periodic** (or triggered) | **Event-driven** |
| Failure of a node | **Revisions in the other routers** | **Other routers compute a new path; no need to change others** |
| Robustness to error | **Good** — small table, local view | **Bad** — corrupted LSA can cause loops or black holes |
| Complexity of implementation | **Simple** | **Complex** (5 LSAs in OSPF, 8 LSA types in IS-IS) |
| Scaling mechanism | Split horizon, aggregation, hierarchy, hold-down | **Areas/levels** |
| Examples | **RIP**, (EIGRP is an advanced hybrid), BGP (path-vector) | **OSPF**, IS-IS |

### Key insight

> Both algorithms are mathematically correct and produce identical optimal routing tables. The
> difference is **how they learn the topology** and the **communication cost** of doing so. DV is
> simpler with more total overhead; LS is more complex with better convergence and more
> robustness against loops.

---

## 4.6 Summary

```
  Distance-Vector (Bellman-Ford):
     - Know only neighbours. Advertise full distance vectors periodically.
     - Simple to implement, but suffers from count-to-infinity.
     - Mitigated by split horizon, poisoned reverse, hold-down, triggered updates.
     - Used by RIP (and adapted into EIGRP and BGP).

  Link-State (Dijkstra):
     - Flood own link costs to everyone; every router has the full map.
     - Complex but fast and loop-free.
     - Uses areas/levels to contain the map size.
     - Used by OSPF and IS-IS.

  Path-Vector (BGP):
     - Advertise the path (AS list) for loop detection and policy.
     - Used for inter-domain routing where policy matters more than speed.
```

---

## 4.7 Self-check questions

1. Write the Bellman-Ford recurrence from memory and explain each term.
2. Demonstrate count-to-infinity on a three-router topology. What are the values after each round?
3. Explain the "bad news travels slowly, good news travels fast" asymmetry.
4. What is split horizon? What is poisoned reverse? How do they differ?
5. Why is Dijkstra's algorithm guaranteed not to have a count-to-infinity problem?
6. Why does Dijkstra require non-negative link costs?
7. Compare the total memory and total message volume of DV and LS for a network of N routers.
8. Give two situations in which a link-state protocol might be *worse* than a distance-vector
   protocol.
9. Explain why OSPF uses cost (inversely proportional to bandwidth) rather than raw hop count.
10. What is the advantage of advertising a path (AS list) rather than a distance?

---

**Next → [05 — Routing Protocols: RIP & OSPF](05-routing-protocols-rip-ospf.md)**