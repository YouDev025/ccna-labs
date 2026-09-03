# Lab 01: VLAN Basics

## Objective
Create VLANs, assign access ports correctly, configure a trunk link between two switches, and verify both intra-VLAN connectivity and correct inter-VLAN isolation (devices in different VLANs should **not** be able to reach each other without a router — that's expected and correct behavior at this stage, not a bug).

---

## Topology

```
                    +-----------+                    +-----------+
                    |    SW1    |------Trunk----------|    SW2    |
                    +-----------+     (Gi0/1)         +-----------+
                Fa0/1  Fa0/2                       Fa0/1  Fa0/2
                  |      |                            |      |
               +-----+ +-----+                     +-----+ +-----+
               | PC1 | | PC2 |                      | PC3 | | PC4 |
               +-----+ +-----+                     +-----+ +-----+
              VLAN 10  VLAN 20                     VLAN 10  VLAN 20
```

- **SW1** and **SW2** each have one device in VLAN 10 and one in VLAN 20.
- A trunk link between SW1 and SW2 carries both VLANs, so that PC1↔PC3 (both VLAN 10) and PC2↔PC4 (both VLAN 20) can communicate across the two switches — while PC1↔PC2 and PC3↔PC4 (different VLANs on the same switch) should **not** be able to reach each other at all, since no router is present yet.

---

## VLAN and Addressing Plan

| VLAN | Name  | Subnet            |
|------|-------|---------------------|
| 10   | USERS | 192.168.10.0/24      |
| 20   | SALES | 192.168.20.0/24      |

| Device | Switch | Port  | VLAN | IP Address       |
|--------|--------|-------|------|--------------------|
| PC1    | SW1    | Fa0/1 | 10   | 192.168.10.11       |
| PC2    | SW1    | Fa0/2 | 20   | 192.168.20.11       |
| PC3    | SW2    | Fa0/1 | 10   | 192.168.10.12       |
| PC4    | SW2    | Fa0/2 | 20   | 192.168.20.12       |

---

## Tasks

### Task 1 — Build the Topology
1. Connect PC1 and PC2 to SW1 (Fa0/1 and Fa0/2), and PC3 and PC4 to SW2 (Fa0/1 and Fa0/2).
2. Connect SW1 and SW2 together via a third port on each (Gi0/1 in this lab).
3. Apply the IP addressing above to all four PCs, with no default gateway configured yet (there's no router in this lab, so it wouldn't be usable regardless).

### Task 2 — Confirm the Default State (Before VLANs)
```
show vlan brief
```
Confirm that, by default, **all** ports on both switches belong to VLAN 1 — this is the out-of-the-box state before any VLAN configuration is applied.
```
! From PC1
ping 192.168.20.11 (PC2)
```
This will currently succeed if IP addressing alone were the only factor — but note that all four PCs are actually on completely different intended subnets while sharing the same default VLAN, which is exactly the kind of unintentional flat-network design VLANs are meant to prevent. Keep this in mind as your "before" baseline.

### Task 3 — Create VLANs on SW1
```
vlan 10
 name USERS
vlan 20
 name SALES
```

### Task 4 — Create the Same VLANs on SW2
```
vlan 10
 name USERS
vlan 20
 name SALES
```
> VLAN numbers and (ideally) names should match across switches that will trunk the same VLANs together — a mismatch in VLAN *numbering* specifically (e.g., calling it VLAN 10 on one switch and VLAN 15 on another for what's meant to be the same logical group) would cause real connectivity problems, since trunk ports match VLANs by number, not by name.

### Task 5 — Assign Access Ports on SW1
```
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
```

### Task 6 — Assign Access Ports on SW2
```
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
```

### Task 7 — Verify VLAN Assignment
```
show vlan brief
```
Confirm Fa0/1 now shows under VLAN 10 and Fa0/2 under VLAN 20, on both switches.

### Task 8 — Test Connectivity Before Trunking (Expect Partial Failure)
```
! From PC1
ping 192.168.10.12 (PC3)
```
This will **fail** — even though both PC1 and PC3 are correctly in VLAN 10, the link between SW1 and SW2 hasn't been configured to carry VLAN traffic between switches yet (it's likely still in its default access-port state, probably still VLAN 1).

### Task 9 — Configure the Trunk Link
```
! On SW1
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20

! On SW2
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```
> `switchport trunk allowed vlan 10,20` explicitly restricts the trunk to only the VLANs actually in use — a trunk with no restriction allows **all** VLANs by default, which is unnecessary exposure; explicitly limiting it to what's needed is good practice even in a simple lab.

### Task 10 — Verify the Trunk
```
show interfaces trunk
```
Confirm Gi0/1 on both switches shows as a trunk, in trunking mode, carrying VLANs 10 and 20.

### Task 11 — Verify Same-VLAN Connectivity Across Switches
```
! From PC1
ping 192.168.10.12 (PC3)
```
Should now succeed — both are VLAN 10, and the trunk now carries that VLAN's traffic between switches.
```
! From PC2
ping 192.168.20.12 (PC4)
```
Should also succeed — both are VLAN 20.

### Task 12 — Verify Different-VLAN Isolation (Expected Failure)
```
! From PC1
ping 192.168.20.11 (PC2, same switch, different VLAN)
```
```
! From PC3
ping 192.168.20.12 (PC4, same switch, different VLAN)
```
Both should **fail** — this is correct, expected behavior, not a problem to fix. VLANs are separate broadcast domains; without a router (covered in a later inter-VLAN routing lab), devices in different VLANs simply cannot reach each other, by design.

### Task 13 — Document the Topology and Save the Configuration
As specified in the original lab notes, document your topology (a simple diagram or the table format used in this lab is sufficient) and save the running configuration on both switches:
```
copy running-config startup-config
```
Confirm the save completed successfully on both SW1 and SW2.

### Task 14 — Final Verification
```
show vlan brief
show interfaces trunk
show interfaces status
show run | section interface
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 — before VLANs | All ports show under VLAN 1 (default) |
| Task 7 — VLAN assignment | Fa0/1 → VLAN 10, Fa0/2 → VLAN 20, on both switches |
| Task 8 — before trunking | Same-VLAN, cross-switch ping fails |
| Task 10 — trunk configured | Gi0/1 shows as trunking, carrying VLANs 10 and 20, on both switches |
| Task 11 — same-VLAN, cross-switch | PC1↔PC3 and PC2↔PC4 both succeed |
| Task 12 — different-VLAN, same-switch | PC1↔PC2 and PC3↔PC4 both fail (expected, correct behavior) |
| Task 13 — configuration saved | `copy running-config startup-config` completes successfully on both switches |

---

## Challenge (Optional)
- Add a third VLAN (e.g., VLAN 30, "GUEST") with one device on each switch, update the trunk's allowed-VLAN list to include it, and verify both same-VLAN cross-switch connectivity and correct isolation from the other two VLANs.
- Deliberately misconfigure the trunk's allowed-VLAN list on only **one** side (e.g., allow VLAN 10 but not 20 on SW2's trunk while SW1 allows both) and observe/document the resulting asymmetric behavior — PC1↔PC3 still works, but PC2↔PC4 fails, even though both individual switches' access-port assignments are completely correct.
- Research and explain in your lab notes why leaving a trunk's allowed-VLAN list at its default (all VLANs permitted) is generally discouraged in production, even though it technically "just works" without the extra `switchport trunk allowed vlan` command used in this lab.