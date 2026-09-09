# Lab 02: Routing Troubleshooting

## Objective
This lab assumes Layer 1/2/3 addressing is already correct (see the Connectivity Troubleshooting lab for that layer) and focuses entirely on **routing-layer** problems: OSPF neighbor adjacency failures, missing or incorrect route advertisements, and a subtle **administrative distance conflict** where a leftover static route silently overrides a perfectly healthy dynamic route. As before, you'll start from a broken configuration and diagnose it systematically.

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

- Same three-router chain used throughout the Routing Protocols labs — this lab reuses it specifically so you're troubleshooting a familiar shape, with the problems entirely at the routing-configuration layer.
- **Intended result**: PC-A1 (behind R-A) should reach PC-C1 (behind R-C) via OSPF. It currently cannot, for three separate reasons.

---

## IP Addressing (Correct — Not the Source of Any Bug in This Lab)

| Device | Interface | IP Address     | Subnet Mask       |
|--------|-----------|-----------------|---------------------|
| R-A    | Gi0/0     | 192.168.1.1      | 255.255.255.0        |
| R-A    | Gi0/1     | 10.0.12.1          | 255.255.255.252      |
| R-B    | Gi0/0     | 10.0.12.2          | 255.255.255.252      |
| R-B    | Gi0/1     | 10.0.23.2          | 255.255.255.252      |
| R-C    | Gi0/0     | 10.0.23.1          | 255.255.255.252      |
| R-C    | Gi0/1     | 192.168.3.1      | 255.255.255.0        |

---

## Task 1 — Build the Topology and Load the Broken Configuration
Build the topology with the addressing above (already correct — confirm this first with a quick `show ip interface brief` on all three routers so you know Layer 3 addressing isn't in question here). Apply the following broken routing configuration exactly as written:

```
! On R-A
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
```

```
! On R-B
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 1
 network 10.0.23.0 0.0.0.3 area 0
```

```
! On R-C
router ospf 1
 router-id 3.3.3.3
 network 192.168.3.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/1
ip route 192.168.1.0 255.255.255.0 10.0.23.2
```

Do not fix anything yet.

---

## Task 2 — Establish the Symptom
```
! From a host on R-A's LAN (or R-A itself)
ping 192.168.3.1
```
Confirm this fails.

---

## Task 3 — Check Neighbor Adjacencies First
Just as the Connectivity Troubleshooting lab checked physical/interface status before IP addressing, routing troubleshooting should check **neighbor adjacency status** before assuming a route-advertisement problem — a route can't be learned from a neighbor that never formed an adjacency in the first place.

```
! On all three routers
show ip ospf neighbor
```

### Find Bug #1
Confirm R-A and R-B show **no neighbor relationship** at all. Investigate why:
```
! On R-A and R-B
show ip protocols
```
Compare the area assignments — R-A's Gi0/1 is in **area 0**, but R-B's corresponding network statement puts the same 10.0.12.0/30 link in **area 1**. OSPF requires both sides of a link to agree on the area number; this mismatch alone is enough to prevent the adjacency from ever forming, regardless of anything else being correct.

### Fix Bug #1
```
! On R-B
router ospf 1
 no network 10.0.12.0 0.0.0.3 area 1
 network 10.0.12.0 0.0.0.3 area 0
```

### Retest
```
show ip ospf neighbor
```
Confirm R-A and R-B now show a `FULL` adjacency.
```
! From R-A's LAN
ping 192.168.3.1
```
Still fails — more to find.

---

## Task 4 — Verify Route Advertisement, Not Just Adjacency
An adjacency being `FULL` confirms two routers can talk to each other, but doesn't guarantee every intended network is actually being advertised.

```
! On R-C
show ip ospf neighbor
show ip protocols
```

### Find Bug #2
Notice R-C's OSPF configuration only includes a `network` statement for `192.168.3.0` — there is **no** network statement covering the `10.0.23.0/30` link toward R-B at all. This means R-C's Gi0/0 interface isn't even participating in OSPF, so no adjacency can form between R-B and R-C in the first place.
```
show ip ospf neighbor
```
Confirm on R-B that no neighbor relationship exists with R-C, consistent with this finding.

### Fix Bug #2
```
! On R-C
router ospf 1
 network 10.0.23.0 0.0.0.3 area 0
```

### Retest
```
show ip ospf neighbor
```
Confirm R-B and R-C now show `FULL` as well — all three routers should now have the expected adjacencies.
```
! From R-A's LAN
ping 192.168.3.1
```
Still fails, somewhat surprisingly, given both adjacency problems are now fixed — proceed to the next layer of investigation.

---

## Task 5 — Check the Routing Table Directly (Not Just Adjacency)
```
! On R-A
show ip route
```

### Find Bug #3
Look closely at the route to `192.168.3.0/24`. Rather than not appearing at all, you may find it's missing, or — depending on how R-C's leftover static route interacts with what's now being learned via OSPF — you may find unexpected behavior specifically on **R-C's own routing table**:
```
! On R-C
show ip route
show ip route static
```
Notice R-C still has a **static route** to `192.168.1.0/24` (a leftover from before OSPF was working, pointing to 10.0.23.2) with the default administrative distance of 1 — while OSPF's administrative distance is 110. Since 1 is numerically better (more preferred) than 110, **the static route wins**, even now that OSPF has a perfectly valid, healthy path to the same destination. This is a subtle but common real-world problem: a static route that made sense as a temporary or historical fix silently overrides a dynamic route indefinitely, with no error or warning generated anywhere.

### Fix Bug #3
```
! On R-C
no ip route 192.168.1.0 255.255.255.0 10.0.23.2
```

### Retest
```
! On R-C
show ip route
```
Confirm the route to `192.168.1.0/24` is now shown as an OSPF-learned route (`O`), not static.
```
! From R-A's LAN
ping 192.168.3.1
```
Should now succeed.

---

## Task 6 — Full Regression Test
```
! From R-A's LAN
ping 192.168.3.1
traceroute 192.168.3.1

! From R-C's LAN
ping 192.168.1.1
```
All should succeed, with `traceroute` showing the expected R-A → R-B → R-C path.

---

## Task 7 — Final Verification
```
show ip ospf neighbor
show ip route
show ip protocols
show run | section router ospf
show run | include ip route
```
On all three routers, confirming the final state matches expectations.

---

## Verification Checklist

| Bug | Symptom | Root Cause | Fix |
|---|---|---|---|
| #1 | R-A↔R-B adjacency never forms | Area mismatch (area 0 vs. area 1) on the same link | Corrected R-B's network statement to area 0 |
| #2 | R-B↔R-C adjacency never forms | Missing `network` statement for the 10.0.23.0/30 link on R-C | Added the missing network statement |
| #3 | Route to 192.168.1.0/24 on R-C doesn't reflect OSPF despite a healthy adjacency | Leftover static route (AD 1) overriding the OSPF-learned route (AD 110) | Removed the stale static route |
| Final regression test | — | — | Both directions succeed; traceroute confirms expected path |

---

## Verification Checklist (Method)

| Step | Confirms |
|---|---|
| `show ip ospf neighbor` | Adjacency actually formed — checked **before** worrying about specific routes |
| `show ip protocols` | Area assignments, network statements, and other process-level configuration |
| `show ip route` | What's actually installed, and via which source/protocol |
| `show ip route static` | Specifically surfaces any static routes that might be competing with (and silently winning over) a dynamic route |
| `traceroute` | Confirms the real hop-by-hop path, not just that *a* path exists |

---

## Challenge (Optional)
- Reintroduce Bug #3 (a leftover static route with a better administrative distance) in a different topology of your own design, and practice explaining out loud (or in writing) why `show ip route` alone might not immediately reveal the problem unless you specifically think to check for competing static routes — what led you to look there in this lab?
- Deliberately create an OSPF **network type mismatch** instead of an area mismatch (e.g., one side point-to-point, the other broadcast) and diagnose the resulting adjacency failure using `show ip ospf interface` — compare the specific symptom against Bug #1's area mismatch.
- Write a personal "routing troubleshooting checklist" combining this lab's method (adjacency → route advertisement → routing table/AD conflicts → path verification) with the Connectivity Troubleshooting lab's method (physical → addressing → end-device → path verification) into one complete, layered process you could apply to almost any connectivity problem from the ground up.