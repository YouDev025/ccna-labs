# Lab 02: Trunking

## Objective
Go beyond the basic trunk setup from the VLAN Basics lab to understand **how** a trunk actually forms: **DTP (Dynamic Trunking Protocol)** negotiation modes and every combination's outcome, and **native VLAN mismatches** — a subtle misconfiguration that doesn't block a trunk from forming at all, but silently causes real problems, and is specifically flagged by CDP with a distinct warning message.

---

## Topology

```
                    +-----------+                    +-----------+
                    |    SW1    |------ Gi0/1 ---------|    SW2    |
                    +-----------+     (test link)      +-----------+
                Fa0/1  Fa0/2                        Fa0/1  Fa0/2
                  |      |                            |      |
               +-----+ +-----+                     +-----+ +-----+
               | PC1 | | PC2 |                      | PC3 | | PC4 |
               +-----+ +-----+                     +-----+ +-----+
              VLAN 10  VLAN 20                     VLAN 10  VLAN 20
```

- Same basic two-switch, two-VLAN shape as the VLAN Basics lab — this lab assumes VLANs 10 and 20 already exist on both switches with the same access-port assignments, and focuses entirely on the Gi0/1 link between them.

---

## VLAN and Addressing Plan (Same as VLAN Basics Lab)

| VLAN | Name  | Subnet            |
|------|-------|---------------------|
| 10   | USERS | 192.168.10.0/24      |
| 20   | SALES | 192.168.20.0/24      |

---

## Tasks

### Task 1 — Build the Topology
Ensure VLANs 10 and 20 exist on both switches, with Fa0/1→VLAN 10 and Fa0/2→VLAN 20 access assignments, exactly as in the VLAN Basics lab. Leave Gi0/1 on both switches at its **default** state for now (do not configure trunking yet).

### Task 2 — Check the Default DTP State
```
show interfaces GigabitEthernet0/1 switchport
```
Confirm the default administrative mode — on most Cisco switches, this defaults to **dynamic auto** or **dynamic desirable** depending on platform/model. Note which one your platform defaults to before proceeding, since it affects what you'll observe in the next task.

### Task 3 — Test DTP Combination 1: Dynamic Auto + Dynamic Auto
```
! On both SW1 and SW2
interface GigabitEthernet0/1
 switchport mode dynamic auto
```
```
show interfaces trunk
show interfaces GigabitEthernet0/1 switchport
```
Confirm **no trunk forms** — `dynamic auto` means "I will become a trunk if the other side actively asks me to, but I will not initiate trunk negotiation myself." With both sides passively waiting for the other to ask, neither ever does, and the link stays in access mode. This is a genuinely common real-world gotcha: two switches at default settings can silently fail to trunk at all.

### Task 4 — Test DTP Combination 2: Dynamic Desirable + Dynamic Auto
```
! On SW1 only
interface GigabitEthernet0/1
 switchport mode dynamic desirable
```
```
show interfaces trunk
```
Confirm a trunk **now forms** — `dynamic desirable` actively initiates negotiation, and SW2's `dynamic auto` responds and agrees, since it's willing to trunk if asked.

### Task 5 — Test DTP Combination 3: Access + Dynamic Desirable
```
! On SW1
interface GigabitEthernet0/1
 switchport mode access
```
```
show interfaces trunk
```
Confirm **no trunk forms**, regardless of SW2's willingness — a port hard-set to `access` mode will never become a trunk through negotiation, since it's not participating in DTP at all in that state. This demonstrates that `access` and `trunk` are absolute, non-negotiated states, while `dynamic auto`/`dynamic desirable` are negotiation postures, not guarantees of a specific outcome.

### Task 6 — Test DTP Combination 4: Trunk + Trunk (No Negotiation)
```
! On both SW1 and SW2
interface GigabitEthernet0/1
 switchport mode trunk
```
```
show interfaces trunk
```
Confirm a trunk forms immediately — both sides are hard-set, no negotiation is even necessary. This is also the **recommended production configuration**: explicit, unambiguous, and immune to the kind of accidental non-trunking seen in Task 3.

### Task 7 — Harden the Trunk Against Further DTP Negotiation
Even with both sides correctly hard-set to `trunk`, DTP frames are still being sent/processed by default unless explicitly disabled — turn this off, since there's no remaining reason to negotiate anything once both sides are manually configured:
```
! On both SW1 and SW2
interface GigabitEthernet0/1
 switchport nonegotiate
```
> Beyond just being unnecessary overhead, leaving DTP active on an already-hard-set trunk is a minor but real security consideration — DTP negotiation has historically been exploitable in certain VLAN-hopping attack scenarios; disabling it once it's no longer needed removes that avenue entirely.

### Task 8 — Set the Allowed VLAN List
```
! On both SW1 and SW2
interface GigabitEthernet0/1
 switchport trunk allowed vlan 10,20
```

### Task 9 — Verify Baseline Trunk Health
```
show interfaces trunk
```
Confirm both switches show Gi0/1 trunking, VLANs 10 and 20 allowed, and no errors.
```
! From PC1
ping 192.168.10.12 (PC3)
```
Should succeed, confirming the trunk is fully functional before moving to the native VLAN task.

---

## Part 2 — Native VLAN Mismatch

### Task 10 — Understand the Native VLAN
On an 802.1Q trunk, one VLAN is designated the **native VLAN** — traffic for that specific VLAN is sent **untagged** across the trunk, while all other VLANs are tagged. By default, this is VLAN 1 on both ends. The two ends of a trunk **must** agree on which VLAN is native, or untagged traffic will be misinterpreted as belonging to whatever VLAN the *receiving* side considers native — not the VLAN it was actually sent from.

### Task 11 — Deliberately Mismatch the Native VLAN
```
! On SW1 only
interface GigabitEthernet0/1
 switchport trunk native vlan 10
```
> Leave SW2's native VLAN at its default (VLAN 1) — this creates the mismatch: SW1 believes VLAN 10 is native, SW2 believes VLAN 1 is native, on the same physical link.

### Task 12 — Observe the CDP Warning
Since both switches are running CDP by default, this specific type of misconfiguration is one CDP is able to detect and flag directly:
```
show logging | include NATIVE_VLAN
```
Look for a message resembling `%CDP-4-NATIVE_VLAN_MISMATCH`, identifying the specific port and the conflicting VLAN IDs on each side. This is a genuinely useful, specific diagnostic message — far more actionable than a generic connectivity failure would be.

### Task 13 — Understand the Practical Impact (Not Just the Warning)
A native VLAN mismatch doesn't necessarily break the trunk itself, but it does cause real, subtle problems: traffic sent untagged from SW1 (which SW1 considers VLAN 10) arrives at SW2 and is treated as VLAN 1 (SW2's native VLAN) — effectively **leaking** VLAN 10 traffic into VLAN 1 unexpectedly, without any error being generated on the data path itself. This is exactly the kind of quiet security/segmentation failure that makes native VLAN mismatches worth checking for specifically, rather than only reacting to obvious connectivity breaks.

### Task 14 — Fix the Mismatch
```
! On SW1
interface GigabitEthernet0/1
 switchport trunk native vlan 1
```
Or, alternatively (and often preferable in production), standardize both switches onto a **dedicated, unused** native VLAN rather than the default VLAN 1 — since VLAN 1 carries various default/control traffic and is a common target in VLAN-hopping attack techniques:
```
! On both SW1 and SW2
vlan 999
 name NATIVE-UNUSED
interface GigabitEthernet0/1
 switchport trunk native vlan 999
```
> Using an unused, dedicated native VLAN (rather than either the default VLAN 1, or one of your actual production VLANs like 10 or 20) is a widely recommended hardening practice — it ensures the native VLAN never carries meaningful production traffic at all, limiting the impact of any future native VLAN mismatch or VLAN-hopping attempt to an intentionally empty VLAN.

### Task 15 — Verify the Fix
```
show logging | include NATIVE_VLAN
```
Confirm no new mismatch warnings appear after making the change (existing historical log entries from Task 12 are expected to remain in the buffer).
```
show interfaces trunk
```
Confirm both switches now agree on the native VLAN.

### Task 16 — Final Verification
```
show interfaces trunk
show interfaces GigabitEthernet0/1 switchport
show cdp neighbors detail
show run interface GigabitEthernet0/1
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — auto/auto | No trunk forms |
| Task 4 — desirable/auto | Trunk forms |
| Task 5 — access/desirable | No trunk forms, regardless of the other side |
| Task 6 — trunk/trunk | Trunk forms immediately, no negotiation needed |
| Task 9 — baseline health | Trunk shows correct allowed VLANs; cross-switch same-VLAN ping succeeds |
| Task 12 — native VLAN mismatch | `%CDP-4-NATIVE_VLAN_MISMATCH` (or platform equivalent) appears in logs, identifying the specific conflict |
| Task 15 — mismatch fixed | No new mismatch warnings; both switches agree on native VLAN |

---

## Challenge (Optional)
- Configure the trunk with `switchport mode dynamic desirable` on **both** sides (rather than the hard-set `trunk`/`trunk` from Task 6) and compare `show interfaces trunk` output — note it still forms a trunk successfully, but discuss in your lab notes why hard-set `trunk`/`trunk` with `nonegotiate` remains the generally recommended production choice over relying on `dynamic desirable` on both ends.
- Research **VLAN hopping** attacks (specifically the "switch spoofing" and "double tagging" techniques) and explain how this lab's `nonegotiate` configuration (Task 7) and dedicated-unused-native-VLAN practice (Task 14) each independently mitigate one of these two specific attack techniques.
- Deliberately allow a VLAN on the trunk's allowed list that doesn't actually exist yet on one of the two switches (e.g., allow VLAN 30 on the trunk before creating VLAN 30 anywhere), and observe/document what `show interfaces trunk` reports for that VLAN's status compared to VLANs 10 and 20, which do exist.