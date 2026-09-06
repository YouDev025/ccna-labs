# Lab: MSTP Configuration

## Objective
Configure **Multiple Spanning Tree Protocol (MSTP)** as the scalable alternative to Rapid PVST+ identified in the previous lab's challenge: instead of one spanning tree instance **per VLAN**, MST maps many VLANs onto a **small number of instances**, retaining load-balancing flexibility with far less overhead as VLAN count grows. This lab covers defining an MST **region**, mapping VLANs to instances, and — critically — what happens when two switches **disagree** on the region definition, since even a tiny mismatch has a real, specific consequence.

---

## Topology

```
                    +-----------+
                    |    SW1    |
                    +-----------+
                    Gi0/1   Gi0/2
                      |         |
                      |         |
                    Gi0/1     Gi0/2
              +-----------+   +-----------+
              |    SW2    |---|    SW3    |
              +-----------+Gi0/1  Gi0/2  +-----------+
```

- Same triangle topology as the STP and Rapid PVST+ labs, now configured with **six** VLANs to make MST's consolidation genuinely meaningful (six separate PVST+ instances collapsed into two MST instances plus the default instance).

---

## VLAN-to-Instance Mapping Plan

| MST Instance | VLANs |
|---|---|
| IST (Instance 0, implicit/default) | Any VLAN not explicitly mapped elsewhere |
| Instance 1 | 10, 20, 30 |
| Instance 2 | 40, 50, 60 |

---

## Tasks

### Task 1 — Build the Topology
1. Recreate the triangle: SW1–SW2, SW2–SW3, SW3–SW1, all trunked.
2. Create VLANs 10, 20, 30, 40, 50, and 60 on all three switches.

### Task 2 — Define the MST Region Identically on All Three Switches
MST region membership is determined by **three** values that must match **exactly** across switches for them to be considered part of the same region: region name, revision number, and the VLAN-to-instance mapping table itself.
```
! On SW1, SW2, and SW3 — identical on all three
spanning-tree mst configuration
 name LAB-MST-REGION
 revision 1
 instance 1 vlan 10,20,30
 instance 2 vlan 40,50,60
```

### Task 3 — Enable MST Mode
```
! On all three switches
spanning-tree mode mst
```

### Task 4 — Verify Region Membership
```
show spanning-tree mst configuration
```
Confirm all three switches show the **identical** name, revision, and instance mapping — this exact match (verified by an internal digest/checksum MST computes from these values, not just a human eyeball comparison) is what places them in the same region.
```
show spanning-tree mst
```
Confirm this shows information for IST (instance 0), Instance 1, and Instance 2 separately — three logical spanning trees total, covering six VLANs, rather than six separate instances as Rapid PVST+ would require.

### Task 5 — Verify Baseline Root Election Per Instance
```
show spanning-tree mst 1
show spanning-tree mst 2
```
Note which switch is currently root for each instance — with default priorities, likely the same switch for both, exactly as PVST+ defaulted to the same root for every VLAN before manual load-balancing configuration.

### Task 6 — Configure Per-Instance Load Balancing
```
! On SW1 — root for Instance 1 (VLANs 10, 20, 30)
spanning-tree mst 1 root primary

! On SW2 — root for Instance 2 (VLANs 40, 50, 60)
spanning-tree mst 2 root primary
```

### Task 7 — Verify Per-Instance Root Election and Load Balancing
```
show spanning-tree mst 1
show spanning-tree mst 2
```
Confirm SW1 is root for Instance 1 and SW2 is root for Instance 2. On SW3, compare the alternate/blocked port for each instance — confirm they differ, exactly as in the Rapid PVST+ lab's per-VLAN load balancing, but now achieved with only **two** instances instead of needing six separate per-VLAN configurations.

---

## Part 2 — Region Mismatch (Understand the Consequence)

### Task 8 — Deliberately Mismatch SW3's Region Configuration
```
! On SW3 only
spanning-tree mst configuration
 revision 2
```
> Note this is a **tiny** change — only the revision number, nothing about the actual VLAN-to-instance mapping — but MST's region-matching digest treats any difference in any of the three region-defining values as a complete mismatch, with no partial credit.

### Task 9 — Verify SW3 Is Now Treated as a Separate Region
```
show spanning-tree mst configuration
```
Confirm SW3's digest/revision no longer matches SW1 and SW2.
```
show spanning-tree mst
```
On SW1 or SW2, look at the boundary ports facing SW3 — confirm they are now treated as **MST region boundary ports**, running a CIST (Common and Internal Spanning Tree) interaction with SW3 as if it were an entirely separate region, rather than participating in the shared internal Instance 1/Instance 2 topology.

### Task 10 — Understand the Practical Impact
In your lab notes, explain what this means operationally: SW3 hasn't lost connectivity outright (the CIST still provides a loop-free topology across region boundaries), but it no longer benefits from MST's fine-grained, per-instance load balancing with SW1 and SW2 — from their perspective, SW3 is now an entire separate region, collapsed to a single logical entity for internal MST purposes, undermining exactly the per-instance flexibility this lab configured in Task 6/7.

### Task 11 — Fix the Mismatch
```
! On SW3
spanning-tree mst configuration
 revision 1
```

### Task 12 — Verify the Fix
```
show spanning-tree mst configuration
show spanning-tree mst
```
Confirm all three switches once again show matching region digests, and SW3's links to SW1/SW2 are no longer treated as region boundaries — the per-instance load balancing from Task 7 should be fully restored.

### Task 13 — Final Verification
```
show spanning-tree mst configuration
show spanning-tree mst
show spanning-tree mst 1
show spanning-tree mst 2
show run | section spanning-tree mst
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 4 — region membership | All three switches show identical name/revision/mapping and are confirmed as one region |
| Task 7 — per-instance load balancing | SW1 root for Instance 1, SW2 root for Instance 2; different alternate ports per instance on SW3 |
| Task 9 — mismatch introduced | SW3 no longer matches; links to SW3 become region boundary ports |
| Task 10 — practical impact | SW3 correctly identified as losing per-instance load-balancing benefit, not losing connectivity entirely |
| Task 12 — mismatch fixed | Region membership restored; per-instance load balancing fully functional again |

---

## Challenge (Optional)
- Add a **third** MST instance (e.g., Instance 3 for a new set of VLANs) and configure a third distinct root bridge for it, extending the load-balancing plan across all three switches and three instances simultaneously.
- Deliberately mismatch the **VLAN-to-instance mapping** itself (rather than just the revision number) between two switches — e.g., map VLAN 30 to Instance 2 on SW3 instead of Instance 1 — and confirm this also produces a region mismatch, demonstrating that all three region-defining values matter equally, not just the revision number tested in this lab.
- Write a short comparison in your lab notes of Rapid PVST+ versus MST for a hypothetical network with 50 VLANs, specifically addressing: how many spanning tree instances each approach would require, the resulting CPU/memory overhead difference, and what load-balancing granularity is lost (if any) by consolidating 50 VLANs into, say, 4 MST instances instead of 50 individual PVST+ trees.