# Lab: Routing Protocol Redistribution

## Objective
Configure **mutual (two-way) redistribution** between an EIGRP domain and an OSPF domain, correctly seed default metrics for each direction, then deliberately introduce a **second redistribution boundary** to observe the classic **redistribution feedback loop / suboptimal routing problem**, and fix it using **route tagging**. This lab assumes familiarity with basic EIGRP and OSPF configuration from earlier labs in this series.

---

## Topology

```
        EIGRP AS 100 Domain                                      OSPF Area 0 Domain
   192.168.1.0/24 (R-E LAN)                                  192.168.3.0/24 (R-O LAN)
             |                                                          |
           SW-E                                                       SW-O
             |                                                          |
       Gi0/0 | .1                                                Gi0/0 | .1
       +-----------+                                              +-----------+
       |    R-E    |                                              |    R-O    |
       +-----------+                                              +-----------+
        Gi0/1     Gi0/2                                            Gi0/1     Gi0/2
    10.0.1.0/30  10.0.2.0/30                                   10.0.3.0/30  10.0.4.0/30
          |            |                                             |            |
          |            +-----------------+           +---------------+            |
          |                              |           |                            |
    +-----------+                  +-----------+ +-----------+                    |
    |   R-BR1   |------------------|   R-BR2   | |  (R-BR2   |--------------------+
    | (redist.  |   (no direct     | (redist.  | |  continued)|
    |  border)  |    link needed   |  border)  | +-----------+
    +-----------+    between BR1/BR2 for this lab)
```

- **R-E** and **R-O** are the "core" routers of each domain, each with a simple local LAN, used purely to generate real routes worth redistributing and to test end-to-end reachability.
- **R-BR1** is the initial, single redistribution boundary — Part 1 uses only this router.
- **R-BR2** is a **second**, redundant redistribution boundary, added in Part 2 specifically to create the feedback-loop scenario and then fix it.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       | Protocol Domain |
|--------|-----------|-----------------|---------------------|-------------------|
| R-E    | Gi0/0     | 192.168.1.1      | 255.255.255.0        | EIGRP (LAN)        |
| R-E    | Gi0/1     | 10.0.1.1          | 255.255.255.252      | EIGRP (to R-BR1)   |
| R-E    | Gi0/2     | 10.0.2.1          | 255.255.255.252      | EIGRP (to R-BR2)   |
| R-BR1  | Gi0/0     | 10.0.1.2          | 255.255.255.252      | EIGRP              |
| R-BR1  | Gi0/1     | 10.0.3.2          | 255.255.255.252      | OSPF               |
| R-O    | Gi0/0     | 192.168.3.1      | 255.255.255.0        | OSPF (LAN)          |
| R-O    | Gi0/1     | 10.0.3.1          | 255.255.255.252      | OSPF (to R-BR1)     |
| R-O    | Gi0/2     | 10.0.4.1          | 255.255.255.252      | OSPF (to R-BR2)     |
| R-BR2  | Gi0/0     | 10.0.2.2          | 255.255.255.252      | EIGRP               |
| R-BR2  | Gi0/1     | 10.0.4.2          | 255.255.255.252      | OSPF                |

---

## Part 1 — Single-Boundary Mutual Redistribution

### Task 1 — Build the Topology (R-BR1 Only for Now)
1. Place R-E, R-O, R-BR1, and their LANs/switches. **Do not** cable or configure R-BR2 yet — that's introduced in Part 2.
2. Apply the addressing above for R-E, R-BR1, and R-O.

### Task 2 — Configure EIGRP Between R-E and R-BR1
```
! R-E
router eigrp 100
 network 192.168.1.0 0.0.0.255
 network 10.0.1.0 0.0.0.3
 no auto-summary
 passive-interface GigabitEthernet0/0

! R-BR1
router eigrp 100
 network 10.0.1.0 0.0.0.3
 no auto-summary
```

### Task 3 — Configure OSPF Between R-BR1 and R-O
```
! R-BR1
router ospf 1
 router-id 10.10.10.10
 network 10.0.3.0 0.0.0.3 area 0

! R-O
router ospf 1
 router-id 20.20.20.20
 network 192.168.3.0 0.0.0.255 area 0
 network 10.0.3.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
```

### Task 4 — Verify Both Protocols Are Up Independently
```
! On R-BR1
show ip eigrp neighbors
show ip ospf neighbor
```
Confirm both adjacencies are healthy. At this point, R-E and R-O **cannot** reach each other yet — nothing is redistributing between the two domains.

### Task 5 — Redistribute EIGRP into OSPF on R-BR1
```
! On R-BR1
router ospf 1
 redistribute eigrp 100 subnets metric-type 1
```
> `subnets` is required — without it, OSPF only redistributes classful networks, silently dropping any subnetted routes, which in practice means almost everything. `metric-type 1` (E1) makes the redistributed route's cost accumulate normally as it's advertised further into OSPF, rather than staying fixed (E2, the default) — either is valid depending on design intent; E1 is used here so cost differences later in the lab remain visible.

### Task 6 — Redistribute OSPF into EIGRP on R-BR1
EIGRP requires an explicit **seed metric** when redistributing from a protocol that doesn't use EIGRP-style metrics (bandwidth/delay/reliability/load/MTU) — without it, the redistributed routes won't be usable:
```
! On R-BR1
router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500
```
> The five values are bandwidth (kbps), delay (tens of microseconds), reliability, load, and MTU — `10000 100 255 1 1500` is a commonly-used reasonable placeholder set for a lab; in production you'd typically choose values that realistically reflect the redistributed path's actual characteristics.

### Task 7 — Verify Redistribution Is Working
```
! On R-E
show ip route eigrp
```
Confirm `192.168.3.0/24` (R-O's LAN) now appears, marked as an **external EIGRP route (`D EX`)** rather than a normal internal EIGRP route — this distinction matters because external routes have a higher default administrative distance (170) than internal EIGRP routes (90), which becomes relevant in Part 2.
```
! On R-O
show ip route ospf
```
Confirm `192.168.1.0/24` (R-E's LAN) appears as an **external OSPF route (`O E1`)**.

### Task 8 — Verify End-to-End Reachability
```
! From a host on R-E's LAN
ping 192.168.3.1

! From a host on R-O's LAN
ping 192.168.1.1
```
Both should succeed — single-boundary mutual redistribution is now fully functional.

---

## Part 2 — The Redundant-Boundary Feedback Problem

### Task 9 — Add R-BR2 as a Second Redistribution Boundary
1. Cable and address R-BR2 per the table above.
2. Configure EIGRP between R-E and R-BR2, and OSPF between R-BR2 and R-O:
```
! R-E — add the second EIGRP-facing network
router eigrp 100
 network 10.0.2.0 0.0.0.3

! R-BR2
router eigrp 100
 network 10.0.2.0 0.0.0.3
 no auto-summary

router ospf 1
 router-id 30.30.30.30
 network 10.0.4.0 0.0.0.3 area 0

! R-O — add the second OSPF-facing network
router ospf 1
 network 10.0.4.0 0.0.0.3 area 0
```
3. Configure R-BR2 with the **same** mutual redistribution as R-BR1:
```
! R-BR2
router ospf 1
 redistribute eigrp 100 subnets metric-type 1

router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500
```

### Task 10 — Observe the Problem
```
! On R-E
show ip route eigrp
```
Look closely at the route to `192.168.3.0/24`. With **two** redistribution points now active, R-E may learn about this network via more than one path with different (and potentially confusing) metrics — or, in some topologies, a redistributed route can even be **fed back into the same protocol it originated from** via the second boundary, potentially creating a suboptimal or looping path. Use:
```
show ip eigrp topology 192.168.3.0 255.255.255.0
```
and the equivalent OSPF check on the other side, to examine exactly how many paths and sources each domain now sees for the other domain's LAN. Document in your lab notes precisely what you observe — the exact symptom depends somewhat on the specific metrics involved, but the core issue is: **a route that originated in Domain A, left through Boundary 1 into Domain B, can be redistributed right back into Domain A through Boundary 2**, with no inherent way for either domain to know it's looking at a route that's already "theirs."

### Task 11 — Fix the Problem with Route Tagging
Mark routes with a **tag** at the point of redistribution, indicating which domain they originated from, then filter based on that tag to prevent re-injection back into the domain of origin:
```
! On R-BR1 — tag routes coming FROM EIGRP as they enter OSPF
router ospf 1
 redistribute eigrp 100 subnets metric-type 1 tag 100

! On R-BR1 — tag routes coming FROM OSPF as they enter EIGRP
router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500 route-map TAG-FROM-OSPF
route-map TAG-FROM-OSPF permit 10
 set tag 200
```
Apply the same tagging on R-BR2, using the **same** tag values (100 for EIGRP-origin, 200 for OSPF-origin) so both boundary routers agree on what each tag means:
```
! On R-BR2
router ospf 1
 redistribute eigrp 100 subnets metric-type 1 tag 100

router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500 route-map TAG-FROM-OSPF
route-map TAG-FROM-OSPF permit 10
 set tag 200
```

### Task 12 — Block Re-Redistribution of Already-Tagged Routes
On **both** R-BR1 and R-BR2, prevent a route tagged as OSPF-origin (200) from being redistributed back into OSPF, and a route tagged as EIGRP-origin (100) from being redistributed back into EIGRP:
```
! On both R-BR1 and R-BR2
route-map BLOCK-OSPF-ORIGIN-BACK-INTO-OSPF deny 10
 match tag 200
route-map BLOCK-OSPF-ORIGIN-BACK-INTO-OSPF permit 20

router ospf 1
 no redistribute eigrp 100 subnets metric-type 1 tag 100
 redistribute eigrp 100 subnets metric-type 1 tag 100 route-map BLOCK-OSPF-ORIGIN-BACK-INTO-OSPF
```
> This specific route-map, as configured, mainly matters once routes have made a full round trip and come back around with the "wrong" tag for the domain they're about to re-enter — in this lab's simple two-boundary topology, the tagging in Task 11 alone is often sufficient to prevent the most obvious loop, but building the explicit blocking route-map here reflects standard practice for more complex, larger-scale redistribution designs where relying on tagging alone isn't enough.

### Task 13 — Verify the Fix
```
! On R-E
show ip route eigrp
show ip eigrp topology 192.168.3.0 255.255.255.0
```
Confirm the route to `192.168.3.0/24` is now clean — a single, sensible path, without the ambiguity/redundancy observed in Task 10.
```
! From R-E's LAN and R-O's LAN
ping 192.168.3.1
ping 192.168.1.1
```
Both should still succeed.

### Task 14 — Final Verification
```
show ip route
show ip protocols
show run | section router ospf
show run | section router eigrp
show route-map
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 7 — Part 1 redistribution | R-E sees `192.168.3.0/24` as `D EX`; R-O sees `192.168.1.0/24` as `O E1` |
| Task 8 — Part 1 reachability | Both cross-domain pings succeed |
| Task 10 — after adding R-BR2 | Redundant/ambiguous path information observed for the cross-domain route, documented in lab notes |
| Task 11 — tagging applied | `show run` on both boundary routers shows matching tag values for each redistribution direction |
| Task 13 — after fix | Cross-domain routes appear clean and unambiguous again; reachability still works |

---

## Challenge (Optional)
- Adjust the EIGRP seed metric values (Task 6) to make the redistributed path deliberately worse than a hypothetical direct path, and explain in your lab notes why seed metric selection is itself a design decision with real traffic-engineering consequences, not just a mandatory formality.
- Research and briefly describe **why redistributing between more than two routing domains at multiple boundary points is generally discouraged in real network design** unless route filtering/tagging (as practiced in this lab) is carefully planned in advance — reference your own Task 10 observation as supporting evidence.
- Extend this lab with a **distribute-list** (instead of, or in addition to, tagging) that blocks a specific individual prefix from being redistributed in one direction only, and verify the more targeted filtering behavior compared to the broader tag-based approach used in Part 2.