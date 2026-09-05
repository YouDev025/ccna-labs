# Lab: VLAN Trunk Pruning

## Objective
Understand and configure both forms of trunk pruning: **manual pruning** (the `switchport trunk allowed vlan` command from the Trunking lab, reviewed here specifically through the lens of unnecessary broadcast flooding) and **automatic VTP pruning**, which dynamically stops flooding a VLAN's broadcast/multicast/unknown-unicast traffic across a trunk segment where **no downstream switch actually has any active ports in that VLAN** — without requiring any manual per-trunk configuration at all.

---

## Topology

```
                    +-----------+
                    |    SW1    |   VTP Server, VLANs 10, 20, 30 all present
                    +-----------+   with active ports in all three
                         | Gi0/1
                         |
                    +-----------+
                    |    SW2    |   VTP Client, VLANs 10, 20, 30 present
                    +-----------+   with active ports in all three
                         | Gi0/1
                         |
                    +-----------+
                    |    SW3    |   VTP Client, only VLANs 10 and 20
                    +-----------+   have active ports — NO VLAN 30 device
                     Fa0/1 Fa0/2    is connected anywhere on SW3
                       |     |
                    +-----+ +-----+
                    | PC1 | | PC2 |
                    |VLAN | |VLAN |
                    | 10  | | 20  |
                    +-----+ +-----+
```

- **SW1** is the VTP server for this domain; **SW2** and **SW3** are VTP clients.
- Crucially, **SW3 has no device in VLAN 30 at all** — this is what makes VTP pruning meaningful to observe on the SW2–SW3 trunk specifically, while the SW1–SW2 trunk should continue carrying VLAN 30 normally, since SW2 does have an active VLAN 30 port.

---

## VLAN Plan

| VLAN | Name  | Active On       |
|------|-------|------------------|
| 10   | USERS | SW1, SW2, SW3     |
| 20   | SALES | SW1, SW2, SW3     |
| 30   | LAB   | SW1, SW2 only (no active port on SW3) |

---

## Tasks

### Task 1 — Build the Topology
1. Connect SW1–SW2 and SW2–SW3 as shown, with trunk links on both.
2. Create VLANs 10, 20, and 30 on SW1 (as VTP server, these will propagate).
3. Connect PC1 (VLAN 10) and PC2 (VLAN 20) to SW3. Do **not** connect anything to VLAN 30 anywhere on SW3.
4. Add at least one device (or simply leave an access port assigned, for this lab's purposes) in VLAN 30 on SW1 or SW2, so VLAN 30 has genuine active presence upstream.

### Task 2 — Configure VTP
```
! On SW1 (server)
vtp domain LABDOMAIN
vtp mode server
vtp password VtpLabSecret123

! On SW2 and SW3 (clients)
vtp domain LABDOMAIN
vtp mode client
vtp password VtpLabSecret123
```
> A matching domain name and password are required for VTP synchronization to work at all — a mismatch on either would leave client switches unable to learn VLANs from the server.

### Task 3 — Verify VTP Synchronization
```
show vtp status
```
On SW2 and SW3, confirm the VTP configuration revision number matches SW1's, and that VLANs 10, 20, and 30 all appear in `show vlan brief` on both client switches — even though SW3 has no actual device using VLAN 30, VTP still propagates the VLAN's *existence* to every switch in the domain.

### Task 4 — Configure Trunks Between All Three Switches
```
! On the relevant interfaces of SW1, SW2, and SW3
switchport mode trunk
switchport nonegotiate
```
Leave the allowed-VLAN list at its default (all VLANs permitted) for now — this lab specifically wants to observe VTP pruning's *automatic* behavior, which requires trunks not already manually restricted to fewer VLANs than VTP would otherwise prune.

### Task 5 — Observe the Baseline (Before Pruning Is Enabled)
```
! On SW2, checking the SW2-SW3 trunk
show interfaces trunk
```
Confirm VLAN 30 is currently listed as allowed and **not pruned** on the SW2–SW3 trunk, despite SW3 having no active VLAN 30 port — without pruning enabled, VTP has no mechanism to recognize or act on this fact, and broadcast/multicast/unknown-unicast traffic for VLAN 30 is still being flooded across the SW2–SW3 link unnecessarily.

### Task 6 — Enable VTP Pruning
```
! On SW1 (VTP server) — pruning is a domain-wide setting, configured once on the server and propagated
vtp pruning
```

### Task 7 — Verify Pruning Eligibility and Effect
```
! On SW2
show interfaces trunk
```
Look at the **Pruning VLANs list** section of the output specifically (distinct from the "Vlans allowed on trunk" section) — confirm VLAN 30 now appears as actively **pruned** on the SW2–SW3 trunk, since VTP has determined no downstream switch on that trunk needs it.
```
! On SW1, checking the SW1-SW2 trunk
show interfaces trunk
```
Confirm VLAN 30 is **not** pruned on this trunk — SW2 genuinely has an active VLAN 30 port, so this segment correctly continues carrying VLAN 30 traffic normally.

### Task 8 — Understand VLANs That Are Never Pruned by Default
```
show interfaces trunk
```
Note that VLANs 1 and 1002–1005 (default/reserved VLANs) are never eligible for pruning regardless of configuration — this is standard, expected VTP behavior, not a misconfiguration to fix.

### Task 9 — Compare Against Manual Pruning
As a contrast, manually restrict the SW2–SW3 trunk's allowed VLANs instead of relying on VTP's automatic pruning:
```
! On SW2 and SW3, the trunk between them
switchport trunk allowed vlan 10,20
```
```
show interfaces trunk
```
Confirm VLAN 30 is now excluded via the **static allowed list** rather than the **dynamic pruning list**. In your lab notes, explain the practical difference: manual pruning is a fixed, administrator-defined restriction that will **not** automatically adjust if a VLAN 30 device is later added to SW3 (you'd have to remember to update the allowed list yourself), while VTP pruning would have automatically started carrying VLAN 30 across that trunk again the moment a genuine VLAN 30 port became active on SW3, with no manual intervention required.

### Task 10 — Restore Automatic Pruning for the Remainder of the Lab
```
! On SW2 and SW3
no switchport trunk allowed vlan
switchport trunk allowed vlan 10,20,30
```
> Explicitly re-listing all three VLANs restores the trunk to "all relevant VLANs allowed," letting VTP pruning (still enabled domain-wide from Task 6) resume automatically managing which of them actually get flooded across this specific trunk based on real downstream demand.

### Task 11 — Test the Dynamic Adjustment
Add a device to VLAN 30 on SW3 (or simply create an access port assignment there for lab purposes):
```
! On SW3
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 30
```
```
! On SW2, recheck the trunk
show interfaces trunk
```
Confirm VLAN 30 is **no longer pruned** on the SW2–SW3 trunk — VTP pruning detected the new active VLAN 30 presence on SW3 and automatically resumed carrying that VLAN's traffic across the link, with zero manual trunk reconfiguration needed, directly demonstrating the dynamic adaptability manual pruning (Task 9) does not provide.

### Task 12 — Final Verification
```
show vtp status
show interfaces trunk
show vlan brief
show run | section switchport trunk
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — VTP sync | VLANs 10/20/30 appear on all three switches; revision numbers match |
| Task 5 — before pruning | VLAN 30 allowed and unpruned on SW2–SW3 trunk, despite no active VLAN 30 port on SW3 |
| Task 7 — after pruning enabled | VLAN 30 shown as pruned on SW2–SW3 trunk; still active (unpruned) on SW1–SW2 trunk |
| Task 9 — manual pruning comparison | VLAN 30 excluded via static allowed-list instead, functionally similar result but statically fixed |
| Task 11 — dynamic adjustment | VLAN 30 automatically un-pruned on SW2–SW3 trunk immediately after a genuine VLAN 30 port becomes active on SW3, with no trunk reconfiguration |

---

## Challenge (Optional)
- Research why **VTP pruning requires VTP versions/modes that support it** and cannot function in a network using VTP transparent mode or no VTP at all — explain what a network without VTP would need instead to achieve a similar traffic-reduction effect (hint: manual `switchport trunk allowed vlan` pruning, adjusted by hand as needs change).
- Configure `vtp pruning` and then deliberately set a VLAN as **pruning-ineligible** on one trunk (`switchport trunk pruning vlan remove 30`, or platform equivalent) even though VTP pruning is otherwise enabled domain-wide, and confirm that specific VLAN continues to be carried across that trunk regardless of downstream demand — useful for a VLAN you always want flooded everywhere for design reasons, independent of automatic pruning logic.
- Write a short comparison in your lab notes of manual pruning versus VTP pruning across: administrative effort to maintain, behavior when the network topology changes, and risk if VTP itself is misconfigured or a rogue VTP server is introduced (tying back to why many real deployments now prefer VTP transparent mode with careful manual trunk configuration over trusting an active VTP domain at all).