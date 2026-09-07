# Lab: Router on a Stick

## Objective
This lab directly extends the **VLAN Basics** lab, where devices in different VLANs correctly could **not** reach each other, since no router was present. Here, you'll add a router connected via a **single trunk link** to the switch, using **subinterfaces** (one per VLAN, each with 802.1Q encapsulation) to provide inter-VLAN routing — "router on a stick," since one physical interface serves every VLAN. You'll also configure **DHCP relay** so VLAN clients can obtain addresses from a centralized DHCP service reachable only through the router.

---

## Topology

```
                       +-----------+
                       |    R1     |   Router on a stick
                       +-----------+
                        Gi0/0 (trunk,
                        single physical
                        interface, two
                        subinterfaces)
                             |
                       Gi0/1 | (trunk)
                       +-----------+
                       |    SW1    |
                       +-----------+
                    Fa0/1      Fa0/2
                      |            |
                   +-----+     +-----+
                   | PC1 |     | PC2 |
                   |VLAN10|    |VLAN20|
                   +-----+     +-----+
```

- **R1** connects to SW1 via a **single** physical link, configured as a trunk on both ends.
- **PC1** (VLAN 10) and **PC2** (VLAN 20) should now be able to reach each other — routed through R1's subinterfaces — despite being in different Layer 2 broadcast domains.

---

## VLAN and Addressing Plan

| VLAN | Name  | Subnet            | Gateway (R1 subinterface) |
|------|-------|---------------------|------------------------------|
| 10   | USERS | 192.168.10.0/24      | 192.168.10.1                  |
| 20   | SALES | 192.168.20.0/24      | 192.168.20.1                  |

| Device | Port  | VLAN | IP Address       |
|--------|-------|------|--------------------|
| PC1    | Fa0/1 | 10   | 192.168.10.10 (DHCP-assigned in Part 2) |
| PC2    | Fa0/2 | 20   | 192.168.20.10 (DHCP-assigned in Part 2) |

---

## Part 1 — Basic Router on a Stick

### Task 1 — Build the Topology
1. Create VLANs 10 and 20 on SW1 (or reuse the VLAN Basics lab's existing configuration if continuing directly from it).
2. Assign Fa0/1 to VLAN 10 and Fa0/2 to VLAN 20 on SW1.
3. Connect R1's Gi0/0 to SW1's Gi0/1 (or another free port).

### Task 2 — Configure the Trunk on SW1's Side
```
! On SW1
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport nonegotiate
```

### Task 3 — Configure Subinterfaces on R1
```
! On R1
interface GigabitEthernet0/0
 no shutdown
 no ip address

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```
> The **physical** interface itself (Gi0/0) needs `no shutdown` but should **not** have an IP address of its own in this design — all addressing lives on the subinterfaces. `encapsulation dot1Q <vlan>` is what ties each subinterface to a specific VLAN's tagged traffic; forgetting this line is one of the most common router-on-a-stick mistakes, and the subinterface simply won't pass any traffic at all without it.

### Task 4 — Configure PC1 and PC2 with Static Addressing (Temporary, Before Part 2's DHCP)
Set PC1 to `192.168.10.10 /24`, gateway `192.168.10.1`, and PC2 to `192.168.20.10 /24`, gateway `192.168.20.1`.

### Task 5 — Verify Subinterfaces Are Up
```
! On R1
show ip interface brief
```
Confirm both subinterfaces show `up/up` — if either shows `up/down` (line protocol down) or doesn't appear correctly, double-check the `encapsulation dot1Q` line and confirm the parent physical interface itself is not shut down.

### Task 6 — Verify Inter-VLAN Routing
```
! From PC1
ping 192.168.20.10 (PC2)
```
This should now **succeed** — directly contrasting the VLAN Basics lab's identical test, which correctly failed with no router present. The only thing that's changed is R1's addition; SW1's VLAN configuration is unchanged from before.

### Task 7 — Verify the Routing Table
```
! On R1
show ip route
```
Confirm both 192.168.10.0/24 and 192.168.20.0/24 appear as directly connected (`C`) networks, each tied to its respective subinterface — R1 doesn't need any static routes or routing protocol here, since both networks are directly attached via the subinterfaces.

### Task 8 — Test a Common Misconfiguration: Missing Encapsulation
Temporarily remove the encapsulation command to observe the failure directly, rather than only being told about it:
```
! On R1
interface GigabitEthernet0/0.10
 no encapsulation dot1Q 10
```
```
show ip interface brief
```
Confirm Gi0/0.10 now shows as down/administratively affected, or otherwise clearly non-functional — restore it immediately afterward:
```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
```
Confirm connectivity is restored.

### Task 9 — Test a Native VLAN Mismatch Between R1 and SW1
By default, dot1Q subinterfaces without an explicit native VLAN handle the native VLAN in a way that must match the switch trunk's native VLAN setting. Introduce a mismatch:
```
! On SW1
interface GigabitEthernet0/1
 switchport trunk native vlan 10
```
```
! On R1 — check for a native VLAN mismatch warning
show logging | include NATIVE_VLAN
```
Confirm this produces the same kind of native VLAN mismatch warning covered in the Trunking lab, now observed in a router-on-a-stick context specifically rather than switch-to-switch — restore SW1's native VLAN to its default (or configure R1's subinterface to explicitly match) before continuing:
```
! On SW1
interface GigabitEthernet0/1
 switchport trunk native vlan 1
```

---

## Part 2 — DHCP Relay for VLAN Clients

### Task 10 — Configure R1 as a DHCP Server (or Point to an External One)
For this lab, configure R1 itself as the DHCP server for both VLANs, to keep the topology simple:
```
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.20.1
ip dhcp pool VLAN10-POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
ip dhcp pool VLAN20-POOL
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
```
> Since R1 is directly the gateway for both subnets via its subinterfaces, `ip helper-address` isn't strictly necessary in this specific design (a genuinely external/centralized DHCP server on a **different** subnet reachable through R1 would need it instead) — this task is included so you understand DHCP works "for free" here specifically because R1 is both the gateway and the DHCP server simultaneously.

### Task 11 — Reconfigure PC1 and PC2 for DHCP
Switch both PCs from static addressing (Task 4) to automatic (DHCP), and release/renew their addressing.

### Task 12 — Verify DHCP Assignment
```
! On R1
show ip dhcp binding
```
Confirm both PCs received addresses within their respective VLAN's pool, with the correct default gateway.
```
! From PC1
ping 192.168.20.10
```
Should still succeed, now with DHCP-assigned addressing instead of static.

### Task 13 — Simulate a Genuinely External DHCP Server (Understand ip helper-address)
As a conceptual extension, imagine DHCP1 were a separate physical server on a **third** subnet, not directly reachable by broadcast from VLAN 10/20 clients (DHCP relies on broadcast, which routers do not forward by default). In this scenario, each subinterface would need:
```
interface GigabitEthernet0/0.10
 ip helper-address <DHCP1's address>
interface GigabitEthernet0/0.20
 ip helper-address <DHCP1's address>
```
> Document in your lab notes why this command is unnecessary in this lab's actual configuration (R1 is the DHCP server itself) but would become essential the moment DHCP service moved to a separate device — `ip helper-address` converts broadcast DHCP requests into unicast, forwarding them to a specific server across a subnet boundary the broadcast itself could never cross.

### Task 14 — Final Verification
```
show ip interface brief
show ip route
show vlan brief
show interfaces trunk
show ip dhcp binding
show run | section interface GigabitEthernet0/0
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — subinterfaces | Both show `up/up` |
| Task 6 — inter-VLAN routing | PC1 ↔ PC2 succeeds, contrasting the VLAN Basics lab's identical (correctly failing) test |
| Task 7 — routing table | Both subnets shown as directly connected via their respective subinterfaces |
| Task 8 — missing encapsulation | Subinterface fails to function correctly; restored once the command is reapplied |
| Task 9 — native VLAN mismatch | Produces the expected CDP mismatch warning; resolved by matching native VLAN on both sides |
| Task 12 — DHCP | Both PCs receive correct, VLAN-appropriate addresses; inter-VLAN connectivity still works post-DHCP |

---

## Challenge (Optional)
- Add a **third** VLAN and subinterface, and confirm inter-VLAN routing extends correctly to the new VLAN without disrupting the existing two.
- Research why router-on-a-stick is generally considered **less scalable** than a **Layer 3 switch with SVIs** (a topic for a dedicated future lab) for larger networks — specifically, the single-physical-link bandwidth constraint this design has, since all inter-VLAN traffic must traverse the one trunk link to R1 and back, versus a Layer 3 switch routing internally at much higher aggregate throughput.
- Apply an access-list on one of R1's subinterfaces to restrict specific inter-VLAN traffic (e.g., PC1 can reach PC2 on HTTP only, similar to the Extended ACLs lab's policy design) — combining inter-VLAN routing with the traffic-filtering skills from earlier in this series.