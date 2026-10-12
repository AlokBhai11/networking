# Networking — Course Notes

Detailed teaching notes for the six sections of the syllabus.

| # | File | Contents |
|---|---|---|
| 01 | [Network Layer Design Issues](01-network-layer-design-issues.md) | Store-and-forward, layer-3 services, routing vs forwarding (data vs control plane), congestion control, internetworking |
| 02 | [IPv4 vs IPv6 Concepts](02-ipv4-vs-ipv6.md) | Addressing, exhaustion, NAT/DHCP/CIDR, header simplification, SLAAC |
| 03 | [Routing Principles & Issues](03-routing-principles-and-issues.md) | Metrics, scalability, convergence, routing loops & TTL |
| 04 | [Routing Algorithms](04-routing-algorithms.md) | Distance-vector (Bellman-Ford), link-state (Dijkstra), count-to-infinity, path-vector |
| 05 | [Routing Protocols: RIP & OSPF](05-routing-protocols-rip-ospf.md) | RIP and OSPF in depth, DV vs LS in practice |
| 06 | [Practical Exercise](06-practical-exercise-configuration.md) | Configuring IP, gateway, DNS and security on Cisco IOS, Linux, Windows, plus NAT and hardening |

## Suggested reading order

Sections 01 → 04 build on each other (concepts → algorithms). Section 05 applies 04 to two
real protocols. Section 06 applies 02–05 to actual devices.

## Cross-references between files

- Routing vs forwarding distinction → §01.4, used in §03
- CIDR aggregation → §02.4, ties into scalability in §03.4
- TTL / Hop Limit → §02.8, detailed in §03.6.2
- Count-to-infinity → §04.3, mitigated by RIP's design in §05.2
- OSPF cost formula → §03.3, applied in §05.3.3
- DHCP/SLAAC → §02.3 and §02.7.5, configured in §06.4
- Path MTU Discovery → §02.9, referenced in the ICMP rules of §06.6.3

## Conventions used

- Command-line examples are prefixed by device type (`Router#`, `Switch(config-if)#`,
  `$`, `PS>`).
- IPv4 documentation addresses (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`) are
  reserved for documentation (RFC 5737); `2001:db8::/32` is the IPv6 documentation prefix
  (RFC 3849). No real hosts are referenced.
- Hardware-specific syntax is Cisco IOS unless stated. Linux (`nftables`/`iproute2`) and
  Windows (PowerShell) equivalents are given alongside in section 06.