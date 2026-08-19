# Lab 02: RIP

## Objective
Configure **RIP version 2** across a three-router topology, correctly advertise networks, disable auto-summary (a near-universal RIPv2 best practice with discontiguous or VLSM addressing), set passive interfaces on LAN segments, and verify route propagation using `show ip rip database`, `show ip route`, and `debug ip rip`.

---

## Topology

```
  192.168.1.0/24                 10.0.12.0/30                 10.0.23.0/30                 192.168.3.0/24
  (Branch A LAN)                (A <-> B link)                (B <-> C link)                (Branch C LAN)
        |                                                                                            |
      SW-A                                                                                          SW-C
        |                                                                                            |
  Gi0/0 | .1                                                                                  Gi0/1 | .1
  +-----------+     Gi0/1      Gi0/0     +-----------+     Gi0/1      Gi0/0     +-----------+
  |    R-A    |-----------------------------|    R-B    |-----------------------------|    R-C    |
  +-----------+     10.0.12.1  10.0.12.2    +-----------+     10.0.23.2  10.0.23.1     +-----------+
        |                                        |
      PC-A1                                 192.168.2.0/24
  192.168.1.10                              (Branch B LAN, Gi0/2 .1)
                                                    |
                                                  SW-B
                                                    |
                                                  PC-B1
                                              192.168.2.10
```

- Same three-router chain used in the Static Routing lab — if you completed that lab, this is a good opportunity to directly compare the manual effort of static routes against RIP's automatic propagation on identical topology and addressing.
- No static routes should be configured in this lab; all inter-network reachability must come from RIP.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       |
|--------|-----------|-----------------|---------------------|
| R-A    | Gi0/0     | 192.168.1.1      | 255.255.255.0        |
| R-A    | Gi0/1     | 10.0.12.1          | 255.255.255.252      |
| R-B    | Gi0/0     | 10.0.12.2          | 255.255.255.252      |
| R-B    | Gi0/1     | 10.0.23.2          | 255.255.255.252      |
| R-B    | Gi0/2     | 192.168.2.1      | 255.255.255.0        |
| R-C    | Gi0/0     | 10.0.23.1          | 255.255.255.252      |
| R-C    | Gi0/1     | 192.168.3.1      | 255.255.255.0        |
| PC-A1  | NIC       | 192.168.1.10      | 255.255.255.0        |
| PC-B1  | NIC       | 192.168.2.10      | 255.255.255.0        |
| PC-C1  | NIC       | 192.168.3.10      | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology and Assign Addressing
1. Place R-A, R-B, R-C, the three switches, and PC-A1, PC-B1, PC-C1 as shown.
2. Apply the addressing above, configure each PC's default gateway to its local router interface, and `no shutdown` all router interfaces.
3. Confirm only directly-connected reachability works so far (e.g., PC-A1 can ping R-A, but not PC-B1 or PC-C1 yet).

### Task 2 — Record the Baseline (No Routing Protocol Yet)
```
show ip route
show ip protocols
```
Confirm `show ip protocols` reports no routing protocol configured, and `show ip route` shows only connected/local routes — this is your "before RIP" baseline.

### Task 3 — Enable RIPv2 on R-A
```
router rip
 version 2
 network 192.168.1.0
 network 10.0.12.0
 no auto-summary
```
> `no auto-summary` disables RIP's default behavior of summarizing routes at classful network boundaries — with `no auto-summary`, RIPv2 advertises the actual subnet masks, which is essential for correct operation on any topology using VLSM or discontiguous subnets (as this one does, with a mix of /24 and /30 networks).

### Task 4 — Enable RIPv2 on R-B
```
router rip
 version 2
 network 10.0.12.0
 network 10.0.23.0
 network 192.168.2.0
 no auto-summary
```

### Task 5 — Enable RIPv2 on R-C
```
router rip
 version 2
 network 10.0.23.0
 network 192.168.3.0
 no auto-summary
```

### Task 6 — Set Passive Interfaces on LAN Segments
RIP has no reason to send periodic route advertisements out an interface where no other router is listening (like each branch's LAN) — doing so wastes bandwidth and unnecessarily exposes routing information to end-host segments:
```
! On R-A
router rip
 passive-interface GigabitEthernet0/0

! On R-B
router rip
 passive-interface GigabitEthernet0/2

! On R-C
router rip
 passive-interface GigabitEthernet0/1
```
> Passive interfaces still have their network **advertised** to RIP neighbors elsewhere — they simply stop sending/receiving RIP updates on that specific interface, which is exactly the behavior wanted for a LAN with no other routers on it.

### Task 7 — Verify RIP Is Running Correctly
```
show ip protocols
```
Confirm each router shows: RIP version 2, the correct list of advertised networks, the correct passive interface, and (once neighbors are up) a list of routing neighbors.

### Task 8 — Verify the RIP Database
```
show ip rip database
```
This shows every route RIP has learned or is advertising, along with its metric (hop count) — compare this against `show ip route` in the next task; the RIP database can contain routes that didn't make it into the actual routing table (e.g., if a better route from another source existed, though that won't happen in this lab since RIP is the only protocol running).

### Task 9 — Verify the Routing Table
```
show ip route
show ip route rip
```
Confirm each router now has **`R`** (RIP) entries for every remote network, each with the correct metric (hop count) and next-hop address.

### Task 10 — Verify End-to-End Reachability
```
! From PC-A1
ping 192.168.2.10
ping 192.168.3.10

! From PC-B1
ping 192.168.1.10
ping 192.168.3.10

! From PC-C1
ping 192.168.1.10
ping 192.168.2.10
```
All six should succeed, confirming RIP has correctly propagated reachability across the entire topology without any static routes.

### Task 11 — Observe RIP Updates in Real Time
```
debug ip rip
```
Wait for the next periodic update cycle (RIP's default update interval is 30 seconds) and observe the advertised networks and metrics being sent/received between routers. Remember to disable the debug afterward:
```
undebug all
```

### Task 12 — Simulate a Link Failure and Observe Convergence
```
! On R-B
interface GigabitEthernet0/1
 shutdown
```
Immediately check:
```
show ip route
```
on R-A and R-C — note that the route to the network beyond the failed link may take up to RIP's full convergence time (which can be significantly longer than link-state protocols, often up to 180 seconds by default for the route to be fully removed via timeout, even though the neighbor relationship itself drops faster) to disappear entirely. Restore the link:
```
interface GigabitEthernet0/1
 no shutdown
```
and confirm reachability is restored once RIP reconverges.

### Task 13 — Final Verification
```
show ip protocols
show ip rip database
show ip route
show run | section router rip
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 baseline | No routing protocol active; only connected/local routes |
| Task 7 — `show ip protocols` | RIPv2 confirmed on all three routers, correct networks and passive interfaces listed |
| Task 8 — `show ip rip database` | Lists all learned/advertised routes with correct hop-count metrics |
| Task 9 — `show ip route` | `R` entries present for every remote network on every router |
| Task 10 — all six inter-branch pings | All succeed |
| Task 12 — link failure | Route to the affected network eventually disappears from `show ip route` (RIP's slower convergence should be visibly noticeable compared to how quickly you'd expect a modern link-state protocol to react) |
| Task 12 — link restored | Reachability fully restored after RIP reconverges |

---

## Challenge (Optional)
- Compare this lab's total configuration effort and convergence behavior directly against the Static Routing lab (same topology/addressing) — write a short comparison in your lab notes covering configuration complexity, scalability if a fourth branch were added, and convergence speed after a failure.
- Add **RIP authentication** (MD5) between R-A and R-B only, and confirm RIP updates between those two routers still succeed while an unauthenticated third router attempting to peer would be rejected (if you have a spare device to test this).
- Research and briefly document why RIP (a distance-vector protocol using simple hop count) is now rarely used in modern production networks compared to link-state protocols like OSPF, referencing at least the convergence-time observation from Task 12 as supporting evidence.