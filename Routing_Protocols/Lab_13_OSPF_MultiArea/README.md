# Lab: OSPF Multi-Area

## Objective
Extend single-area OSPF (see Lab 03: OSPF) into a **multi-area** design: configure **Area Border Routers (ABRs)**, observe the difference between **intra-area (`O`)** and **inter-area (`O IA`)** routes, apply **area route summarization** at the ABR to reduce routing table size, and configure a **stub area** to reduce unnecessary LSA flooding into a simple branch area.

---

## Topology

```
        Area 1                          Area 0 (Backbone)                        Area 2
   192.168.1.0/24                                                            192.168.3.0/24
   192.168.11.0/24                                                           192.168.13.0/24
   (Branch A, two LANs)                                                      (Branch C, two LANs)
        |                                                                              |
      SW-A                                                                           SW-C
        |                                                                              |
  Gi0/0,Gi0/3 |                                                                Gi0/1,Gi0/3 |
  +-----------+     Gi0/1      Gi0/0     +-----------+     Gi0/1      Gi0/0     +-----------+
  |    R-A    |-----------------------------|    R-B    |-----------------------------|    R-C   |
  |  (ABR)    |     10.0.12.1  10.0.12.2    +-----------+     10.0.23.2  10.0.23.1     |  (ABR)   |
  +-----------+                          Area 0 only,                                 +-----------+
                                          no local LAN
```

- **R-A** is an ABR: its LAN-facing interfaces belong to **Area 1**, and its link to R-B belongs to **Area 0**.
- **R-B** is a backbone-only router — every one of its interfaces belongs to **Area 0**, connecting the two areas' ABRs together.
- **R-C** is an ABR: its LAN-facing interfaces belong to **Area 2**, and its link to R-B belongs to **Area 0**.
- Each branch area (1 and 2) has **two** LANs specifically so you can practice area **summarization** at the ABR in a meaningful way (summarizing a single /24 wouldn't demonstrate much).

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       | Area |
|--------|-----------|-------------------|--------------------|------|
| R-A    | Gi0/0     | 192.168.1.1        | 255.255.255.0      | 1    |
| R-A    | Gi0/3     | 192.168.11.1        | 255.255.255.0      | 1    |
| R-A    | Gi0/1     | 10.0.12.1            | 255.255.255.252    | 0    |
| R-B    | Gi0/0     | 10.0.12.2            | 255.255.255.252    | 0    |
| R-B    | Gi0/1     | 10.0.23.2            | 255.255.255.252    | 0    |
| R-C    | Gi0/0     | 10.0.23.1            | 255.255.255.252    | 0    |
| R-C    | Gi0/1     | 192.168.3.1        | 255.255.255.0      | 2    |
| R-C    | Gi0/3     | 192.168.13.1        | 255.255.255.0      | 2    |
| PC-A1  | NIC       | 192.168.1.10        | 255.255.255.0      | —    |
| PC-A2  | NIC       | 192.168.11.10        | 255.255.255.0      | —    |
| PC-C1  | NIC       | 192.168.3.10        | 255.255.255.0      | —    |
| PC-C2  | NIC       | 192.168.13.10        | 255.255.255.0      | —    |

> `192.168.1.0/24` and `192.168.11.0/24` (both Area 1) summarize cleanly to `192.168.0.0/22`... but note they are **not** contiguous with each other in a way that summarizes to a tight boundary without also covering unused space — read Task 8 carefully before assuming the "obvious" summary is correct. This is intentional, to build the habit of verifying a summary boundary rather than assuming it.

---

## Tasks

### Task 1 — Build the Topology and Assign Addressing
1. Place R-A, R-B, R-C, both branch switches (each connecting two LANs), and all four PCs as shown.
2. Apply the addressing above and `no shutdown` all router interfaces.
3. Confirm baseline: only directly-connected reachability works so far.

### Task 2 — Configure OSPF on R-A (ABR: Area 1 + Area 0)
```
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 1
 network 192.168.11.0 0.0.0.255 area 1
 network 10.0.12.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
 passive-interface GigabitEthernet0/3
```

### Task 3 — Configure OSPF on R-B (Backbone Only)
```
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
```

### Task 4 — Configure OSPF on R-C (ABR: Area 0 + Area 2)
```
router ospf 1
 router-id 3.3.3.3
 network 10.0.23.0 0.0.0.3 area 0
 network 192.168.3.0 0.0.0.255 area 2
 network 192.168.13.0 0.0.0.255 area 2
 passive-interface GigabitEthernet0/1
 passive-interface GigabitEthernet0/3
```

### Task 5 — Verify Neighbor Formation
```
show ip ospf neighbor
```
Confirm R-A↔R-B and R-B↔R-C both show `FULL` — note there is **no direct adjacency between R-A and R-C**, since they aren't directly connected; all inter-area traffic must transit R-B.

### Task 6 — Verify Area Assignment
```
show ip ospf interface brief
```
Confirm each interface shows the correct area number matching your design — a common configuration mistake is putting an interface in the wrong area, which this command will immediately reveal.

### Task 7 — Verify Intra-Area vs. Inter-Area Routes
On **R-A**:
```
show ip route ospf
```
Confirm the routes to R-C's LANs (192.168.3.0/24 and 192.168.13.0/24) show as **`O IA`** (inter-area), while nothing in this topology on R-A is a same-area OSPF route from R-C's side, since R-A's own LANs are directly connected, not learned via OSPF at all. Compare this against what a router **within** Area 0 itself would see (check `show ip route ospf` on R-B) — R-B should show `O IA` for **both** branch LANs on both sides, since neither is in Area 0 with R-B.

### Task 8 — Verify End-to-End Reachability
```
! From PC-A1 and PC-A2
ping 192.168.3.10
ping 192.168.13.10

! From PC-C1 and PC-C2
ping 192.168.1.10
ping 192.168.11.10
```
All should succeed.

### Task 9 — Configure Area Summarization at R-A
Rather than advertising 192.168.1.0/24 and 192.168.11.0/24 as two separate `O IA` entries into Area 0, summarize them at the ABR. First, verify what summary boundary is actually correct:
- 192.168.1.0/24 = 192.168.**00000001**.0
- 192.168.11.0/24 = 192.168.**00001011**.0

These do **not** share a clean short prefix boundary close together (bit patterns `00000001` vs `00001011` differ starting from a high-order bit), so a tight summary isn't available the way it was for the sequential /24s in the Enhanced Static Routing lab's summarization task. The closest usable summary covering both without excessive extra space is a /20:
```
router ospf 1
 area 1 range 192.168.0.0 255.255.240.0
```
> This deliberately demonstrates that **not every pair of subnets summarizes efficiently** — always check the actual binary boundaries before assuming a summary will be tight, exactly as flagged in the addressing table note above.

### Task 10 — Verify the Summary Took Effect
On **R-B**:
```
show ip route ospf
```
Confirm R-B now sees a **single** `192.168.0.0/20` entry from Area 1 instead of two separate /24 entries. Retest reachability from Task 8 to confirm both original subnets are still fully reachable despite now being represented by one summarized route.

### Task 11 — Convert Area 1 to a Stub Area
Area 1 is a simple branch with no need to receive detailed external route information (there are no external/redistributed routes in this lab, but the concept still applies to any real deployment) — converting it to a stub area reduces the LSA types R-A needs to flood into it, and R-A will inject a default route instead:
```
! On R-A
router ospf 1
 area 1 stub

! On R-B (all routers in Area 1 must agree — but R-B is not in Area 1, so no change needed there; only R-A, as the sole Area 1 router besides the branch itself, needs this)
```
> In this topology, R-A is the only router with interfaces in Area 1, so no other router needs matching stub configuration — in a topology with multiple routers inside the same area, **all** of them must agree on the stub setting or adjacencies will fail to form.

### Task 12 — Verify Stub Area Behavior
```
show ip ospf interface GigabitEthernet0/0
```
Confirm the area is now reported as a stub area. On a router that would be **inside** Area 1 (if you had a second router there), you would also check `show ip route` for a default route (`O*IA`) injected by R-A — since this lab's Area 1 only has R-A itself as its OSPF-speaking router, this specific verification step is conceptual here; note this limitation in your lab notes and describe what you'd expect to see with an additional Area 1 router present.

### Task 13 — Final Verification
```
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip protocols
show run | section router ospf
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — neighbors | R-A↔R-B and R-B↔R-C both `FULL`; no direct R-A↔R-C adjacency |
| Task 6 — area assignment | Each interface reports the correct configured area |
| Task 7 — route types | R-A sees R-C's LANs as `O IA`; R-B sees **both** branch LANs as `O IA` |
| Task 8 — all four cross-area pings | All succeed |
| Task 10 — summarization | R-B's routing table shows one `/20` entry replacing two `/24` `O IA` entries for Area 1; reachability unaffected |
| Task 12 — stub area | R-A's Area 1 interfaces report stub area status |

---

## Challenge (Optional)
- Add a genuine second router inside Area 1 (attached to one of R-A's LANs) running OSPF, and fully verify the stub area behavior from Task 12 that this lab's topology could only describe conceptually — confirm the second router receives a default route rather than specific inter-area routes for Area 2's subnets.
- Convert Area 1 to a **totally stubby area** instead (Cisco-proprietary, configured only on the ABR: `area 1 stub no-summary`) and compare the resulting route count on the hypothetical/added second Area 1 router against the standard stub area configuration from Task 11.
- Deliberately misconfigure one interface into the wrong area (e.g., put R-C's Gi0/0 in Area 1 instead of Area 0) and use `show ip ospf interface brief` and `show ip ospf neighbor` to diagnose why the expected adjacency fails to form — document the specific symptom this produces.