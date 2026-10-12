# 02 — IPv4 vs IPv6 Concepts

> Scope: addressing in IPv4 and IPv6, IPv4 address exhaustion and the tools that postpone the
> problem (NAT, DHCP, CIDR), simplified IPv4/IPv6 headers, and IPv6 Stateless Address
> Autoconfiguration (SLAAC).

---

## 2.1 IPv4 Addressing

### 2.1.1 The dotted-quad

IPv4 addresses are **32 bits**, written as four decimal octets separated by dots:

```
  a.b.c.d        a,b,c,d  in decimal, 0–255 each   ->  8 bits x 4 = 32 bits

  192.168.10.1   = 11000000.10101000.00001010.00000001
  10.0.0.5       = 00001010.00000000.00000000.00000101
  8.8.8.8        = 00001000.00001000.00001000.00001000
```

> Note the leading-zero trap: `010` is technically **octal** in some parsers and `10` is decimal.
> Always write full decimal, never `192.168.010.1`.

### 2.1.2 Structure: network + host

```
 31          24 23          16 15           8 7            0
+--------------+--------------+--------------+--------------+
|   network    |    network   |    network   |     host     |
+--------------+--------------+--------------+--------------+

  Class A:     0....  network(7)              host(24)
  Class B:    10...  network(14)             host(16)
  Class C:   110...  network(21)             host(11)
  Class D:  1110...  multicast group          (no host part)
  Class E: 1111...  reserved / experimental
```

Classful addressing is **obsolete**. It wasted enormous space (a Class C gave 254 hosts; a Class B
gave 65 534) and every network boundary reset the routing table. It was replaced by **CIDR**
(§2.4). Classes survive only as a historical concept and in a couple of legacy conventions
(Class E, and "Class A/B/C" marketing language).

### 2.1.3 Special-use IPv4 addresses (memorise these)

| Address / Range | Name | Meaning |
|---|---|---|
| `0.0.0.0` | "this network" | Unspecified / default route source |
| `127.0.0.0/8` | **Loopback** | Local host. Always routes to itself; never leaves the machine |
| `10.0.0.0/8` | **Private** (RFC 1918) | Class A private — 16,777,216 addresses |
| `172.16.0.0/12` | **Private** | 172.16.0.0 – 172.31.255.255 |
| `192.168.0.0/16` | **Private** | Class C private — 65,536 addresses |
| `169.254.0.0/16` | **APIPA / link-local** | Self-assigned "there's no DHCP server" address |
| `127.0.0.1` | localhost | The loopback interface |
| `224.0.0.0/4` | Multicast | Class D; 224.0.0.0/24 = link-local control (OSPF, etc.) |
| `240.0.0.0/4` | Reserved | Class E |

**Private (RFC 1918) addresses are not globally unique and must never appear on the public Internet.**
They are the basis of NAT (§2.5).

### 2.1.4 Subnet mask, prefix length, and host count

A **subnet mask** separates network bits from host bits.

```
  IP address     192.168.10.37
  Mask           255.255.255.0   =  /24
                 ^^^ ^ ^^ ^^^^ = 24 ones followed by 8 zeros

  AND operation:
     192.168.10.37
 AND 255.255.255.0
   = 192.168.10.0   <-- network address
```

```
  /24 mask in binary:  11111111.11111111.11111111.00000000
  Wildcard (inverse):  00000000.00000000.00000000.11111111   <- used in ACLs
```

**Host count formula:**

```
  usable hosts = 2^(32 - prefix) - 2

  /24  -> 2^8  - 2 =      254 usable   (192.168.10.0   network, 192.168.10.255 broadcast)
  /25  -> 2^7  - 2 =      126 usable
  /26  -> 2^6  - 2 =       62 usable
  /27  -> 2^5  - 2 =       30 usable
  /28  -> 2^4  - 2 =       14 usable
  /29  -> 2^3  - 2 =        6 usable
  /30  -> 2^2  - 2 =        2 usable   (a point-to-point link)
  /31  -> 2^1  - 2 =        0 usable   (RFC 3021 point-to-point, 2 usable addresses)
  /32  -> 2^0  - 2 =       -2         (one host route; loopback / router interface)
```

The **-2** is for the all-zeros **network address** (identifies the subnet) and the all-ones
**broadcast address** (limited broadcast within the subnet).

### 2.1.5 Type of addresses

* **Unicast** — one host. Global (`8.8.8.8`) or private (`192.168.1.1`).
* **Broadcast** — every host on the subnet. Directed broadcast `192.168.10.255`; limited
  broadcast `255.255.255.255`.
* **Multicast** — a group. `224.0.0.0/4`. Used by OSPF, EIGRP, VRRP, and IPTV.

### 2.1.6 Fragmentation (IPv4 only)

If a packet is larger than the outgoing link's MTU, an IPv4 router **may** fragment it:

```
  Original:  [ DF | MF=0 | offset=0   | payload 4000 B ]
                       |__ IP header: Fragment Offset (13 bits, in 8-byte units)
  Frag 1:    [ DF | MF=1 | offset=0   | first  1480 B ]
  Frag 2:    [ DF | MF=0 | offset=185 | rest  2520 B ]

  DF = Don't Fragment flag   ->  set it and routers must drop instead of fragment
  MF = More Fragments flag
```

* **IPv4:** any router may fragment. ICMP *Destination Unreachable / fragmentation needed* is
  generated when the DF bit is set and the packet is too big (this is how Path MTU Discovery works).
* **IPv6:** only the **original source** may fragment, and only via extension headers.
  **Routers never fragment.** A router that cannot forward a packet drops it and returns
  ICMPv6 **Packet Too Big (type 2)** so the source can reduce its size.

> Modern practice: fragmentation is avoided, not used. Path MTU Discovery (RFC 1191 / 1981)
> finds the smallest MTU end-to-end. See §2.9.

---

## 2.2 IPv4 Address Exhaustion — The Problem

IPv4 has **2^32 ≈ 4.3 billion** addresses. Wasted space makes it far fewer in practice.

### Why 4.3 billion ran out (~2011)

* **Classful waste** (largely replaced by CIDR, but historical).
* **Static assignment** — organisations were given far more than they needed.
* **Unmanaged growth** — every new department, lab, partner connection took a new network.
* **"Just use a new ISP" theory of allocation** — IANA deliberately allocated /8s to ISPs, who
  were supposed to route and re-allocate them downstream. Many never did; they held whole /8s
  for a single customer or left them idle. **Route table pollution** — all these dormant /8s had to
  be announced on the global BGP table, wasting routing table space.
* **Mobile devices and IoT** — by the 2010s every phone, tablet, TV, and sensor needed an address.
* **Leaked addresses** — although reputation systems and NAT gave some protection.

**Timeline:** IANA free pool exhausted **3 February 2011**. The RIR free pools followed
through 2015–2017; **APNIC ran out entirely in April 2017**.

### Why not simply allocate all 4.3 billion addresses?

1. **Routing table exhaustion.** Assigning addresses randomly makes prefixes **non-contiguous**,
   so no aggregation is possible. The global BGP routing table (~1 million IPv4 routes in 2024)
   would need one entry per address — 4.3 billion entries. BGP routers cannot hold that.
   *This is the fundamental reason, not the shortage of addresses per se.*
2. **Total address space exhausted**, so any leak permanently destroys value.
3. Even a country-sized allocation is more than it needs (e.g. the USA has ~1.5 billion IPv4 for
   ~330 million people).

### The three delays

| # | Technique | Section | Effect |
|---|---|---|---|
| 1 | **CIDR** (supernetting / prefix aggregation) | §2.4 | Saves routing table space; postpones exhaustion |
| 2 | **DHCP** (address reuse over time) | §2.3 | Addresses shared by devices that are not on at once |
| 3 | **NAT / PAT** (address reuse across sites) | §2.5 | Thousands-to-millions of hosts share one public IP |
| 4 | **IPv6** (the real fix) | §2.7 | 2^128 addresses |

---

## 2.3 DHCP — Dynamic Host Configuration Protocol

DHCP (RFC 2131) solves two problems at once: **address exhaustion** (hosts reuse a small pool
rather than holding a permanent address) and **configuration** (mask, gateway, DNS, NTP, domain).

### Operation: DORA

```
   Client                                                Server
     |  broadcast DHCP DISCOVER  (src 0.0.0.0, dst 255.255.255.255)
     |-------------------------------------------------------->
     |                                                        |
     |  DHCP OFFER  (proposed address + options)              |
     |<--------------------------------------------------------
     |                                                        |
     |  DHCP REQUEST  ("I want THAT one", broadcast or unicast)
     |-------------------------------------------------------->
     |                                                        |
     |  DHCP ACK  (address, netmask, router, DNS, lease time)  |
     |<--------------------------------------------------------
```

| Message | Broadcast? | Purpose |
|---|---|---|
| **D**ISCOVER | Yes | "Is there a DHCP server out there?" |
| **O**FFER | Yes (normally) | Server proposes an address + parameters |
| **R**EQUEST | Yes | Client accepts one offer (broadcast so all servers know) |
| **A**CK | Yes (normally) | Server confirms the lease |

### Client state machine

```
INITIAL -> SELECT -> REQUESTING -> BOUND -> RENEWING -> REBINDING -> EXPIRED
  |           |  (RELEASE)         |
  |           |                    |
  REBOOT ---> RENEWING            DHCPINFORM (already has an address, just wants options)
  (restart, try the cached address first)
```

### The DORA messages on the wire

```
DHCP DISCOVER  (UDP 68 -> 67, broadcast)
   src IP 0.0.0.0    dst IP 255.255.255.255   src MAC xx  dst MAC ff:ff:ff:ff:ff:ff
   op=1 BOOTREQUEST  htype=1 hlen=6  hops=0
   xid=<transaction id>
   secs=0  flags=0  ciaddr=0.0.0.0  yiaddr=0.0.0.0  siaddr=0.0.0.0  giaddr=0.0.0.0
   chaddr=<client MAC>  sname="" file=""
   magic cookie 0x63825363
   options:
      53 (message type) = 1 (DISCOVER)
      55 (parameter request list) = 1,3,6,15,51,54,...

DHCP OFFER / ACK adds:
      54 (server identifier) = 192.168.1.1
      51 (IP address lease time) = 86400
      1  (subnet mask)      = 255.255.255.0
      3  (router / gateway) = 192.168.1.1
      6  (DNS server)       = 8.8.8.8, 8.8.4.4
      15 (domain name)      = example.com
      51 (lease)            = 86400 s (24 h)
```

### Important DHCP options (option 43 vendor, 60 vendor class, 66/67 TFTP, 69 TFTP for PXE)

| Code | Name |
|---|---|
| 1 | Subnet Mask |
| 3 | Default Gateway (Router) |
| 6 | Domain Name Server |
| 12 | Host Name |
| 15 | Domain Name |
| 26 | Interface MTU |
| 28 | Broadcast Address |
| 42 | NTP Servers |
| 51 | IP Address Lease Time |
| 53 | DHCP Message Type |
| 54 | Server Identifier |
| 55 | Parameter Request List |
| 58 | **Renewal Time (T1)** — typically 50% of the lease |
| 59 | **Rebinding Time (T2)** — typically 87.5% of the lease |
| 60 | Vendor Class Identifier |
| 66 / 67 | Boot Server / Boot File Name (PXE network boot) |

### DHCP relay (routing the request beyond the local subnet)

DHCP uses **broadcast**, which routers do not forward. A **DHCP relay agent** on each subnet:

1. Receives the client's broadcast DISCOVER.
2. **Adds the GIADDR (gateway IP address)** field = the relay's own interface IP on that subnet.
3. Unicast-forwards it as **UDP 67 -> 67** to the DHCP server.
4. The server uses GIADDR to choose the right subnet for the pool.
5. The server unicasts the OFFER back to GIADDR; the relay **broadcasts** it on the client's subnet.

> This is exactly the case in the practical exercise (§06) — a DHCP server in the datacentre
> handing out addresses to clients across routed subnets.

### DHCP and IPv6: DHCPv6 vs SLAAC

| | **DHCPv6** | **SLAAC** |
|---|---|---|
| Model | Stateful (server tracks leases) | Stateless (server only sends parameters) |
| Addresses | From the server | Client generates them itself (§2.8) |
| Extra config | DNS, TFTP, domain | **RDNSS** option (RFC 8106) in Router Advertisements |
| Broadcast | No — uses UDP 547/546 to multicast `ff02::1:2` | No |
| Today | Still common on enterprise networks | Default on most consumer OSes (Windows/macOS/Android) |
| Combined | **DHCPv6-PD** for routed prefixes | Also common: prefix delegation + SLAAC for hosts |

> Many networks now use **SLAAC for addresses + DHCPv6 (stateless) for options + RDNSS** for
> DNS. The classic exam answer is "SLAAC is the default in IPv6 because it removes the need for a
> DHCP server."

---

## 2.4 CIDR — Classless Inter-Domain Routing

CIDR (RFC 4632, obsoleting RFC 1519) does two things at once:

1. **Adds a prefix length** (`/n`) to an address, so the network/host split is flexible — no
   more classes.
2. **Allows supernetting** — one routing entry can represent many adjacent subnets.

### Supernetting (route aggregation)

```
  Four separate routes in the old world:
      10.0.1.0/24
      10.0.2.0/24
      10.0.3.0/24
      10.0.4.0/24

  One route in the CIDR world:
      10.0.0.0/21      (covers 10.0.0.0 - 10.0.7.255)
        10.0.0.0  = 00001010.00000000.0000000[0]
        10.0.7.255= 00001010.00000000.0000000[1]
                   ^ the 21st bit varies -> /21 covers 8 x /24
```

```
  Router A in another AS only needs:   10.0.0.0/21
  instead of 4 entries.  (And it can point 10.0.0.0/16 at Router A to be even more general.)
```

### Subnetting inside a supernet (FLSM vs VLSM)

| | **FLSM** — Fixed-Length Subnet Mask | **VLSM** — Variable-Length Subnet Mask |
|---|---|---|
| All subnets the same size? | Yes | No — each sized to its actual need |
| Wastes address space? | Yes | Minimally |
| Still taught? | In exams | In real deployments |

**Worked VLSM example.** Allocate from `192.168.10.0/22` (= 192.168.8.0 – 192.168.11.255):

| Need | Hosts required | Smallest subnet | Size | Address range | Gateway (first usable) |
|---|---|---|---|---|---|
| LAN A | 500 | 2^9 - 2 = 510 | **/23** | 192.168.8.0 – 192.168.9.255 | 192.168.8.1 |
| LAN B | 100 | 2^7 - 2 = 126 | **/25** | 192.168.10.0 – 192.168.10.127 | 192.168.10.1 |
| LAN C | 50 | 2^6 - 2 = 62 | **/26** | 192.168.10.128 – 192.168.10.191 | 192.168.10.129 |
| WAN link | 2 | 2^2 - 2 = 2 | **/30** | 192.168.10.192 – 192.168.10.195 | 192.168.10.193 |
| Spare | — | — | /22 | 192.168.11.0 – 192.168.11.255 | — |

**Rules of thumb:** choose the subnet that satisfies the *largest* requirement first; use
`network + size` to find the next free block; never overlap; always leave the network and
broadcast addresses unused.

### Longest-prefix match (most-specific match wins)

```
  Routing table:
      0.0.0.0/0        via 192.168.1.1        <-- default
      10.0.0.0/8       via 172.16.0.5
      10.1.0.0/16      via 172.16.0.6        <-- more specific
      10.1.2.0/24      directly connected
      10.1.2.128/25    via 10.1.2.200        <-- MOST specific

  Packet to 10.1.2.130  ->  matches /8, /16, /24, /25  ->  use /25  (via 10.1.2.200)
  Packet to 10.1.2.5    ->  matches /8, /16, /24        ->  use /24  (directly connected)
  Packet to 8.8.8.8     ->  matches /0 only            ->  default route
```

> Aggregation and longest-prefix match coexist: aggregation reduces the **global** table,
> while longest-prefix match allows an **exception** to be punched into an aggregate inside the
> network. This is how CIDR works *and* how it scales. See also §03 on scalability.

---

## 2.5 NAT — Network Address Translation

NAT translates addresses (and usually port numbers) at the boundary between a private network and
the public Internet.

### Variants

| Type | What is translated | Capacity |
|---|---|---|
| **Static NAT** | 1 private IP ↔ 1 public IP, permanently configured | Unlimited (1:1) |
| **Dynamic NAT** | A pool of public IPs, one per internal host at a time | Limited by pool size |
| **NAT / PAT** (NAPT) — *the one everyone means* | Address **and TCP/UDP port** | Thousands → millions per public IP |

### PAT: the reason home broadband works

A typical home router has 192.168.1.0/24 inside and one (or a handful of) public IP outside.
Each NAT table entry is a 5-tuple:

```
  Inside local:port  <--->  Outside global:port   [protocol]   (timeout)
  192.168.1.10:51514 <---> 203.0.113.7:40001    TCP          3600 s
  192.168.1.11:51560 <---> 203.0.113.7:40002    TCP          1800 s
  192.168.1.12:33891 <---> 203.0.113.7:40003    UDP           300 s
```

Since 65 536 ports exist per protocol per public address, and ~28,000 of them are reserved,
a single public IPv4 supports roughly **30 000–60 000 simultaneous NAT sessions**. With typical
port allocations (`32768–60999` on Windows, `49152–65535` on Linux), the usual figure cited is
**~16 384 (16K) simultaneous sessions per public IP** when clients use high source ports.

### NAT's real purpose

NAT was **never** designed as a security feature. Its purpose was to **let an organisation grow
and change its internal addressing without requesting new public address space** (and to
reconcile two opposite design philosophies — see §2.6). It does reduce exposure by accident,
but it is not a firewall and gives no security guarantee.

### NAT problems (a classic exam topic)

| Problem | Explanation |
|---|---|
| **Breaks end-to-end transparency** | Intermediate devices cannot be the endpoints of a connection. This is the **fundamental** architectural objection. |
| **Breaks IPv6** | IPv6 has enough addresses that NAT is **unnecessary and harmful**. Every host can be globally unique, so end-to-end can be restored. |
| **Breaks peer-to-peer / VoIP / P2P / gaming** | Neither side can accept an inbound connection. Requires **STUN/TURN/ICE** and sometimes relay servers. |
| **Port exhaustion** | `200 OK` to a TCP SYN but then nothing — the NAT table is full. Fix: shorter timeouts, more public IPs, carrier-grade NAT. |
| **Checksum problems** | TCP/UDP checksums include the **pseudo-header** (IP addresses). NAT must recompute them; hardware NAT does this correctly, "cheap" NAT offloads it to the host (bad). |
| **Fragmentation of fragmented fragments** | NAT must handle IP fragments, each of which may need translation, and only fragment 0 has the ports. |
| **Hairpinning / NAT loopback** | Two internal hosts cannot reach each other via the public address. Needs a hairpin/NAT loopback feature. |
| **End-to-end IPsec incompatibility** | IKE uses IP addresses in its payloads; NAT traversal (RFC 3948 / RFC 7296 NAT-T) is required. |
| **Logging / accountability** | Many hosts hide behind one IP, making abuse tracing harder. |
| **Slow path** | Translation plus checksum recomputation consumes CPU. |

### NAT traversal summary

| Technique | Mechanism |
|---|---|
| **STUN** | Client asks a public server "what is my public address?" — works only if the client is not symmetric-NATed. |
| **TURN** | Relay through a public server. Always works, but media flows through the server. |
| **ICE** | Orchestrates STUN/TURN candidates to find the best path. |
| **UPnP / NAT-PMP / PCP** | The client asks the router to punch a hole for itself. |
| **Public IPv6** | Removes the problem entirely — the correct answer in 2024+. |

---

## 2.6 The Two Design Philosophies (a classic conceptual question)

Two schools of thought emerged in the 1970s–80s:

**Philosophy A — Classful addressing.** The Internet was a network of networks. Each network got
a **fixed-size class**. Hosts within a network used one address scheme. Routers aggregated by
class boundary, so the routing table stayed small. A network needed a large class *in advance*.

**Philosophy B — Connectionless datagram.** Any host can send to any host; the network is
stateless; addresses are assigned per-host.

The conflict: **Philosophy B needs a huge number of addresses (one per host); Philosophy A needs
few, large, well-structured blocks.**

**Resolution — Classless addressing (CIDR, 1993).** Take Philosophy B's need for addresses, and
give it Philosophy A's *aggregation*: allocate addresses in powers of two and advertise them as
prefixes, so **aggregation** is preserved. CIDR is the compromise that lets both philosophies
coexist — and it is the direct ancestor of IPv6's design (hierarchy of routing domains, size
allocation by need rather than by class).

---

## 2.7 IPv6 Addressing

### 2.7.1 Size and format

* **128 bits** (16 octets) = 2^128 ≈ 3.4 × 10^38 addresses.
* Written as **8 groups of 4 hexadecimal digits**, colon-separated:

```
  2001:0db8:0000:0000:0000:ff00:0042:8329
```

**Compression rules:**

1. **Leading zeros within a group** may be omitted (at most once per group):
   `2001:0db8:0000:0000:0000:ff00:0042:8329` -> `2001:db8::ff00:42:8329`
2. **One run of consecutive all-zero groups** may be replaced by `::` (at most **once** per
   address, to keep it unambiguous):
   `2001:db8:0:0:0:0:0:1` -> `2001:db8::1`
3. `::` is also the **unspecified address** (equivalent to `0.0.0.0`).
4. **IPv4-mapped/compatible** form allows IPv4 in the last 32 bits:
   `::ffff:192.168.1.1` = `::ffff:c0a8:101` = `0000:...:0000:ffff:c0a8:0101`

**`::1` is the IPv6 loopback** (equivalent to `127.0.0.1`).

### 2.7.2 Address scope and types

| Address | Prefix | Purpose |
|---|---|---|
| **Global unicast** | `2000::/3` | Public Internet. Currently `2000::/3` is allocated |
| **Link-local unicast** | `fe80::/10` | Single link only; **never routed**. Auto-configured on every interface; used for routing-protocol next hops (OSPFv3, EIGRP), NDP, DHCPv6 |
| **Unique local (ULA)** | `fc00::/7` (`fd..`) | The IPv6 equivalent of RFC 1918 private space. Randomly generated /48, from the `fd00::/8` block |
| **Multicast** | `ff00::/8` | One-to-many. Replaces broadcast entirely |
| **Loopback** | `::1/128` | Local host |
| **Unspecified** | `::/128` | "No address yet" |
| **Site-local** | `fec0::/10` | **DEPRECATED** — do not use. Superseded by ULA |
| **4over6 / Teredo / 6to4** | `2002::/16`, `2001::/32`, `2001:0000::/32` | Transition mechanisms |

**IPv6 has no broadcast.** Multicast replaces it: all-nodes multicast is `ff02::1`.

### 2.7.3 IPv6 address types (unicast categories — RFC 4291)

* **Interface identifiers (IIDs)** are the low-order 64 bits. Every IPv6 LAN address is
  `64-bit prefix + 64-bit IID`.
* **Link-local**: `fe80::/64` + IID. Default gateway = first router on the link.
* **Anycast**: an address assigned to *multiple* interfaces; the routing system delivers to the
  *nearest* one. Replaces IPv4's "anycast servers" hack. Examples: DNS resolvers, subnet routers.

### 2.7.4 Prefix / subnetting

```
  2001:db8:1234:5600::/64   <-- typical LAN prefix
  |------ 64-bit network ---| |------- 64-bit interface ID -------|

  Subnets under it:  2001:db8:1234:5600:0:0:0:0/64   (subnet 0)
                     2001:db8:1234:5600:0:0:1:0/64   (subnet 1)
                     2001:db8:1234:5600:0:0:2:0/64   (subnet 2)

  You almost always subnet a /48 into /64s — a /64 is the standard LAN size, and every common
  LAN protocol (DHCPv6, SLAAC, ARP-like NDP) assumes exactly /64.
```

IPv6 hands out a **/48** (or larger) from a RIR and the subscriber creates many **/64**s — hence
"there are 2^64 subnets in your /48".

### 2.7.5 SLAAC — Stateless Address Autoconfiguration

**SLAAC (RFC 4862)** lets a host configure its own IPv6 address with **no DHCP server at all**.

**Mechanism 1 — Router Information Solicitation/Advertisement (RISC/RA, ICMPv6):**

```
   Host                                          Router
     |  ICMPv6 Router Solicitation (133)          |
     |  src fe80::1%eth0   dst ff02::2            |
     |-------------------------------------------->|
     |                                             |
     |  ICMPv6 Router Advertisement (134)          |
     |<--------------------------------------------|
     |  src fe80::1   dst ff02::1                  |
     |  hop limit 255                              |
     |  Cur Hop Limit=64                           |
     |  Router Flags: A=1 (autonomous), O=0       |
     |  Router Lifetime=1800 s                     |
     |  Reachable Time=30000 ms  Retrans Timer=1000 |
     |  Prefix Length=64  (on-link)                |
     |  Prefix Option: 2001:db8:1234:5600::/64    |
     |     Autonomous flag = 1                     |
     |     On-link flag     = 1                    |
     |     Valid Lifetime = 2592000 s (30 d)       |
     |     Preferred Lifetime = 604800 s (7 d)     |
```

**Mechanism 2 — the host generates the IID.**

Two methods, both may be used *simultaneously*:

**(a) EUI-64 (modified) — IEEE 802 MAC based.**
```
  MAC:  00:1A:2B:3C:4D:5E

  Insert ff:fe in the middle (flip the Universal/Locally-admin bit):
     00:1A:2B : ff:fe : 3C:4D:5E
  Strip the colons, lowercase:
     001a2bfffe3c4d5e

  Full address:
     2001:db8:1234:5600:0:0:001a:2bff:fe3c:4d5e
                        ^^^^^^^^^^^^ interface identifier (low 64 bits)
```
The bit flip marks the IID as **not globally unique** (it was derived from a MAC, which can be
reused/spoofed).

**(b) Random / temporary addresses (privacy, RFC 4941).**
```
  address = <64-bit prefix> || <random 64 bits generated by the host>
```
Privacy extensions make the IID change periodically, so a device is not tracked by its
identifier over time. **RFC 7217 (stable-privacy)** generates one random IID per prefix and
keeps it stable — the modern recommendation, because it stops long-term tracking while staying
predictable for inbound connections.

**Flags in the Prefix Information Option:**

| Flag | Meaning |
|---|---|
| **On-link (L)** | The prefix is on this link; use it for on-link determination and neighbour discovery |
| **Autonomous (A)** | Hosts may **self-configure** addresses from this prefix (SLAAC) |
| A=1, O=1 | "Managed configuration address" — a **DHCPv6 server** supplies the address |

**DNS in IPv6 — RDNSS (RFC 8106).** Since there is no DHCP server, DNS server addresses are
advertised in an **RDNSS option** inside the Router Advertisement:
```
  Type 25 (RDNSS), Length odd, Lifetime, then IPv6 address(es)
  Type 25, Len 3, Lifetime=1800,  2001:db8::53
```

**Duplicate Address Detection (DAD).** Before using a self-generated address, a host runs DAD
using **Neighbour Discovery**: it sends a **Neighbour Solicitation** for its own tentative
address to `ff02::1` and listens for a **Neighbour Advertisement**. If anyone answers, the
address is in use and a new one is generated. (DAD is an unfortunate acronym in this course.)

**Neighbour Discovery (ND / NDP)** — IPv6's replacement for ARP. ICMPv6-based:

| ND function | ICMPv6 type | Replaces IPv4's |
|---|---|---|
| Router Solicitation | 133 | (ICMP router discovery) |
| Router Advertisement | 134 | |
| **Neighbour Solicitation** | 135 | **ARP request** |
| **Neighbour Advertisement** | 136 | **ARP reply** |
| Redirect | 137 | ICMP redirect |
| **Path MTU Discovery** | Packet Too Big (2) | Fragmentation needed (DF) |

NDP also supports **Router Renegotiation** and uses the **Destination Unreachable / Parameter
Problem** messages.

**IPv6 does not need ARP because it uses multicast**, which lets a host solicit all neighbours at
once instead of broadcasting.

---

## 2.8 IPv4 vs IPv6 — Header Comparison

### IPv4 header (minimum 20 bytes, max 60 with options)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  | DSCP |ECN|          Total Length              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |   Protocol    |       Header Checksum         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Options                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Bits | Purpose |
|---|---|---|
| Version | 4 | 4 = IPv4, 6 = IPv6 |
| **IHL** | 4 | **Header Length** in 32-bit words (5 = 20 bytes, max 15 = 60 bytes) |
| **DSCP** | 6 | **Differentiated Services Code Point** — QoS marking (RFC 2474). Replaced the 3-bit ToS |
| ECN | 2 | Explicit Congestion Notification bits |
| **Total Length** | 16 | Total datagram length in bytes (max 65 535) — needed for fragmentation & buffering |
| **Identification** | 16 | Identifies fragments of the same original datagram |
| **Flags** | 3 | bit0 = reserved, **bit1 = DF** (Don't Fragment), **bit2 = MF** (More Fragments) |
| **Fragment Offset** | 13 | Offset in **8-byte** units; tells where this fragment belongs |
| **TTL** | 8 | Time To Live — max 255 hops; **prevents loops** (§03) |
| **Protocol** | 8 | Next-layer protocol: 1=ICMP, 6=TCP, 17=UDP, 41=IPv6, 89=OSPF |
| **Header Checksum** | 16 | Checksum of the header only (**must be recomputed at every hop**) |
| Source / Destination | 32 each | 32-bit addresses |
| Options | 0–40 bytes | Rarely used today; padded to 4-byte boundary |

### IPv6 header (fixed 40 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               +
+                         Source Address                        +
+                                                               +
|                                                               +
+                                                               +
+                                                               +
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               +
+                         Destination Address                   +
|                                                               +
+                                                               +
|                                                               +
+                                                               +
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Bits | Purpose |
|---|---|---|
| Version | 4 | 6 |
| **Traffic Class** | 8 | DSCP (6) + ECN (2) — *same as IPv4, but **fixed position, never fragmented off*** |
| **Flow Label** | 20 | NEW: identifies a flow so routers can keep state / QoS |
| **Payload Length** | 16 | Length of what follows the 40-byte header (0 = jumbogram with Jumbo option) |
| **Next Header** | 8 | Next protocol **or** the first extension header |
| **Hop Limit** | 8 | Replaces IPv4 TTL (same value, same purpose) |
| Source / Destination | 128 each | IPv6 addresses |

### Simplifications and what replaced them

| IPv4 | IPv6 | Why |
|---|---|---|
| **IHL + options** variable-length | **Next Header + extension headers**, fixed 40 bytes | Routers only read a **fixed 40 bytes** in hardware -> much faster forwarding, and options can be skipped by untrusted nodes |
| **Header Checksum** | **Removed** | IPv4's checksum must be recalculated at every hop — a huge cost in routers. **The link layer and transport layer already have checksums.** Error detection is handled by the receiver |
| **Fragmentation** (fields + any router may fragment) | **Removed** | Fragmentation is expensive and a security risk. Use **Path MTU Discovery** (source fragments only) |
| **Protocol** | **Next Header** | Must be able to point at extension headers |
| **Identification / Flags / Fragment Offset** | **Removed** | No router fragmentation |
| **TTL** | **Hop Limit** | Same function |
| **Address 32 bits** | **Address 128 bits** | Solves exhaustion |
| **Broadcast** | **Removed** -> multicast | Broadcast is inefficient and breaks routing; link-local multicast `ff02::1` replaces it |
| **ARP** | **Neighbour Discovery** (ICMPv6) | IPv6 has no ARP because it can use multicast |
| **Header 20–60 bytes** | **Header fixed 40 bytes** | Predictable, fast hardware processing |

### IPv6 extension headers (in order they appear)

| Order | Header | Purpose |
|---|---|---|
| 1 | **Hop-by-Hop Options** | Must be examined by every router (RFC 8200) |
| 2 | **Routing** | Source-routing / type 0 (deprecated for security) |
| 3 | **Fragment** | Only the source fragments |
| 4 | **Destination Options** | Examined only by the destination (e.g. **RDNSS**) |
| 5 | **AH** (Authentication) | IPsec |
| 6 | **ESP** (Encapsulating Security Payload) | IPsec |
| 7 | **No Next Header** | |
| last | **Destination Options** | Examined only by the ultimate destination |

> Rule: only the **Hop-by-Hop Options** header is processed by every intermediate router.
> Everything else is skipped by routers using the "next header" chain — a design that is both
> faster and safer.

### Jumbo packets

If a payload exceeds 65 535 bytes, the **Payload Length** field is set to 0 and a **Jumbo Payload
option** in the Hop-by-Hop header carries the real length (up to 4 GB). Requires a **jumbo MTU**
(≥ 9 216 B) on every link in the path.

---

## 2.9 Path MTU Discovery

```
ICMPv4 "Destination Unreachable — fragmentation needed and DF set"  (type 3, code 4)
ICMPv6 "Packet Too Big"                                            (type 2)
```

Both carry the **MTU of the next link** in the "unused"/MTU field.

```
  Host (MTU 1500) --> R1 (MTU 1500) --> R2 (MTU 1400) --> R3 (MTU 1300) --> dest

  Host sends 1500-byte packet, DF set.
  R2 cannot forward (1400 < 1500), DF set -> drop + send ICMP "frag needed, MTU=1400".
  Host caches: PMTU to this destination = 1400, caches it in the routing table.
  Host now sends 1400-byte packets (or uses IPv6 fragmentation / smaller segments).
  PMTU is cached; the path may change, so entries expire.
```

**"PMTU black hole"** — if ICMP is filtered, the sender never learns the smaller MTU, packets
vanish, and the connection hangs. This is why blocking all ICMP is a classic and dangerous
firewall mistake.

---

## 2.10 Transition mechanisms (IPv6 readiness)

| Mechanism | Type | Notes |
|---|---|---|
| **Dual stack** | Run both | Simplest and most common. Doubles the attack surface, needs both stacks secured. |
| **Tunnelling** (6in4, 6over4, Teredo, GRE) | Encapsulate | `6in4`: IPv6-in-IPv4 with protocol 41. Teredo (`2001::/32`) tunnels through NAT via UDP 3544. |
| **Protocol translation** (NAT64 / SIIT) | Translate | `NAT64`: `64:ff9b::/96` prefix maps IPv4; DNS64 synthesises AAAA records. Used by mobile operators. |

---

## 2.11 IPv4 vs IPv6 — Summary comparison table

| Feature | IPv4 | IPv6 |
|---|---|---|
| Address size | 32 bits (~4.3 billion) | 128 bits (~3.4 × 10^38) |
| Notation | Dotted decimal | Colon-hex, `::` compression |
| Header size | 20–60 bytes (variable) | Fixed 40 bytes |
| Header checksum | Yes (per hop) | **No** |
| Fragmentation | Routers **and** source | Source **only**, via extension header |
| Broadcast | Yes | **No** -> multicast `ff02::1` |
| Address discovery | **ARP** (separate protocol) | **NDP / ICMPv6** |
| Address configuration | **DHCP** (required) | **SLAAC** (default), DHCPv6 optional |
| Default gateway discovery | ICMP router discovery / DHCP option 3 | **Router Advertisement** (ICMPv6 134) |
| DNS server discovery | DHCP option 6 | **RDNSS** option in RA (or DHCPv6) |
| Type of service | ToS byte (3-bit, legacy) | **Traffic Class** (DSCP + ECN) |
| Flow identification | None (fragile) | **Flow Label** (20 bits) |
| Loop prevention | **TTL** | **Hop Limit** |
| NAT | Essential | **Not needed** — and a design mistake |
| Security | IPsec optional | **IPsec intended to be mandatory at layer 3** |
| Address privacy | Weak (see macOS `fe80` tricks) | Built in (temporary/random addresses) |
| DHCP | Stateful | DHCPv6 (stateful or stateless) / SLAAC |
| Minimum MTU | 576 bytes | **1280 bytes** |
| Subnet size (typical LAN) | /24 (254 hosts) | **/64** (2^64 addresses per LAN) |

---

## 2.12 IPv4 Exhaustion Timeline (revision aid)

| Year | Event |
|---|---|
| 1981 | IPv1 defined (RFC 791); quickly obsolete |
| 1983 | **ARPANET switches to TCP/IP, 1 January 1983** — IPv4 begins |
| 1987 | IANA (RFC 1591) allocates the last Class B block |
| 1989 | RFC 1108 "hurry-up, IPv6" (first attempt) |
| 1993 | **CIDR (RFC 1519)** — de facto; routing table problem solved |
| 1993 | **NAT** commercialised |
| 1993 | "IPv6" defined; 6bone experimental backbone |
| 1994 | **NATS** (RFC 1634) — translation at the protocol level |
| 1996 | **6bone** ends; IPv6 deployed on real networks |
| 1999 | RFC 2460 — IPv6 becomes a standard |
| 2000 | 6bone decommissioned |
| 2002 | **3GPP** mandates IPv6 support |
| 2008 | **IANA** allocates the last /8s to the RIRs |
| **3 Feb 2011** | **IANA free pool exhausted** |
| 2011–2015 | RIR pools run out one by one (RIPE 2012, ARIN 2014, APNIC 2015) |
| **Apr 2017** | **APNIC final RIR pool exhausted** |
| 2017 | RFC 8200 — core IPv6 standard |
| 2018–2025 | NAT64/DNS64, 464XLAT, mobile IPv6-only networks |
| 2024+ | >45% of traffic to Google is IPv6; IPv6-only default for many operators |

---

## 2.13 Worked address-planning exercise

**Brief:** a company has been allocated `2001:db8:8000::/34` (IPv6) and `198.51.100.0/22` (IPv4).
Design the addressing plan.

**IPv4 (from 198.51.100.0/22 = 198.51.100.0 – 198.51.103.255):**

| Subnet | Purpose | Prefix | Range | Gateway | Broadcast |
|---|---|---|---|---|---|
| 1 | HQ LAN (254 hosts) | /24 | 198.51.100.0 – .255 | 198.51.100.1 | 198.51.100.255 |
| 2 | DMZ | /24 | 198.51.101.0 – .255 | 198.51.101.1 | 198.51.101.255 |
| 3 | Server farm | /24 | 198.51.102.0 – .255 | 198.51.102.1 | 198.51.102.255 |
| 4 | Branch LAN | /24 | 198.51.103.0 – .255 | 198.51.103.1 | 198.51.103.255 |

**IPv6 (from 2001:db8:8000::/34):**
Allocate a /36 per site; inside each site use /48s, and /64 per LAN.

```
  2001:db8:8000::/34          (allocated)
  +-- 2001:db8:8000:0000::/48   HQ
  |     +-- 2001:db8:8000:0000:0000::/64   HQ LAN
  |     +-- 2001:db8:8000:0000:0001::/64   HQ DMZ
  |     +-- 2001:db8:8000:0000:0002::/64   HQ Servers
  +-- 2001:db8:8000:0001::/48   Branch
        +-- 2001:db8:8000:0001:0000::/64   Branch LAN
        +-- 2001:db8:8000:0001:0001::/64   Branch WAN
```

Remember: the **first and last /64 in a /64 subnet** (all-zeros and all-ones subnets) are reserved
by RFC 4291 and should not be allocated as LANs.

---

## 2.14 Self-check questions

1. Why is IPv4 address exhaustion *not* just a shortage of numbers? Give the deeper reason.
2. What is the difference between CIDR and NAT, and why does CIDR solve a different problem?
3. Explain VLSM vs FLSM, and compute the prefix needed for a subnet of 100 hosts and for a
   point-to-point link.
4. What is longest-prefix match? Show a table with three overlapping prefixes and say which wins.
5. Put the DHCP message sequence in order and say which are broadcasts.
6. Why does a DHCP relay agent set the GIADDR field, and what does the server use it for?
7. Why was the IPv4 header checksum removed in IPv6? Why was fragmentation removed?
8. What is SLAAC, and what are the On-link and Autonomous flags?
9. Show how the EUI-64 interface ID is derived from the MAC `00:1A:2B:3C:4D:5E`.
10. Explain why IPv6 removed broadcast and how ND replaced ARP.
11. What is Path MTU Discovery, and what is a "PMTU black hole"?
12. List five problems caused by NAT and explain the fundamental one.

---

**Next → [03 — Routing Principles & Issues](03-routing-principles-and-issues.md)**