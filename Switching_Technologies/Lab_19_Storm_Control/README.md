# Lab: Storm Control

## Objective
Configure and verify **storm control** — a safety net against excessive broadcast, multicast, or unknown-unicast traffic, independent of STP. This lab specifically demonstrates a realistic scenario where **STP cannot help at all**: two ports configured with `bpdufilter` (which suppresses BPDUs entirely, unlike BPDU Guard) are accidentally connected together, creating a genuine physical loop that STP never even detects — leaving storm control as the only thing standing between that loop and a network-wide outage.

---

## Topology

```
                          +-----------+
                          |    SW1    |
                          +-----------+
               Fa0/1        Fa0/3    Fa0/4
                 |             |       |
              +-----+       (accidentally connected together
              | PC1 |        by a patch cable — simulating a
              +-----+        user plugging both ends of a
                              cable into the same patch panel
                              row, a real and common mistake)
```

- **Fa0/1 (PC1)**: a normal, correctly-configured access port, used to confirm the rest of the network stays healthy even while the storm is contained on Fa0/3/Fa0/4.
- **Fa0/3 and Fa0/4**: both configured as end-user-style access ports with `bpdufilter` — meaning **neither port will ever generate or process BPDUs at all**. If accidentally connected to each other (directly, or via an unmanaged hub/switch), this creates a real Layer 2 loop that STP has no visibility into whatsoever.

---

## Tasks

### Task 1 — Build the Topology
1. Connect PC1 to Fa0/1.
2. Leave Fa0/3 and Fa0/4 physically disconnected from each other for now — you'll connect them together deliberately in Task 4, once storm control is already in place.

### Task 2 — Configure Fa0/3 and Fa0/4 as BPDU-Filtered Edge Ports
```
interface range FastEthernet0/3 - 4
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpdufilter enable
```
> **This is deliberately different from BPDU Guard** (used in earlier labs): `bpdufilter` doesn't just disable the port if a BPDU is *received* — it stops the port from sending or processing BPDUs **at all**, meaning if a loop is accidentally created here, STP will never generate the blocking behavior that would normally prevent a broadcast storm. This setting exists for specific edge-case scenarios (e.g., certain non-Cisco end devices that get confused by receiving BPDUs at all) but carries real risk if applied to a port that could ever end up looped — which is exactly the scenario this lab sets up intentionally, to show why a second layer of protection matters.

### Task 3 — Configure Storm Control Before Creating the Loop
Configure storm control **first**, so it's active protection in place before the accidental loop even happens — exactly as it should be in a real deployment (a safety net installed proactively, not reactively after an incident):
```
interface range FastEthernet0/3 - 4
 storm-control broadcast level 5.00
 storm-control multicast level 5.00
 storm-control unicast level 10.00
 storm-control action shutdown
```
> Broadcast and multicast are set more conservatively (5%) than unicast (10%) in this example, since a Layer 2 loop specifically causes broadcast/multicast traffic to multiply catastrophically (each switch re-floods it endlessly), while a unicast storm is a somewhat different (though still real) concern. `storm-control action shutdown` means the port will be **disabled entirely** once the threshold is exceeded, rather than merely suppressing the excess traffic — appropriate for this scenario, since a genuine loop needs to be stopped decisively, not just throttled.

### Task 4 — Verify Storm Control Configuration
```
show storm-control broadcast
show storm-control multicast
show storm-control unicast
```
Confirm the configured thresholds and action are correctly applied to both Fa0/3 and Fa0/4 before proceeding.

### Task 5 — Create the Accidental Loop
Physically connect Fa0/3 and Fa0/4 together with a patch cable — simulating the real-world mistake this lab is built around.

### Task 6 — Observe What Happens (and What Doesn't)
```
show spanning-tree vlan 10
```
Confirm STP shows **no** indication of a problem on Fa0/3/Fa0/4 at all — since `bpdufilter` means these ports never exchange BPDUs, STP genuinely has no way to detect this loop exists. This is the central point of the lab: without storm control, this configuration would allow a broadcast storm to build unchecked and potentially impact the entire switch (and network beyond it), with STP offering zero protection.

### Task 7 — Verify Storm Control Triggers
```
show storm-control broadcast
show interfaces FastEthernet0/3 status
show interfaces FastEthernet0/4 status
```
Confirm that once broadcast (or multicast/unicast) traffic on the looped segment exceeds the configured threshold, both ports are placed into an **err-disabled** state due to the storm-control shutdown action — this is what actually stops the loop, entirely independent of STP.
```
show logging | include storm
```
Confirm a log message identifies the specific interface and traffic type that triggered the shutdown.

### Task 8 — Confirm PC1 Was Unaffected
```
! From PC1
ping <switch management address or another known-good device>
```
Confirm PC1's connectivity remained healthy throughout — storm control's containment was limited to the specific ports involved in the loop, not the entire switch or VLAN.

### Task 9 — Physically Remove the Loop
Disconnect the patch cable between Fa0/3 and Fa0/4.

### Task 10 — Configure Errdisable Recovery
```
errdisable recovery cause storm-control
errdisable recovery interval 300
```
Without this, the ports would remain err-disabled indefinitely, requiring a manual `shutdown`/`no shutdown` — appropriate for a scenario where a human should investigate before automatically re-enabling, but for this lab, configure automatic recovery so you can observe the full cycle.

### Task 11 — Verify Recovery
```
show interfaces FastEthernet0/3 status
show interfaces FastEthernet0/4 status
```
After the configured interval, confirm both ports return to a normal, up state — now that the physical loop has actually been removed (Task 9), they should stay healthy rather than immediately re-triggering storm control again.

### Task 12 — Final Verification
```
show storm-control broadcast
show storm-control multicast
show storm-control unicast
show errdisable recovery
show run | section storm-control
show run | section bpdufilter
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 6 — STP visibility into the loop | None — `show spanning-tree` shows no problem, confirming BPDU filtering genuinely blinds STP to this specific loop |
| Task 7 — storm control triggers | Fa0/3 and Fa0/4 both enter err-disabled state once the broadcast/multicast/unicast threshold is exceeded |
| Task 8 — PC1 unaffected | Remains healthy and reachable throughout, confirming the containment was limited to the looped ports |
| Task 11 — recovery | Both ports return to normal operation after the loop is physically removed and the errdisable recovery interval elapses |

---

## Challenge (Optional)
- Repeat this lab using `storm-control action trap` instead of `shutdown`, and compare the resulting behavior — the loop and resulting traffic multiplication would **not** be stopped by this action alone (it only sends an SNMP notification), meaning you'd need to combine it with manual intervention or a separate automated response; discuss in your lab notes why `shutdown` is generally the safer default action for broadcast/multicast thresholds specifically.
- Reconfigure Fa0/3 and Fa0/4 with `spanning-tree bpduguard enable` instead of `bpdufilter`, recreate the physical loop, and confirm STP **does** now detect and react to it (since BPDU Guard, unlike BPDU Filter, still processes and reacts to received BPDUs) — write a clear comparison in your lab notes of when each setting is appropriate, given how different their behavior turned out to be in this lab.
- Research why storm control thresholds are usually expressed as a **percentage of link bandwidth** rather than an absolute packet-per-second or bit-per-second value in older configurations, and calculate the actual approximate broadcast traffic rate (in Mbps) that a 5% threshold represents on a Gigabit Ethernet port.