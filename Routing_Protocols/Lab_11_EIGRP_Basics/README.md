# Lab: EIGRP Basics

## Objective
Configure **EIGRP** (Enhanced Interior Gateway Routing Protocol) across a three-router topology, understand the **autonomous system number**, disable auto-summary, set passive interfaces, and verify neighbor relationships and route propagation using `show ip eigrp neighbors`, `show ip eigrp topology`, and `show ip route eigrp`. This lab also introduces EIGRP's **feasible successor** concept — a key advantage over RIP that contributes to much faster convergence.

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
        |                                        |                                            |
      PC-A1                                 192.168.2.0/24                                  PC-C1
  192.168.1.10                              (Branch B LAN, Gi0/2 .1)                    192.168.3.10
                                                    |
                                                  SW-B
                                                    |
                                                  PC-B1
                                              192.168.2.10
```

- Same three-router chain and addressing used in the Static Routing, RIP, and OSPF labs — this continues the same side-by-side comparison across all four routing approaches on identical infrastructure.
- A direct **R-A ↔ R-C** backup link is added for this lab specifically (see addressing table) so you can observe EIGRP's feasible successor behavior with a genuine alternate path, not just a single chain.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       |
|--------|-----------|-----------------|---------------------|
| R-A    | Gi0/0     | 192.168.1.1      | 255.255.255.0        |
| R-A    | Gi0/1     | 10.0.12.1          | 255.255.255.252      |
| R-A    | Gi0/2     | 10.0.13.1          | 255.255.255.252      |
| R-B    | Gi0/0     | 10.0.12.2          | 255.255.255.252      |
| R-B    | Gi0/1     | 10.0.23.2          | 255.255.255.252      |
| R-B    | Gi0/2     | 192.168.2.1      | 255.255.255.0        |
| R-C    | Gi0/0     | 10.0.23.1          | 255.255.255.252      |
| R-C    | Gi0/1     | 192.168.3.1      | 255.255.255.0        |
| R-C    | Gi0/2     | 10.0.13.2          | 255.255.255.252      |
| PC-A1  | NIC       | 192.168.1.10      | 255.255.255.0        |
| PC-B1  | NIC       | 192.168.2.10      | 255.255.255.0        |
| PC-C1  | NIC       | 192.168.3.10      | 255.255.255.0        |

> The new `10.0.13.0/30` link between R-A and R-C is a **higher-bandwidth/higher-cost or lower-bandwidth** backup path, deliberately less preferred than going via R-B — configure it with a slower interface bandwidth if your platform allows, or simply treat it conceptually as the backup path; EIGRP will naturally prefer the lower-metric path through R-B as long as its bandwidth/delay values are better.

---

## Tasks

### Task 1 — Build the Topology and Assign Addressing
1. Place R-A, R-B, R-C, the three switches, PC-A1, PC-B1, PC-C1, and the new direct R-A↔R-C link, as shown.
2. Apply the addressing above and `no shutdown` all router interfaces.
3. Confirm baseline: only directly-connected reachability works so far.

### Task 2 — Enable EIGRP with an Autonomous System Number
All routers in the same EIGRP domain must use the **same AS number** — it doesn't need to match any real-world AS; it's purely a local identifier that determines which routers will form adjacencies with each other:
```
! R-A
router eigrp 100
 network 192.168.1.0 0.0.0.255
 network 10.0.12.0 0.0.0.3
 network 10.0.13.0 0.0.0.3
 no auto-summary

! R-B
router eigrp 100
 network 10.0.12.0 0.0.0.3
 network 10.0.23.0 0.0.0.3
 network 192.168.2.0 0.0.0.255
 no auto-summary

! R-C
router eigrp 100
 network 10.0.23.0 0.0.0.3
 network 192.168.3.0 0.0.0.255
 network 10.0.13.0 0.0.0.3
 no auto-summary
```
> Like OSPF, EIGRP's `network` command uses a wildcard mask. `no auto-summary` is included for the same VLSM/discontiguous-subnet reasons as in the RIP lab, even though modern EIGRP defaults to no auto-summary on many platforms — setting it explicitly avoids relying on a default that varies by IOS version.

### Task 3 — Set Passive Interfaces on LAN Segments
```
! On R-A
router eigrp 100
 passive-interface GigabitEthernet0/0

! On R-B
router eigrp 100
 passive-interface GigabitEthernet0/2

! On R-C
router eigrp 100
 passive-interface GigabitEthernet0/1
```

### Task 4 — Verify Neighbor Formation
```
show ip eigrp neighbors
```
Confirm each router shows the expected neighbor(s), including R-A and R-C now seeing each other directly over the new backup link, **in addition to** their existing neighbor relationship with R-B. Note the Hold time and uptime columns, which indicate how long the adjacency has been stable.

### Task 5 — Examine the EIGRP Topology Table
```
show ip eigrp topology
```
This is EIGRP-specific and worth understanding carefully: unlike a routing table (which shows only the single best route), the topology table shows **every loop-free path EIGRP has learned** for each destination, along with each one's **feasible distance (FD)** and **reported distance (RD)**. Look specifically for the route to `192.168.3.0/24` (Branch C's LAN) — you should see two paths: one via R-B (the primary, lower-metric path) and one via the direct R-A↔R-C link (the backup path).

### Task 6 — Identify the Successor and Feasible Successor
```
show ip eigrp topology 192.168.3.0 255.255.255.0
```
This detailed view explicitly labels the current **successor** (the route actually installed in the routing table) and, if present, any **feasible successor** — a backup route that EIGRP has already pre-validated as loop-free and can switch to **instantly**, without needing to recompute anything, if the successor fails. Confirm whether the direct R-A↔R-C path qualifies as a feasible successor for reaching Branch C (it will, as long as its reported distance is less than the current successor's feasible distance — the formal feasibility condition).

### Task 7 — Verify the Routing Table
```
show ip route eigrp
```
Confirm only the **successor** route (the single best path) appears in the actual routing table, even though the topology table showed two viable paths — this is the key distinction between the topology table (all viable options) and the routing table (only what's currently in use).

### Task 8 — Verify End-to-End Reachability
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
All six should succeed.

### Task 9 — Test Feasible Successor Failover Speed
```
! On R-B
interface GigabitEthernet0/1
 shutdown
```
Immediately check:
```
show ip route eigrp
show ip eigrp topology
```
on R-A. Because a **feasible successor** was already known (the direct R-A↔R-C link), EIGRP should install it almost instantly — no DUAL recomputation needed, since the backup was already validated as loop-free in Task 6. Compare this failover speed against your OSPF lab's convergence observation, and especially against the RIP lab's — this near-instant switch to a pre-validated backup is one of EIGRP's most significant practical advantages.

Confirm reachability from PC-A1 to PC-C1 is maintained (now via the direct link) throughout the failure. Restore the link:
```
interface GigabitEthernet0/1
 no shutdown
```
and confirm R-B's path resumes as the successor once it's back up (assuming it remains the better metric).

### Task 10 — Verify
```
show ip eigrp neighbors
show ip eigrp topology
show ip route eigrp
show ip protocols
show run | section router eigrp
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 4 — neighbors | R-A, R-B, R-C all form the expected adjacencies, including R-A↔R-C directly |
| Task 5 — topology table | Shows multiple loop-free paths to 192.168.3.0/24, each with FD/RD values |
| Task 6 — successor/feasible successor | Current successor identified (via R-B); direct R-A↔R-C link qualifies as a feasible successor |
| Task 7 — routing table | Only the successor route is installed, despite multiple topology-table entries existing |
| Task 8 — all six inter-branch pings | All succeed |
| Task 9 — link failure | Failover to the feasible successor happens almost immediately, noticeably faster than the RIP lab and at least comparable to (often faster than) the OSPF lab's convergence, since no recomputation is required |
| Task 9 — link restored | R-B's path resumes as successor |

---

## Challenge (Optional)
- Manually adjust interface bandwidth or delay (`bandwidth <kbps>` / `delay <tens of microseconds>`) on the direct R-A↔R-C link to make it deliberately worse, and recheck `show ip eigrp topology` to confirm EIGRP's metric calculation (which factors in bandwidth and delay, not just hop count) reflects the change appropriately.
- Deliberately configure a route on the R-A↔R-C link that would **not** satisfy the feasibility condition (this typically requires a topology where the "backup" path's reported distance is not lower than the successor's feasible distance) and observe the difference in `show ip eigrp topology` output — specifically, that a path failing the feasibility condition is *not* labeled a feasible successor even though it's still a valid loop-free route EIGRP knows about.
- Write a four-way comparison (Static Routing, RIP, OSPF, and this EIGRP lab, same base topology throughout) covering configuration effort, convergence speed, and scalability considerations — using your own observed failover timings from each lab as supporting evidence rather than general claims.