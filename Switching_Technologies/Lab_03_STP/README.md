# Lab 03: STP

## Objective
Build a switched topology with an intentional physical loop, observe **default root bridge election**, manually influence the election to a predictable, intentional root, identify each port's **STP role** (root port, designated port, blocking port), configure **PortFast** and **BPDU Guard** correctly on true edge ports only, and observe traditional STP's **convergence time** after a topology change.

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
                Fa0/1                       Fa0/1
                  |                            |
               +-----+                      +-----+
               | PC1 |                      | PC2 |
               +-----+                      +-----+
```

- **SW1, SW2, and SW3** form a physical triangle: SW1–SW2, SW2–SW3, and SW3–SW1 — a deliberate loop, which STP's entire job is to prevent from becoming a broadcast storm by logically blocking one redundant path.
- **PC1** and **PC2** are ordinary end devices, used later to configure and test PortFast/BPDU Guard correctly.

---

## Tasks

### Task 1 — Build the Topology
1. Connect SW1, SW2, and SW3 in the triangle shown, and connect PC1 to SW2 and PC2 to SW3.
2. All three switches should have VLAN 1 (default) on every port for this lab — no VLAN segmentation is needed to observe STP behavior itself.
3. Confirm all trunk/inter-switch links are up before proceeding — do **not** worry about a broadcast storm occurring during this brief window; STP will begin converging as soon as the links come up.

### Task 2 — Observe the Default Root Bridge Election
```
show spanning-tree vlan 1
```
Run this on all three switches. Identify which one shows `This bridge is the root` — with default priority (32768) on every switch, the root is decided purely by the **lowest MAC address**, since priority alone doesn't break the tie. Record which switch won and its actual Bridge ID (priority + MAC) in your lab notes.

### Task 3 — Identify Port Roles on the Non-Root Switches
On each of the two non-root switches, still using `show spanning-tree vlan 1`:
- Identify the **root port** — the single port with the best (lowest-cost) path back to the root bridge.
- Identify any **designated ports** — ports that are the best path *for their segment* toward the root, and therefore forward traffic.
- Identify the **blocking port** — somewhere in this triangle, exactly one port must be placed in a blocking (or "alternate," in RSTP terminology) state to break the physical loop, since only one path back to the root can be actively forwarding per segment.

Confirm your identification against the `show spanning-tree vlan 1` output's role/status columns directly — don't just infer it from the topology diagram.

### Task 4 — Manually Set a Predictable Root Bridge
Relying on "whichever switch happens to have the lowest MAC address" is not a real design — in production, you choose the root bridge deliberately (typically your most central, highest-capacity core switch). Set **SW1** as the intentional root:
```
! On SW1
spanning-tree vlan 1 root primary
```
> This command doesn't just set an arbitrary lower priority — IOS automatically calculates a priority value low enough to beat the current root, checking the existing topology first. If you need a specific numeric priority instead (e.g., for finer control across many VLANs), you can also configure it directly with `spanning-tree vlan 1 priority <value>` (must be a multiple of 4096).

### Task 5 — Configure a Predictable Secondary Root
```
! On SW2
spanning-tree vlan 1 root secondary
```
> This sets SW2's priority to a value that will make it the root **only if SW1 becomes unavailable** — a deliberate, planned failover hierarchy rather than leaving the secondary election to chance (i.e., whichever remaining switch happens to have the next-lowest MAC address).

### Task 6 — Verify the New Root Bridge
```
show spanning-tree vlan 1
```
On all three switches, confirm SW1 now shows as root, and re-identify each switch's root port, designated ports, and blocking port — note whether the blocking port's location changed compared to Task 3, now that the root bridge itself has changed (it may or may not have, depending on the specific topology and original election — document what you actually observe).

### Task 7 — Configure PortFast on True Edge Ports Only
```
! On SW2 (PC1's port)
interface FastEthernet0/1
 spanning-tree portfast

! On SW3 (PC2's port)
interface FastEthernet0/1
 spanning-tree portfast
```
> PortFast skips the normal STP listening/learning delay (up to 30 seconds by default) for a port, letting an end device get network access almost immediately after the link comes up — but this is only safe on ports that will **never** have a switch or other STP-speaking device connected, since a PortFast port doesn't participate in loop-prevention negotiation the normal way. **Never** configure PortFast on the SW1–SW2, SW2–SW3, or SW3–SW1 inter-switch links.

### Task 8 — Configure BPDU Guard as a Safety Net
```
! On the same two ports
interface FastEthernet0/1
 spanning-tree bpduguard enable
```
> BPDU Guard is what makes PortFast safe to rely on rather than just convenient: if a PortFast-enabled port ever *does* receive a BPDU (meaning someone connected a switch, hub-with-a-switch, or similar to what was assumed to be a simple end-device port), the port is immediately disabled — protecting the topology from an accidental or malicious loop being introduced through a port that was never supposed to see one.

### Task 9 — Verify PortFast and BPDU Guard
```
show spanning-tree interface FastEthernet0/1 detail
```
On both SW2 and SW3, confirm the port shows as a PortFast edge port and BPDU Guard is enabled.
```
! From PC1
ping 192.168.1.1 (or any reachable address on the network)
```
Note (or simply take on faith from the configuration, if not easily timed in your lab platform) that PC1 was able to reach the network essentially immediately after its link came up, unlike the inter-switch ports which underwent normal STP negotiation.

### Task 10 — Observe Convergence Time After a Topology Change
Identify the currently-blocking port from Task 6, then bring down the link on the opposite side (i.e., disable a currently-**forwarding** inter-switch link, forcing STP to recalculate and bring the previously-blocked path into service):
```
! On whichever switch has the forwarding link you're testing
interface GigabitEthernet0/X
 shutdown
```
Immediately begin checking:
```
show spanning-tree vlan 1
```
repeatedly (or use timestamps if your platform logs STP state transitions) on the switch whose port must transition from blocking to forwarding. Note how long this takes — traditional 802.1D STP's listening and learning states each take roughly 15 seconds by default, meaning a full transition can take **up to 30-50 seconds**. Document your actual observed time. Restore the link:
```
interface GigabitEthernet0/X
 no shutdown
```

### Task 11 — Final Verification
```
show spanning-tree vlan 1
show spanning-tree summary
show spanning-tree interface FastEthernet0/1 detail
show run | section spanning-tree
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 — default election | One switch (lowest MAC, default priority) correctly identified as root |
| Task 3 — port roles | Root port, designated port(s), and the single blocking port correctly identified from `show spanning-tree` output, not just inferred |
| Task 6 — manual root election | SW1 confirmed as root; SW2 confirmed as secondary priority |
| Task 9 — PortFast/BPDU Guard | Both confirmed active on PC1's and PC2's ports specifically, and nowhere on the inter-switch links |
| Task 10 — convergence | Blocked port transitions to forwarding after the topology change, taking the expected ~30-50 second range for traditional STP |

---

## Challenge (Optional)
- Enable **Rapid PVST+** (`spanning-tree mode rapid-pvst`) on all three switches and repeat Task 10's convergence test — document the significant difference in convergence time compared to traditional 802.1D STP, and explain conceptually why RSTP achieves this (proposal/agreement handshake instead of fixed timer-based states).
- Deliberately connect a **fourth switch** to one of PC1's or PC2's PortFast+BPDU-Guard-enabled ports (simulating someone plugging an unauthorized switch into what was assumed to be an end-device port) and observe the port immediately enter an err-disabled state — confirming BPDU Guard's protection in a realistic accidental-loop scenario, not just as a theoretical feature.
- Research **UplinkFast** and **BackboneFast** (legacy Cisco-proprietary STP convergence enhancements, largely superseded by Rapid PVST+/RSTP) and explain in your lab notes what specific convergence scenario each was designed to speed up, for historical/conceptual understanding even if you configure Rapid PVST+ instead in modern practice.