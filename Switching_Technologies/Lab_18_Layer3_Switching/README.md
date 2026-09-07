# Lab: Layer 3 Switching

## Objective
This lab is the scalability answer flagged in the Router on a Stick lab's challenge section: instead of routing all inter-VLAN traffic through a single trunk link to an external router, a **Layer 3 switch** routes internally using **SVIs (Switched Virtual Interfaces)** — one logical routed interface per VLAN, all within the same physical device, with no single-link bottleneck. This lab also covers **routed ports** (a physical switch port with no VLAN/switchport behavior at all, acting exactly like a router interface) for connecting out to an edge router.

---

## Topology

```
                                      +-----------+
                                      |  R-EDGE   |   Simulated ISP/upstream router
                                      +-----------+
                                       203.0.113.2/30
                                            |
                                      Gi0/3 | 203.0.113.1/30 (ROUTED PORT — no VLAN)
                                      +---------------+
                                      |     L3SW      |   Multilayer switch
                                      +---------------+
                                Gi0/1 (SVI Vlan10)  Gi0/2 (SVI Vlan20)
                                192.168.10.1/24      192.168.20.1/24
                                      |                    |
                                   +-----+              +-----+
                                   | PC1 |              | PC2 |
                                   |VLAN10|             |VLAN20|
                                   +-----+              +-----+
```

- **L3SW** is a multilayer switch: VLANs 10 and 20 exist on it exactly as on any switch, but it also has **SVIs** acting as each VLAN's default gateway, routing between them **internally** — no external router or single trunk link is involved in inter-VLAN traffic at all.
- **Gi0/3** is configured as a **routed port** (not a VLAN access/trunk port) — a physical interface behaving exactly like a router's, used here for the uplink to R-EDGE.

---

## VLAN and Addressing Plan

| VLAN | SVI Address | Subnet |
|------|-------------|--------|
| 10   | 192.168.10.1/24 | 192.168.10.0/24 |
| 20   | 192.168.20.1/24 | 192.168.20.0/24 |

| Interface | Type | Address |
|---|---|---|
| L3SW Gi0/3 | Routed port | 203.0.113.1/30 |
| R-EDGE (facing L3SW) | Router interface | 203.0.113.2/30 |

---

## Tasks

### Task 1 — Build the Topology
1. Connect PC1 to L3SW (an access port, VLAN 10), PC2 to L3SW (an access port, VLAN 20), and L3SW's Gi0/3 to R-EDGE.
2. Create VLANs 10 and 20 on L3SW, and assign the PC-facing ports accordingly.

### Task 2 — Enable IP Routing on the Switch
This is the single most commonly forgotten step in this entire lab — without it, the switch behaves as a pure Layer 2 device regardless of any SVI configuration:
```
ip routing
```
> Many multilayer switches ship with this disabled by default. If you configure everything else correctly in this lab but inter-VLAN traffic still doesn't route, this is the first thing to check.

### Task 3 — Configure the SVIs
```
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface Vlan20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```

### Task 4 — Verify SVI Status (and Understand the Common Gotcha)
```
show ip interface brief
```
Confirm both SVIs show `up/up`. If an SVI shows `up/down` (line protocol down), the most common cause is that **VLAN has no active ports** — an SVI's line-protocol state is tied to whether at least one port assigned to that VLAN is actually up; an SVI for a VLAN with zero active ports, or a VLAN that doesn't even exist in the VLAN database, will not come up no matter how correctly the SVI itself is configured. Confirm PC1 and PC2's ports are correctly assigned and up before troubleshooting the SVI configuration itself.

### Task 5 — Configure PC1 and PC2
Set static or DHCP addressing within each VLAN's subnet, with the corresponding SVI as the default gateway.

### Task 6 — Verify Inter-VLAN Routing via SVIs
```
! From PC1
ping 192.168.20.10 (PC2)
```
Should succeed — routed entirely within L3SW via its SVIs, with no external router or trunk link involved at all, unlike the Router on a Stick lab's design.

### Task 7 — Configure the Routed Port for the Uplink
```
interface GigabitEthernet0/3
 no switchport
 ip address 203.0.113.1 255.255.255.252
 no shutdown
```
> `no switchport` is what converts this from a normal Layer 2 switch port into a genuine Layer 3 routed interface — it now behaves exactly like a router's physical interface: no VLAN membership, no trunking, just a directly-addressed point-to-point link.

### Task 8 — Configure R-EDGE and Basic Routing
```
! On R-EDGE
interface <facing L3SW>
 ip address 203.0.113.2 255.255.255.252
 no shutdown
ip route 192.168.10.0 255.255.255.0 203.0.113.1
ip route 192.168.20.0 255.255.255.0 203.0.113.1
```
```
! On L3SW — a default route toward R-EDGE
ip route 0.0.0.0 0.0.0.0 203.0.113.2
```

### Task 9 — Verify the Routed Port
```
! On L3SW
show ip interface brief
```
Confirm Gi0/3 shows an IP address directly (not as an SVI, and not showing any VLAN association) — this is the visible difference between a routed port and a normal switchport.
```
! From PC1
ping 203.0.113.2
```
Should succeed, confirming both SVI-based inter-VLAN routing and routed-port external connectivity are working together correctly.

### Task 10 — Compare Bandwidth Characteristics Against Router on a Stick
In your lab notes, contrast this design against the Router on a Stick lab directly: there, **all** inter-VLAN traffic (PC1↔PC2) had to physically traverse the single trunk link to R1 and back — meaning that link's bandwidth was a hard ceiling on total inter-VLAN throughput, even though PC1 and PC2 might be sitting on the same physical switch. Here, PC1↔PC2 traffic never leaves L3SW at all — it's routed at full internal switching fabric speed, with the Gi0/3 uplink reserved purely for genuinely external traffic. This is the core practical reason Layer 3 switching is preferred at scale.

### Task 11 — Final Verification
```
show ip interface brief
show ip route
show vlan brief
show run | section interface
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 — `ip routing` | Enabled; without it, no inter-VLAN routing would occur regardless of SVI configuration |
| Task 4 — SVI status | Both SVIs `up/up`, contingent on their VLANs having active ports |
| Task 6 — inter-VLAN routing | PC1 ↔ PC2 succeeds entirely within L3SW |
| Task 7/9 — routed port | Gi0/3 shows a direct IP address, no VLAN/switchport association; external ping succeeds |
| Task 10 — bandwidth comparison | Correctly articulates why SVI-based routing avoids the single-trunk-link bottleneck inherent to router-on-a-stick |

---

## Challenge (Optional)
- Shut down PC1's access port (removing the only active port in VLAN 10) and observe the SVI for VLAN 10 transition to `up/down` — directly reproducing the Task 4 gotcha rather than just reading about it, then restore the port and confirm recovery.
- Add a routing protocol (e.g., OSPF, from earlier in this series) between L3SW and R-EDGE instead of static routes, and confirm inter-VLAN and external connectivity both continue to work correctly with dynamic routing in place.
- Research **CEF (Cisco Express Forwarding)** and its role in making Layer 3 switching fast enough for high-throughput inter-VLAN routing at line rate — contrast this briefly against the process-switching concepts that older, purely software-based routing relied on, and explain why this matters specifically for a device expected to route traffic between many VLANs simultaneously at high speed.