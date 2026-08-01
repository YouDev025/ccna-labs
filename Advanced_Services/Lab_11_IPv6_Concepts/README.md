# Lab 03: IPv6 Concepts

## Objective

This lab introduces advanced IPv6 concepts by configuring a small enterprise network with two routers and two LANs connected over a WAN link. The lab focuses on IPv6 addressing, routing, Neighbor Discovery Protocol (NDP), and connectivity verification.

---

## Topology

```
                    LAN A                                   LAN B

        PC0                     PC1             PC2                     PC3
         |                       |               |                       |
         +-----------+-----------+               +-----------+-----------+
                     |                                       |
                 +--------+                             +--------+
                 |Switch0 |                             |Switch1 |
                 +--------+                             +--------+
                     |                                       |
                 G0/0/0                                 G0/0/0
                 +--------+      Serial Link      +--------+
                 |Router0 |=======================|Router1 |
                 +--------+                       +--------+
                  S0/1/0                           S0/1/0
```

---

## Devices Used

- 2 × Cisco 2911 Routers
- 2 × Cisco 2960 Switches
- 4 × PCs
- 1 × Serial DCE Cable
- Copper Straight-Through Cables

---

## IPv6 Addressing Plan

| Device | Interface | IPv6 Address | Prefix |
|----------|-----------|----------------|--------|
| Router0 | G0/0/0 | 2001:DB8:10::1 | /64 |
| Router0 | S0/1/0 | 2001:DB8:12::1 | /64 |
| Router1 | S0/1/0 | 2001:DB8:12::2 | /64 |
| Router1 | G0/0/0 | 2001:DB8:20::1 | /64 |
| PC0 | Fa0 | 2001:DB8:10::10 | /64 |
| PC1 | Fa0 | 2001:DB8:10::20 | /64 |
| PC2 | Fa0 | 2001:DB8:20::10 | /64 |
| PC3 | Fa0 | 2001:DB8:20::20 | /64 |

### Default Gateways

| Device | Gateway |
|----------|----------------|
| PC0 | 2001:DB8:10::1 |
| PC1 | 2001:DB8:10::1 |
| PC2 | 2001:DB8:20::1 |
| PC3 | 2001:DB8:20::1 |

---

## Tasks

- Build the network topology.
- Configure IPv6 addresses on all router interfaces.
- Enable IPv6 routing.
- Configure static IPv6 routes.
- Configure IPv6 addresses on PCs (or use SLAAC).
- Verify end-to-end IPv6 connectivity.

---

## Router Configuration Summary

### Router0

- Enable IPv6 routing.
- Configure:
  - G0/0/0 → `2001:DB8:10::1/64`
  - S0/1/0 → `2001:DB8:12::1/64`
- Add a static route to `2001:DB8:20::/64`.

### Router1

- Enable IPv6 routing.
- Configure:
  - G0/0/0 → `2001:DB8:20::1/64`
  - S0/1/0 → `2001:DB8:12::2/64`
- Add a static route to `2001:DB8:10::/64`.

---

## Verification Commands

Verify interface status:

```bash
show ipv6 interface brief
```

Display interface information:

```bash
show ipv6 interface
```

View the IPv6 routing table:

```bash
show ipv6 route
```

Display Neighbor Discovery entries:

```bash
show ipv6 neighbors
```

Display IPv6 protocol information:

```bash
show ipv6 protocols
```

Test IPv6 connectivity:

```bash
ping ipv6 2001:DB8:20::1
```

Trace the IPv6 path:

```bash
traceroute ipv6 2001:DB8:20::10
```

---

## PC Connectivity Tests

From **PC0**

```text
ping 2001:DB8:20::10
```

```text
ping 2001:DB8:20::20
```

```text
tracert 2001:DB8:20::10
```

---

## Expected Results

- All interfaces should be **up/up**.
- Routers should exchange IPv6 traffic successfully.
- PCs should communicate across different IPv6 networks.
- The routing table should contain connected and static IPv6 routes.
- Neighbor Discovery should populate the neighbor table after communication.
- Ping and traceroute should complete successfully.

---

## Concepts Covered

- IPv6 Global Unicast Addressing
- Link-Local Addresses
- IPv6 Static Routing
- Neighbor Discovery Protocol (NDP)
- Router Advertisements (RA)
- Stateless Address Autoconfiguration (SLAAC)
- ICMPv6
- IPv6 Routing
- IPv6 Connectivity Verification

---

## Skills Practiced

- Configure IPv6 interfaces.
- Enable IPv6 routing.
- Configure static IPv6 routes.
- Verify Neighbor Discovery.
- Test IPv6 communication.
- Troubleshoot IPv6 connectivity using Cisco IOS commands.

---

## Outcome

After completing this lab, you will be able to deploy a basic IPv6 network, configure routing between multiple IPv6 LANs, understand Neighbor Discovery and Router Advertisements, and verify IPv6 connectivity using Cisco IOS troubleshooting commands.