# Glossary

- **AS (Autonomous System) / ASN**: a network under one routing policy, identified by a number
  (for example AS3356). "Origin ASN" is the AS that announces the BGP prefix containing an address.
- **AS boundary**: the point where a path moves from one AS to another. Interconnect addresses
  often come from one side's address space, so the exact border is uncertain (see bdrmap).
- **CGNAT**: carrier-grade NAT. Uses 100.64.0.0/10 (RFC 6598). These addresses usually belong to the user's ISP.
- **CoPP (Control Plane Policing)**: router policy that rate-limits traffic sent to the router's
  CPU, including expired-TTL packets. A main reason intermediate hops "lose" probes that were
  forwarded just fine.
- **Data plane / slow path**: routers forward transit packets in hardware (data plane). They
  generate ICMP replies in software on the CPU (slow path), at low priority.
- **ECMP (Equal-Cost Multi-Path)**: routers spread traffic over several equal paths, usually by
  hashing flow fields (per-flow). With classic traceroute this produces fake "diamonds" and loops.
- **Flow tuple / flow ID**: the header fields a load balancer hashes: source/destination IP and port,
  protocol, TOS, and for ICMP the code and checksum.
- **ICMP Time Exceeded (type 11)**: sent by a router when a packet's TTL reaches zero. Traceroute
  relies on it. **Destination Unreachable (type 3)**, code 3 (port unreachable), marks a UDP probe
  arriving at the target.
- **ICMP deprioritization / rate limiting**: routers limit how fast they generate ICMP errors
  (token bucket). This shows up as loss or delay at that hop only. It does not persist downstream
  and does not affect forwarded traffic.
- **IXP (Internet Exchange Point)**: shared fabric (a peering LAN) where networks interconnect.
  PeeringDB `ixpfx` lists the LAN prefixes, and `netixlan` lists each member's address on the LAN.
- **LSP / LSR**: MPLS label-switched path / label-switching router.
- **MDA (Multipath Detection Algorithm)**: sends probes with deliberately varied flow IDs until a
  statistical stopping rule says all ECMP branches are found. scamper `tracelb`. Not in v0.1.
- **MOAS**: multiple origin AS. One prefix announced by several ASes.
- **MPLS**: label switching inside a network. It can hide hops (pipe mode) or make every hop inside a tunnel
  show the tunnel-exit RTT (ICMP tunneling). RFC 4950 lets routers quote the label stack in ICMP.
- **Paris traceroute**: traceroute that keeps the flow tuple constant across all probes and carries
  the probe ID in fields load balancers ignore (UDP checksum, ICMP sequence with the identifier
  adjusted, TCP sequence). All probes then follow one path.
- **Persistence rule**: real forwarding loss or delay introduced at hop k shows up at hop k **and**
  every later responding hop, including the destination.
- **q-TTL (quoted TTL)**: the TTL of the original probe as quoted inside an ICMP reply. A value > 1
  hints at MPLS.
- **Return path / asymmetry**: replies come back on a path traceroute cannot see, which may differ
  from the forward path. RTT includes both directions.
- **Round**: one complete Paris trace (one probe per TTL). A run is N rounds with the same flow tuple.
- **rDNS**: reverse DNS (PTR) name of an address. Router names often contain location codes (IATA airport codes).
- **Runs test (Wald–Wolfowitz)**: counts runs in a reply/no-reply sequence. Too many runs means
  alternation (rate-limit pattern), and too few means bursts.
- **Token bucket**: rate limiter with refill rate L and bucket size B. Probing faster than L
  produces an on/off reply pattern.
- **TTL**: IP time-to-live (hop limit in IPv6). Traceroute sends probes with TTL 1, 2, 3, … In this
  project, hop number = TTL.
- **Wilson score interval**: a confidence interval for a proportion (for example loss = x/n). It stays
  sensible for small n and for zero losses, unlike the normal approximation. 0/100 lost gives 0–3.7% at 95%.
- **Newcombe interval**: confidence interval for the difference of two proportions, built from
  their Wilson intervals. Used to decide whether one hop's loss is significantly greater than another's.
