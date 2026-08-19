# Lab 03: OSPF

## Objective
Configure **single-area OSPF (Area 0)** across a three-router topology, correctly advertise networks, understand **router IDs**, observe **DR/BDR election** on Ethernet segments (including the common surprise of DR/BDR election happening even on a link with only two routers), and verify neighbor relationships and route propagation with `show ip ospf neighbor`, `show ip ospf interface`, and `show ip route`.

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

- Same three-router chain and addressing used in the Static Routing and RIP labs — build this one for a direct three-way comparison of configuration effort, convergence, and verification style across static, distance-vector, and link-state approaches.
- The entire topology lives in a single OSPF area: **Area 0** (the backbone), so no multi-area complexity is introduced here.

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
2. Apply the addressing above, configure each PC's default gateway, and `no shutdown` all router interfaces.
3. Confirm baseline: only directly-connected reachability works so far.

### Task 2 — Set Explicit Router IDs
OSPF will auto-select a router ID (highest loopback, or highest active IP if no loopback exists) if you don't set one — relying on this is a common source of confusion later when a new interface changes the auto-selected ID unexpectedly. Set it explicitly and predictably on every router:
```
! R-A
router ospf 1
 router-id 1.1.1.1

! R-B
router ospf 1
 router-id 2.2.2.2

! R-C
router ospf 1
 router-id 3.3.3.3
```
> Changing the router ID after OSPF has already formed adjacencies typically requires clearing the process (`clear ip ospf process`) to take effect — set it now, before adjacencies form, to avoid that extra step.

### Task 3 — Advertise Networks and Assign Area 0
```
! R-A
router ospf 1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0

! R-B
router ospf 1
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
 network 192.168.2.0 0.0.0.255 area 0

! R-C
router ospf 1
 network 10.0.23.0 0.0.0.3 area 0
 network 192.168.3.0 0.0.0.255 area 0
```
> Note OSPF's `network` command uses a **wildcard mask** (like ACLs), not a regular subnet mask — a common point of confusion for students coming from static routing, where a plain subnet mask is used instead.

### Task 4 — Set Passive Interfaces on LAN Segments
Same rationale as in the RIP lab — there's no reason to send OSPF Hello packets out an interface with no other routers on it:
```
! On R-A
router ospf 1
 passive-interface GigabitEthernet0/0

! On R-B
router ospf 1
 passive-interface GigabitEthernet0/2

! On R-C
router ospf 1
 passive-interface GigabitEthernet0/1
```

### Task 5 — Verify Neighbor Formation
```
show ip ospf neighbor
```
Confirm each router shows the expected neighbor(s) in **FULL** state. If a neighbor is stuck at `2WAY` (on a broadcast/multi-access network with a DR/BDR present) or `EXSTART`/`EXCHANGE` (often an MTU mismatch), that's a real problem to diagnose — `FULL` is the only state that means the adjacency is completely formed and routes are being exchanged.

### Task 6 — Observe DR/BDR Election (the Common Surprise)
Even though each inter-router link here only has **two** routers on it, Ethernet interfaces default to the OSPF **broadcast** network type, which means DR/BDR election still happens — even on a link where a DR is arguably unnecessary, since there are only two routers to synchronize with each other anyway. Check:
```
show ip ospf interface GigabitEthernet0/1
```
on R-A, and the equivalent on R-B/R-C for each link. Note the DR and BDR addresses shown, and which router won each election (OSPF uses router priority, then router ID, as tiebreakers, both configurable).

### Task 7 — (Optional) Eliminate Unnecessary DR/BDR Election on the Point-to-Point Links
Since these inter-router links are genuinely point-to-point (only ever two routers, never a shared multi-access segment), you can explicitly change the OSPF network type to skip DR/BDR election entirely, which is both more efficient and better reflects the actual topology:
```
! On R-A
interface GigabitEthernet0/1
 ip ospf network point-to-point

! On R-B (both relevant interfaces)
interface GigabitEthernet0/0
 ip ospf network point-to-point
interface GigabitEthernet0/1
 ip ospf network point-to-point

! On R-C
interface GigabitEthernet0/0
 ip ospf network point-to-point
```
Retest:
```
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/1
```
Confirm neighbors are still `FULL`, but now **no DR/BDR** is shown at all for these links — since point-to-point network type doesn't elect one.

### Task 8 — Verify the Routing Table
```
show ip route
show ip route ospf
```
Confirm `O` (OSPF) entries exist for every remote network, each showing a cost (metric) rather than RIP's simple hop count.

### Task 9 — Verify End-to-End Reachability
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

### Task 10 — Simulate a Link Failure and Compare Convergence to RIP
```
! On R-B
interface GigabitEthernet0/1
 shutdown
```
Immediately check:
```
show ip route
```
on R-A and R-C. OSPF (a link-state protocol) should detect and propagate this failure **significantly faster** than RIP did in the RIP lab — note the approximate time it takes for the route to disappear from the routing table and compare it directly against your RIP lab observations. Restore the link:
```
interface GigabitEthernet0/1
 no shutdown
```
and confirm reachability and adjacency are restored.

### Task 11 — Verify
```
show ip ospf neighbor
show ip ospf interface brief
show ip protocols
show ip route ospf
show run | section router ospf
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — neighbor states | All expected neighbors show `FULL` |
| Task 6 — DR/BDR on Ethernet links | A DR (and BDR, if applicable) is elected even on the two-router links, demonstrating the default broadcast network-type behavior |
| Task 7 — point-to-point network type | Neighbors remain `FULL`; no DR/BDR shown afterward |
| Task 8 — `show ip route ospf` | `O` entries present for every remote network with a cost value |
| Task 9 — all six inter-branch pings | All succeed |
| Task 10 — link failure | Route disappears from the routing table noticeably faster than the RIP lab's equivalent test |
| Task 10 — link restored | Adjacency and reachability both fully restored |
| `show ip protocols` | Confirms OSPF process ID, router ID, and area assignment on each router |

---

## Challenge (Optional)
- Change OSPF interface **cost** manually on one path (`ip ospf cost <value>`) to make a deliberately longer physical path preferred instead, and confirm with `show ip route` that OSPF now selects the manually-preferred path — demonstrating how cost, not just hop count, drives path selection.
- Add a **second area** (e.g., make Branch C's LAN Area 3 instead of Area 0, with R-C as an ABR) if your course has covered multi-area OSPF, and observe the resulting `show ip ospf neighbor` and `show ip route` differences, including the appearance of inter-area (`O IA`) routes.
- Write a three-way comparison in your lab notes (Static Routing lab vs. RIP lab vs. this OSPF lab, same topology throughout) covering configuration effort, scalability to a hypothetical fourth branch, and convergence speed — using your own Task 10 timing observations as evidence for the convergence comparison specifically.