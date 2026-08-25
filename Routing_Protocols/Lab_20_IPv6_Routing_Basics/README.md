# Lab: IPv6 Routing Basics

## Objective
Configure and verify **IPv6 static routing** across a three-router topology — the IPv6 counterpart to the IPv4 Static Routing lab. This lab covers IPv6 addressing fundamentals (global unicast vs. link-local), the two ways to specify a static route's next hop, an IPv6 default route, and a specific IPv6-only gotcha: **link-local next-hop addresses require an exit interface to be usable in a static route**, which trips up almost everyone the first time they try it.

---

## Topology

```
  2001:DB8:1::/64                 2001:DB8:12::/64                 2001:DB8:23::/64                 2001:DB8:3::/64
  (Branch A LAN)                 (A <-> B link)                    (B <-> C link)                    (Branch C LAN)
        |                                                                                                     |
      SW-A                                                                                                  SW-C
        |                                                                                                     |
  Gi0/0 | ::1                                                                                          Gi0/1 | ::1
  +-----------+     Gi0/1        Gi0/0     +-----------+     Gi0/1        Gi0/0     +-----------+
  |    R-A    |-------------------------------|    R-B    |-------------------------------|    R-C    |
  +-----------+     2001:DB8:12::1  ::2       +-----------+     2001:DB8:23::2  ::1       +-----------+
```

- Same three-router chain shape as the IPv4 Static Routing lab, re-addressed entirely in IPv6.
- No dynamic IPv6 routing protocol (OSPFv3, EIGRP for IPv6) is used in this lab — every route between non-directly-connected networks must be added manually, exactly as in the IPv4 static lab.

---

## IPv6 Addressing Table

| Device | Interface | IPv6 Address              | Prefix Length |
|--------|-----------|------------------------------|-----------------|
| R-A    | Gi0/0     | 2001:DB8:1::1                 | /64              |
| R-A    | Gi0/1     | 2001:DB8:12::1                | /64              |
| R-B    | Gi0/0     | 2001:DB8:12::2                | /64              |
| R-B    | Gi0/1     | 2001:DB8:23::2                | /64              |
| R-C    | Gi0/0     | 2001:DB8:23::1                | /64              |
| R-C    | Gi0/1     | 2001:DB8:3::1                 | /64              |
| PC-A1  | NIC       | 2001:DB8:1::10 (or SLAAC)      | /64              |
| PC-C1  | NIC       | 2001:DB8:3::10 (or SLAAC)      | /64              |

---

## Tasks

### Task 1 — Build the Topology and Enable IPv6 Routing
1. Place R-A, R-B, R-C, the two switches, and PC-A1/PC-C1 as shown.
2. On **every router**, enable IPv6 unicast routing before configuring any addresses:
   ```
   ipv6 unicast-routing
   ```
3. Apply the global unicast addresses from the table above to every interface, and `no shutdown` each one:
   ```
   interface GigabitEthernet0/0
    ipv6 address 2001:DB8:1::1/64
    no shutdown
   ```
   (repeat for each interface on each router with its own address)

### Task 2 — Understand Link-Local Addresses (Automatic, Always Present)
Every IPv6-enabled interface automatically gets a **link-local address** (`FE80::/10` range) in addition to any global unicast address you configure — this happens automatically and cannot be disabled on most platforms. Check:
```
show ipv6 interface brief
```
Confirm each interface shows **both** a link-local and a global address. You'll need the link-local addresses of R-B's interfaces in Task 4.

### Task 3 — Verify Baseline (Directly Connected Only)
```
show ipv6 route
```
Confirm only `C` (connected) and `L` (local) entries exist on each router — nothing beyond directly-connected segments is reachable yet.
```
! From PC-A1 (or R-A itself, if PCs aren't available in your platform)
ping 2001:DB8:12::2
```
This should succeed (directly connected). A ping to R-C's LAN would fail at this point — confirm this too, as your "before" baseline.

### Task 4 — Add a Static Route Using a Global Unicast Next Hop
This is the more straightforward option, directly equivalent to IPv4 static routing:
```
! On R-A — reach Branch C's LAN via R-B's global address
ipv6 route 2001:DB8:3::/64 2001:DB8:12::2
```
Verify:
```
show ipv6 route static
```
Confirm the route is installed and correctly points to R-B's global address as the next hop.

### Task 5 — Add a Static Route Using a Link-Local Next Hop (the Gotcha)
Link-local addresses are **not globally unique** — the same `FE80::...` address could exist on many different interfaces throughout the network. Because of this, IPv6 static routes using a link-local next hop **must also specify the exit interface**, or the router won't know which of its several links that link-local address is even reachable through. First, find R-B's link-local address on the R-A-facing interface:
```
! On R-B
show ipv6 interface GigabitEthernet0/0
```
Note the `FE80::...` address shown. Now attempt to configure a route using it — **without** an interface, to observe the failure first:
```
! On R-A (this will likely be rejected or behave unpredictably — that's the point)
ipv6 route 2001:DB8:3::/64 FE80::<R-B's link-local ID>
```
Most IOS versions will reject this outright, or the route will fail to resolve properly. Now configure it **correctly**, with the exit interface specified:
```
! On R-A — remove the broken attempt if it was accepted, then:
no ipv6 route 2001:DB8:3::/64 FE80::<R-B's link-local ID>
ipv6 route 2001:DB8:3::/64 GigabitEthernet0/1 FE80::<R-B's link-local ID>
```
Verify:
```
show ipv6 route static
```
Confirm this version installs correctly, now that both the exit interface and the link-local next hop are specified together.

### Task 6 — Choose One Method and Complete the Routing for R-A
Pick either the global-unicast-next-hop method (Task 4) or the interface-plus-link-local method (Task 5) — both are valid; production networks commonly use the link-local method for point-to-point links specifically because it doesn't depend on knowing/tracking the neighbor's global address, but either works correctly. Using your chosen method, add R-A's remaining required route:
```
! R-A also needs a route to the R-B<->R-C link itself (for completeness/troubleshooting visibility), though not strictly required for the LAN-to-LAN reachability tested in this lab
ipv6 route 2001:DB8:23::/64 2001:DB8:12::2
```

### Task 7 — Configure Static Routes on R-B
```
ipv6 route 2001:DB8:1::/64 2001:DB8:12::1
ipv6 route 2001:DB8:3::/64 2001:DB8:23::1
```

### Task 8 — Configure Static Routes on R-C
```
ipv6 route 2001:DB8:1::/64 2001:DB8:23::2
ipv6 route 2001:DB8:12::/64 2001:DB8:23::2
```

### Task 9 — Verify Full Reachability
```
! From PC-A1
ping 2001:DB8:3::10

! From PC-C1
ping 2001:DB8:1::10
```
Both should succeed.

### Task 10 — Configure an IPv6 Default Route (Simplification, R-C Only)
As in the IPv4 lab, R-C only has one way off its local network, making it a good candidate for a default route instead of individual specific routes:
```
! On R-C
no ipv6 route 2001:DB8:1::/64 2001:DB8:23::2
no ipv6 route 2001:DB8:12::/64 2001:DB8:23::2
ipv6 route ::/0 2001:DB8:23::2
```
Retest reachability from Task 9 — it should still succeed, now using a single default route.

### Task 11 — Verify
```
show ipv6 route
show ipv6 route static
show ipv6 interface brief
ping ipv6 2001:DB8:3::10   (alternate syntax on some platforms)
traceroute ipv6 2001:DB8:3::10
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 — link-local addresses | Every interface shows both a link-local and global address automatically |
| Task 3 — baseline | Only connected/local routes exist; cross-branch ping fails |
| Task 4 — global-next-hop static route | Installs correctly |
| Task 5 — link-local static route without interface | Rejected or fails to resolve; with interface specified, installs correctly — the core gotcha this lab is built to demonstrate |
| Task 9 — full reachability | Both cross-branch pings succeed |
| Task 10 — default route on R-C | Reachability still works using a single `::/0` route instead of two specific ones |
| `traceroute ipv6` | Shows the expected hop-by-hop IPv6 path |

---

## Challenge (Optional)
- Configure every static route in this lab using the **link-local + interface** method instead of global unicast next hops, and discuss in your lab notes the practical tradeoff: you no longer need to track/document each neighbor's global address for routing purposes, but the configuration is arguably less immediately readable to someone unfamiliar with the topology.
- Research OSPFv3 (IPv6's OSPF) at a conceptual level and identify which parts of this lab's addressing and topology would carry over directly to a dynamic-routing version of this same network, versus what would need to change.
- Deliberately misconfigure one static route's prefix length (e.g., use `/48` instead of `/64` for a LAN route) and use `show ipv6 route` and `ping`/`traceroute` to diagnose the resulting unexpected behavior — document what you observe.