# Lab: RIPv2 to OSPF Migration

## Objective
Practice a **live, staged migration** from RIPv2 to OSPF across a three-router network — the kind of cutover a real network team would need to perform without a full outage. Rather than simply replacing RIP with OSPF all at once, you will migrate **one router at a time**, using **temporary mutual redistribution** to keep both protocols talking to each other during the transition, and confirm connectivity is **never lost** at any stage. The lab finishes by fully decommissioning RIP once every router has cut over.

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
```

- Same three-router chain and addressing as the RIP and OSPF labs — this lab assumes RIPv2 is already fully running across all three routers exactly as configured in the RIP lab, and walks through converting it to OSPF live, one router at a time.
- Migration order for this lab: **R-A first, then R-C, then R-B last** — migrating the two "edge" routers before the "middle" router is a deliberate, realistic strategy, since it limits the window where the middle router is the only one still depending purely on RIP for full reachability.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       |
|--------|-----------|-----------------|---------------------|
| R-A    | Gi0/0     | 192.168.1.1      | 255.255.255.0        |
| R-A    | Gi0/1     | 10.0.12.1          | 255.255.255.252      |
| R-B    | Gi0/0     | 10.0.12.2          | 255.255.255.252      |
| R-B    | Gi0/1     | 10.0.23.2          | 255.255.255.252      |
| R-C    | Gi0/0     | 10.0.23.1          | 255.255.255.252      |
| R-C    | Gi0/1     | 192.168.3.1      | 255.255.255.0        |

---

## Tasks

### Task 1 — Establish the Starting State (RIPv2 Everywhere)
Configure RIPv2 on all three routers exactly as in the RIP lab, with `no auto-summary` and appropriate passive interfaces:
```
! R-A
router rip
 version 2
 network 192.168.1.0
 network 10.0.12.0
 no auto-summary
 passive-interface GigabitEthernet0/0

! R-B
router rip
 version 2
 network 10.0.12.0
 network 10.0.23.0
 no auto-summary

! R-C
router rip
 version 2
 network 10.0.23.0
 network 192.168.3.0
 no auto-summary
 passive-interface GigabitEthernet0/1
```
Verify full reachability before proceeding:
```
! From a host on R-A's LAN
ping 192.168.3.10 (R-C's LAN host)
```
This should succeed — confirm the network is fully functional under pure RIP before starting the migration.

### Task 2 — Plan the Administrative Distance Strategy
OSPF's default AD (110) is already more preferred than RIP's (120) — this works in your favor during migration: as soon as a router learns a route via **both** protocols, OSPF will automatically win without any manual AD manipulation. This is worth confirming explicitly before you rely on it in the following tasks.

### Task 3 — Enable OSPF on R-A (First Migration Step)
Add OSPF **alongside** the existing RIP configuration — do not remove RIP yet:
```
! On R-A
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
```
At this point, R-A speaks both RIP and OSPF on its Gi0/1 link — but R-B does **not** yet run OSPF, so no OSPF adjacency will form yet. Confirm:
```
show ip ospf neighbor
```
Should show no neighbors yet — this is expected at this stage.

### Task 4 — Enable OSPF on R-C (Second Migration Step)
```
! On R-C
router ospf 1
 router-id 3.3.3.3
 network 192.168.3.0 0.0.0.255 area 0
 network 10.0.23.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/1
```
Same situation as Task 3 — R-B still isn't running OSPF, so still no adjacencies yet. At this point, **both edge routers** are dual-running RIP and OSPF, but neither has an OSPF neighbor, so RIP is still doing all the actual work network-wide. Verify reachability is still intact via RIP alone:
```
ping 192.168.3.10
```
Should still succeed.

### Task 5 — Enable OSPF on R-B (The Critical Cutover Step)
```
! On R-B
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
```
This is the moment both OSPF adjacencies (R-A↔R-B and R-B↔R-C) will form simultaneously, since all three routers now run OSPF.

### Task 6 — Verify OSPF Adjacencies Form and Immediately Test Reachability
```
show ip ospf neighbor
```
Confirm both adjacencies show `FULL`.
```
ping 192.168.3.10
```
This is the key moment to verify carefully — confirm reachability was **never lost** during this transition. Because both RIP and OSPF are now fully operational network-wide simultaneously, and OSPF's AD is better, routers should now be silently preferring OSPF routes over the still-present RIP routes for the same destinations, with no visible disruption.

### Task 7 — Confirm OSPF Has Actually Taken Over (Don't Just Assume)
```
! On R-A
show ip route 192.168.3.0
```
Confirm the active route is now `O` (OSPF), not `R` (RIP) — even though RIP is technically still running and still has a route to the same destination, it should no longer be the **installed** route, since OSPF's better AD wins. Repeat this check on R-B and R-C for their respective remote destinations.

### Task 8 — Run in the Mixed State Briefly and Observe Both Protocols
```
show ip protocols
```
On each router, confirm both RIP and OSPF show as active processes — this "both running, OSPF preferred" state is intentionally where a real migration might sit for a period of time (hours, days, or longer in a real network) while the team confirms full stability before fully decommissioning the old protocol. Use this window to also confirm:
```
show ip route rip
show ip route ospf
```
on each router — RIP routes should still exist in each router's RIP-specific table view even though they're not the active/installed routes.

### Task 9 — Decommission RIP, One Router at a Time
Now remove RIP entirely, again working through the routers in a sensible order (this direction — starting wherever you're most confident — matters less than the R-A/R-C/R-B ordering did during activation, since OSPF is already fully functional and doing all the real work at this point):
```
! On R-A
no router rip
```
```
! On R-B
no router rip
```
```
! On R-C
no router rip
```
After each individual removal, re-verify reachability before moving to the next router:
```
ping 192.168.3.10
```

### Task 10 — Final Verification of a Pure OSPF Network
```
show ip protocols
show ip route
show ip ospf neighbor
show run | include router
```
Confirm `router rip` no longer appears anywhere in any router's configuration, only `router ospf 1` remains, and all previous reachability tests still pass.

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 1 — starting state | Full reachability under pure RIPv2 |
| Task 3–4 — edge routers dual-running | No OSPF neighbors yet; RIP still carrying all traffic; reachability unaffected |
| Task 5–6 — R-B cutover | Both OSPF adjacencies form `FULL`; reachability never interrupted |
| Task 7 — route source verification | `show ip route` confirms `O` (OSPF) is the installed route, not `R` (RIP), on all three routers |
| Task 8 — mixed state | Both protocols show as active in `show ip protocols`; RIP routes still visible via `show ip route rip` even though not installed |
| Task 9 — RIP removal | Reachability confirmed intact after **each individual** router's RIP removal, not just at the very end |
| Task 10 — final state | No `router rip` remains anywhere; pure OSPF network, fully reachable |

---

## Challenge (Optional)
- Repeat this entire migration, but intentionally reverse the order (migrate R-B, the middle router, first) and observe/document why this creates a longer window of potential inconsistency compared to the edge-first order used in this lab — tie your explanation back to the fact R-B is the only transit point between R-A and R-C.
- Research and describe how a **passive-interface** misconfiguration during a migration like this (e.g., forgetting to remove RIP's passive-interface setting appropriately, or applying it incorrectly to OSPF) could create a false sense of successful migration — routes might still work due to the *other* protocol silently covering the gap, masking a real configuration mistake until that other protocol is later removed.
- Design (in writing, without necessarily implementing) a rollback plan for this migration — specifically, what you would need to keep in place or verify before Task 9, so that if OSPF exhibited a problem partway through decommissioning RIP, you could safely revert a given router back to RIP without a service interruption.