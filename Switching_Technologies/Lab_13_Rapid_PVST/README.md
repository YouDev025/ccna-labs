# Lab: Rapid PVST+

## Objective
Building on the STP lab's basic triangle topology, this lab covers **Rapid PVST+** (Cisco's per-VLAN implementation of RSTP): the new **port roles and states** RSTP introduces, why it converges dramatically faster than 802.1D (the proposal/agreement handshake, not just shorter timers), and PVST+'s specific advantage over plain RSTP — **per-VLAN** spanning tree instances, allowing you to load-balance traffic across redundant links by making different switches root for different VLANs.

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
                Fa0/1  Fa0/2                Fa0/1  Fa0/2
                  |      |                    |      |
               +-----+ +-----+             +-----+ +-----+
               | PC1 | | PC2 |             | PC3 | | PC4 |
               +-----+ +-----+             +-----+ +-----+
              VLAN 10  VLAN 20            VLAN 10  VLAN 20
```

- Same triangle physical topology as the 802.1D STP lab, now with **two VLANs** (10 and 20) present on all three switches — this is what makes PVST+'s per-VLAN behavior meaningful to demonstrate, since a single-instance protocol (plain RSTP, or 802.1D) would have to pick just **one** root bridge for the entire network regardless of VLAN.

---

## Tasks

### Task 1 — Build the Topology
1. Recreate the triangle from the STP lab: SW1–SW2, SW2–SW3, SW3–SW1.
2. Create VLANs 10 and 20 on all three switches, with PC1/PC3 in VLAN 10 and PC2/PC4 in VLAN 20 as shown.
3. Trunk all three inter-switch links, carrying both VLANs.
4. Confirm the switches are currently running default 802.1D STP (or whatever your platform's default is) before converting.

### Task 2 — Enable Rapid PVST+
```
! On all three switches
spanning-tree mode rapid-pvst
```

### Task 3 — Verify the Mode Change
```
show spanning-tree summary
```
Confirm all three switches now report `rapid-pvst` as the active mode, and that this exists as a separate instance **per VLAN** — you should see distinct spanning tree information for VLAN 10 and VLAN 20 independently.

### Task 4 — Observe RSTP Port Roles and States (New Terminology)
```
show spanning-tree vlan 10
```
Note the terminology differences from 802.1D:
- **Port roles**: Root, Designated, and now also **Alternate** (a backup path to the root, replacing 802.1D's plain "blocking" concept with more specific meaning) and **Backup** (a backup designated port on a shared segment — less commonly seen on typical point-to-point switch links).
- **Port states**: RSTP collapses 802.1D's four states (blocking, listening, learning, forwarding) down to three meaningful ones — **Discarding** (combining blocking and listening), **Learning**, and **Forwarding**.

Identify the root bridge, root port, designated ports, and the alternate port for VLAN 10 specifically.

### Task 5 — Verify Edge Port Auto-Detection
Unlike 802.1D, where PortFast had to be manually configured to get fast access-port behavior, RSTP includes a more automatic concept of an **edge port**:
```
show spanning-tree vlan 10 interface FastEthernet0/1
```
If PortFast was already configured on this port (carried over from the STP lab), confirm it's now reported using RSTP's terminology as an edge port. Note in your lab notes that RSTP's edge port detection can also occur somewhat automatically in some configurations (a port that has never received a BPDU may be treated similarly), but explicitly configuring PortFast remains the reliable, recommended practice rather than depending on automatic detection alone.

### Task 6 — Verify Link Type
RSTP's fast convergence on point-to-point links relies on knowing a link is genuinely point-to-point (a single switch-to-switch connection, not a shared hub-based segment):
```
show spanning-tree vlan 10 interface GigabitEthernet0/1 detail
```
Confirm the inter-switch links are correctly identified as **point-to-point** (which they are, since they're direct switch-to-switch connections) — this is what enables the rapid proposal/agreement handshake described in the next task. If a link were incorrectly treated as shared (e.g., connected through an old-style hub), RSTP would fall back to slower, timer-based convergence on that specific link even while running in rapid-pvst mode overall.

### Task 7 — Understand Why RSTP Converges So Much Faster (Not Just Shorter Timers)
Unlike 802.1D's fixed 15-second listening and 15-second learning timers (adding up to the 30-50 second convergence observed in the STP lab), RSTP uses an active **proposal/agreement handshake** on point-to-point links: a switch proposing to become designated on a link explicitly asks the other side, which — if it agrees the proposing switch has better information — immediately synchronizes and agrees, without waiting out fixed timers at all. This is a fundamentally different mechanism, not simply "the same process with smaller numbers."

### Task 8 — Test Convergence Speed Directly
Identify the currently-alternate port for VLAN 10, then fail the corresponding forwarding link, exactly as in the STP lab's Task 10:
```
interface GigabitEthernet0/X
 shutdown
```
```
show spanning-tree vlan 10
```
Time how quickly the alternate port transitions to forwarding. Confirm this happens in a small number of seconds (often near-instantaneous to a few seconds under good conditions), a dramatic difference from the 802.1D lab's 30-50 second observation. Restore the link:
```
interface GigabitEthernet0/X
 no shutdown
```

---

## Part 2 — Per-VLAN Load Balancing (PVST+'s Distinct Advantage)

### Task 9 — Check the Current Root for Both VLANs
```
show spanning-tree vlan 10
show spanning-tree vlan 20
```
Note which switch is currently root for each VLAN — with default priorities on all switches, both VLANs will have elected the **same** root bridge (whichever switch has the lowest MAC address), meaning the same physical link ends up carrying the traffic load for both VLANs' active forwarding path, while the alternate path sits completely idle for both.

### Task 10 — Configure Different Root Bridges Per VLAN
```
! On SW1 — make it root for VLAN 10
spanning-tree vlan 10 root primary

! On SW2 — make it root for VLAN 20
spanning-tree vlan 20 root primary
```

### Task 11 — Verify Per-VLAN Root Election
```
show spanning-tree vlan 10
show spanning-tree vlan 20
```
Confirm SW1 is now root for VLAN 10, while SW2 is root for VLAN 20 — genuinely different root bridges for different VLANs, something a single-instance protocol (plain RSTP, MST with only one instance, or 802.1D) cannot achieve at all.

### Task 12 — Verify the Load-Balancing Effect
```
show spanning-tree vlan 10
show spanning-tree vlan 20
```
On SW3 specifically, compare which port is the alternate (blocked) port for VLAN 10 versus VLAN 20 — confirm they are now **different physical ports**, meaning VLAN 10's traffic and VLAN 20's traffic now take different paths through the triangle under normal conditions, actively using both redundant links simultaneously rather than leaving one permanently idle for all traffic.

### Task 13 — Final Verification
```
show spanning-tree summary
show spanning-tree vlan 10
show spanning-tree vlan 20
show spanning-tree interface GigabitEthernet0/1 detail
show run | section spanning-tree
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — mode confirmed | All three switches report `rapid-pvst`, with per-VLAN instances |
| Task 4 — port roles/states | RSTP terminology (alternate, discarding) correctly identified via `show spanning-tree` |
| Task 6 — link type | Inter-switch links correctly identified as point-to-point |
| Task 8 — convergence speed | Alternate port transitions to forwarding in a small number of seconds, dramatically faster than the 802.1D STP lab's observation |
| Task 9 — before per-VLAN root config | Both VLANs share the same root bridge and the same idle alternate path |
| Task 11 — per-VLAN root | SW1 root for VLAN 10; SW2 root for VLAN 20 — confirmed genuinely different |
| Task 12 — load balancing | Different physical ports are alternate/blocked for VLAN 10 versus VLAN 20 on SW3 |

---

## Challenge (Optional)
- Deliberately connect a link through an old-style hub (or simulate a shared-medium segment if your platform supports it) and observe/document how RSTP's link-type detection affects convergence behavior on that specific segment compared to the point-to-point links used throughout the rest of this lab.
- Research **Multiple Spanning Tree (MST)** and explain how it addresses PVST+'s main scalability limitation — PVST+ runs a completely separate spanning tree instance per VLAN, which becomes resource-intensive with many VLANs, while MST maps multiple VLANs onto a smaller number of instances, retaining much of PVST+'s load-balancing flexibility with less overhead at scale.
- Extend this lab's per-VLAN load balancing to a third VLAN, and design (in writing, or implemented if your topology allows) a root-bridge assignment plan across three VLANs and three switches that maximizes the use of all redundant links simultaneously, rather than always favoring the same one or two switches as root.