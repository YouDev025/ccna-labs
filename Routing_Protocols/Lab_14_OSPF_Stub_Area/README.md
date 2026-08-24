# Lab: OSPF Stub Area

## Objective
This lab isolates **one specific skill** the Multi-Area lab could only partially demonstrate: fully verifying **stub area** behavior, including how it filters **external (Type 5) LSAs** and injects a default route instead. The topology here gives every area a genuine second internal router, so stub behavior can be observed from *inside* the area, not just inferred from the ABR. You'll compare a **stub area** side-by-side against an otherwise-identical **standard (non-stub) area**, then go further with a **totally stubby area**.

---

## Topology

```
                            +-----------+
                            | R-ASBR    |   Injects a simulated external/Internet
                            | (Area 0)  |   route via redistributed static
                            +-----------+
                                  |
                           10.0.0.0/30
                                  |
                            +-----------+
                            |    R-B    |   Area 0 backbone only
                            +-----------+
                            /            \
                    10.0.12.0/30       10.0.23.0/30
                          /                  \
                  +-----------+          +-----------+
                  |    R-A    |          |    R-C    |
                  |  (ABR,    |          |  (ABR,    |
                  | Area 1 =  |          | Area 2 =  |
                  |  STUB)    |          | STANDARD) |
                  +-----------+          +-----------+
                        |                      |
                  10.1.1.0/30            10.2.1.0/30
                        |                      |
                  +-----------+          +-----------+
                  |   R-A2    |          |   R-C2    |
                  | (inside   |          | (inside   |
                  |  Area 1)  |          |  Area 2)  |
                  +-----------+          +-----------+
                        |                      |
                  192.168.1.0/24         192.168.3.0/24
                        |                      |
                     PC-A1                  PC-C1
```

- **R-ASBR** simulates an external connection (e.g., a default route toward "the Internet") and redistributes it into OSPF as an **external route (Type 5 LSA)** — this is the specific thing a stub area is designed to filter out.
- **Area 1** (R-A, R-A2) will be configured as a **stub area**.
- **Area 2** (R-C, R-C2) stays a **standard area** for direct comparison — same structure, different area type, so any difference you observe is attributable to the stub configuration alone.

---

## IP Addressing Table

| Device  | Interface | IP Address     | Subnet Mask       | Area |
|---------|-----------|-----------------|---------------------|------|
| R-ASBR  | Gi0/0     | 10.0.0.1          | 255.255.255.252      | 0    |
| R-B     | Gi0/0     | 10.0.0.2          | 255.255.255.252      | 0    |
| R-B     | Gi0/1     | 10.0.12.2          | 255.255.255.252      | 0    |
| R-B     | Gi0/2     | 10.0.23.2          | 255.255.255.252      | 0    |
| R-A     | Gi0/0     | 10.0.12.1          | 255.255.255.252      | 0    |
| R-A     | Gi0/1     | 10.1.1.1            | 255.255.255.252      | 1    |
| R-A2    | Gi0/0     | 10.1.1.2            | 255.255.255.252      | 1    |
| R-A2    | Gi0/1     | 192.168.1.1      | 255.255.255.0        | 1    |
| R-C     | Gi0/0     | 10.0.23.1          | 255.255.255.252      | 0    |
| R-C     | Gi0/1     | 10.2.1.1            | 255.255.255.252      | 2    |
| R-C2    | Gi0/0     | 10.2.1.2            | 255.255.255.252      | 2    |
| R-C2    | Gi0/1     | 192.168.3.1      | 255.255.255.0        | 2    |
| PC-A1   | NIC       | 192.168.1.10      | 255.255.255.0        | —    |
| PC-C1   | NIC       | 192.168.3.10      | 255.255.255.0        | —    |

---

## Tasks

### Task 1 — Build the Topology and Assign Addressing
1. Place all seven routers, both PCs, and cable exactly as shown.
2. Apply the addressing above and `no shutdown` all router interfaces.

### Task 2 — Configure OSPF on the Backbone and Both ABRs
```
! R-B
router ospf 1
 router-id 2.2.2.2
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0

! R-A (ABR, Area 1)
router ospf 1
 router-id 1.1.1.1
 network 10.0.12.0 0.0.0.3 area 0
 network 10.1.1.0 0.0.0.3 area 1

! R-C (ABR, Area 2)
router ospf 1
 router-id 3.3.3.3
 network 10.0.23.0 0.0.0.3 area 0
 network 10.2.1.0 0.0.0.3 area 2
```

### Task 3 — Configure OSPF on the Internal Routers
```
! R-A2 (fully inside Area 1)
router ospf 1
 router-id 12.12.12.12
 network 10.1.1.0 0.0.0.3 area 1
 network 192.168.1.0 0.0.0.255 area 1
 passive-interface GigabitEthernet0/1

! R-C2 (fully inside Area 2)
router ospf 1
 router-id 32.32.32.32
 network 10.2.1.0 0.0.0.3 area 2
 network 192.168.3.0 0.0.0.255 area 2
 passive-interface GigabitEthernet0/1
```

### Task 4 — Configure R-ASBR and Redistribute an External Route
```
! On R-ASBR
router ospf 1
 router-id 99.99.99.99
 network 10.0.0.0 0.0.0.3 area 0
 redistribute static subnets
ip route 198.51.100.0 255.255.255.0 Null0
```
> The `ip route ... Null0` line creates a simple static route to advertise (representing a simulated external/Internet-facing prefix) without needing a real destination — `redistribute static subnets` then injects it into OSPF as a **Type 5 (external) LSA**, which is exactly the LSA type stub areas are designed to block.

### Task 5 — Verify Baseline Propagation (Before Any Stub Configuration)
On **R-C2** (standard Area 2):
```
show ip route ospf
show ip ospf database external
```
Confirm `198.51.100.0/24` appears as an **`O E2`** (external) route, and the external LSA is visible in the database — this confirms external routes flow normally into a standard area.

On **R-A2** (still standard Area 1 at this point, not yet converted to stub):
```
show ip route ospf
show ip ospf database external
```
Confirm the **same** external route and LSA appear here too — at this point in the lab, both areas behave identically, since neither is a stub yet.

### Task 6 — Convert Area 1 to a Stub Area
All OSPF routers with an interface in Area 1 must agree — that's R-A (the ABR) **and** R-A2 (the internal router):
```
! On R-A
router ospf 1
 area 1 stub

! On R-A2
router ospf 1
 area 1 stub
```

### Task 7 — Verify Stub Behavior on R-A2
```
show ip route ospf
show ip ospf database external
```
Confirm:
- The external route `198.51.100.0/24` is **gone** from R-A2's routing table.
- `show ip ospf database external` returns **no entries** (or an error/empty result depending on platform) — the Type 5 LSA is no longer being flooded into Area 1 at all.
- A **default route** (`O*IA`, via R-A) now appears instead — this is the ABR automatically injecting a default route so Area 1 routers can still reach external destinations, just without the detailed external route table.

### Task 8 — Confirm Area 2 (Standard) Is Unaffected
On **R-C2**:
```
show ip route ospf
show ip ospf database external
```
Confirm the external route and LSA are still present exactly as in Task 5 — Area 2 was never converted to stub, so nothing should have changed here. This side-by-side contrast (Task 7 vs. Task 8) is the core evidence this lab is built to produce.

### Task 9 — Verify End-to-End Reachability Is Still Intact
```
! From PC-A1 and PC-C1
ping 10.0.0.1     (R-ASBR's Area 0 interface, reachable via the default/summary route rather than a specific external entry for Area 1)
```
Both should succeed — Area 1's default route and Area 2's specific external route both provide working reachability, just via different mechanisms.

### Task 10 — Upgrade Area 1 to a Totally Stubby Area
A totally stubby area (Cisco-proprietary) goes further than a standard stub area: it also filters **inter-area (Type 3)** routes from other areas, not just external (Type 5) ones, replacing them too with just the single default route. This command is configured **only on the ABR**:
```
! On R-A only
router ospf 1
 area 1 stub no-summary
```
> Unlike the standard stub conversion in Task 6, `no-summary` is an ABR-only setting — R-A2 does not need (or get) any additional configuration for this step.

### Task 11 — Verify Totally Stubby Behavior
On **R-A2**:
```
show ip route ospf
```
If any inter-area (`O IA`) routes were present before this change (none were explicitly created in this lab's simple topology, but note this for a topology with additional areas), they would now also be replaced by the default route. Confirm the default route is still present and reachability from Task 9 still works.

### Task 12 — Final Verification
```
show ip ospf database
show ip route ospf
show ip protocols
show run | section router ospf
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — before stub conversion | Both R-A2 and R-C2 show the external route and its Type 5 LSA |
| Task 7 — R-A2 after stub conversion | External route and Type 5 LSA both gone; default route present instead |
| Task 8 — R-C2 (standard area, unchanged) | External route and Type 5 LSA still present, exactly as before — confirms the change was isolated to Area 1 |
| Task 9 — reachability | Both PC-A1 and PC-C1 can still reach R-ASBR's address, via different route types |
| Task 11 — totally stubby | Default route still present on R-A2 after `no-summary`; any inter-area routes (in a richer topology) would also be suppressed |

---

## Challenge (Optional)
- Add a third area behind R-B (a genuine second standard area) so the totally stubby area's Type 3 filtering in Task 10/11 can be observed with real inter-area routes present, not just conceptually noted.
- Attempt to configure `area 1 stub` on R-A **without** also configuring it on R-A2, and observe what happens to the R-A↔R-A2 adjacency — document the specific symptom of a stub-flag mismatch between routers in the same area.
- Research and briefly compare a standard **stub area**, a **totally stubby area**, and a **not-so-stubby area (NSSA)** — specifically, explain what problem NSSA solves that a plain stub area cannot (hint: it relates to allowing limited external route redistribution *from within* an otherwise-stub-like area).