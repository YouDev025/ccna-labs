# Lab: Switch Security Best Practices

## Objective
This is the **capstone lab for the switching series**: combine access port hardening (from the Access Port Configuration lab), port security, and STP protections into one consolidated baseline, and finally implement two controls earlier labs specifically flagged but didn't build — **DHCP Snooping** and **Dynamic ARP Inspection (DAI)** — plus a new STP-specific protection, **Root Guard**, which prevents a rogue or misconfigured switch from ever becoming root even if it advertises a numerically better bridge ID.

---

## Topology

```
              +-----------+
              | DHCP1     |   Legitimate DHCP server
              |192.168.10.2/24|
              +-----------+
                    |
              +-----------+
              | SW-DIST   |   Distribution switch — should ALWAYS be root
              +-----------+
                    | Gi0/1 (trunk, Root Guard enabled here)
                    |
              +-----------+
              | SW-ACCESS |   Access switch — end-user ports
              +-----------+
          Fa0/1   Fa0/2   Fa0/3
            |       |       |
         +-----+ +-----+ +---------+
         | PC1 | | PC2 | | ROGUE   |
         +-----+ +-----+ | DEVICE  |
                          |(rogue   |
                          | DHCP +  |
                          | ARP     |
                          | spoof)  |
                          +---------+
```

- **SW-DIST** is the distribution/core switch and should always remain the STP root — **Root Guard** on its downstream port ensures this even if SW-ACCESS (or something worse, plugged into it) tries to claim root.
- **SW-ACCESS** hosts end-user devices, hardened per the Access Port Configuration lab's baseline.
- **ROGUE DEVICE** simulates two distinct attacks on the same physical connection: acting as an unauthorized DHCP server, and attempting ARP spoofing against PC1/PC2.

---

## VLAN and Addressing Plan

| VLAN | Subnet            |
|------|---------------------|
| 10   | 192.168.10.0/24      |

---

## Part 1 — Access Port and Port Security Baseline (Recap and Consolidate)

### Task 1 — Build the Topology
Place SW-DIST, SW-ACCESS, DHCP1, PC1, PC2, and ROGUE DEVICE as shown, with VLAN 10 throughout and a trunk between SW-DIST and SW-ACCESS.

### Task 2 — Apply the Hardened Access Port Baseline (from the Access Port Configuration Lab)
```
! On SW-ACCESS, Fa0/1 and Fa0/2 (PC1, PC2)
interface range FastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### Task 3 — Configure Fa0/3 (Where ROGUE DEVICE Will Connect) Identically
```
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
```
> Note this port gets the **same** hardened baseline as any other access port — the point of this lab's remaining tasks is that port security and access-port hardening alone do **not** stop every kind of attack, specifically the two demonstrated in Part 3, which is exactly why DHCP Snooping and DAI exist as additional, complementary layers.

---

## Part 2 — Root Guard

### Task 4 — Confirm SW-DIST Is Currently Root
```
show spanning-tree vlan 10
```
Confirm SW-DIST (or configure it to be, via `spanning-tree vlan 10 root primary` if it isn't already) shows as root.

### Task 5 — Configure Root Guard on SW-DIST's Downstream Port
```
! On SW-DIST, the port facing SW-ACCESS
interface GigabitEthernet0/1
 spanning-tree guard root
```
> Root Guard doesn't just prevent SW-ACCESS specifically from becoming root — it prevents **any** switch reachable through this port (including SW-ACCESS itself, or anything plugged into SW-ACCESS further downstream) from ever being accepted as a superior root bridge, regardless of what bridge ID it advertises.

### Task 6 — Simulate an Attempt to Become Root
On SW-ACCESS, temporarily configure an artificially low priority, simulating either a misconfigured switch or an attacker attempting to manipulate the topology:
```
! On SW-ACCESS
spanning-tree vlan 10 priority 0
```

### Task 7 — Verify Root Guard Blocks the Attempt
```
! On SW-DIST
show spanning-tree vlan 10
show spanning-tree inconsistentports
```
Confirm SW-DIST's downstream port is now in a **root-inconsistent** state (a specific blocking condition Root Guard triggers) rather than actually ceding the root role to SW-ACCESS — SW-DIST remains root despite SW-ACCESS advertising a numerically superior priority. Restore SW-ACCESS's priority:
```
spanning-tree vlan 10 priority 32768
```
Confirm the port automatically recovers once the superior BPDUs stop being received.

---

## Part 3 — DHCP Snooping

### Task 8 — Understand the Threat
Without any protection, **ROGUE DEVICE** could run its own DHCP server, racing legitimate DHCP1 to answer client requests — whichever server responds first typically wins, meaning an attacker's rogue server can silently hand out its own gateway/DNS addresses to unsuspecting clients, enabling traffic interception.

### Task 9 — Enable DHCP Snooping
```
! On SW-ACCESS
ip dhcp snooping
ip dhcp snooping vlan 10
```

### Task 10 — Designate Trusted vs. Untrusted Ports
```
! The uplink toward SW-DIST (and ultimately DHCP1) is trusted
interface GigabitEthernet0/1
 ip dhcp snooping trust

! All end-user-facing ports remain untrusted by default — no additional command needed,
! but confirm this explicitly:
show ip dhcp snooping
```
> **Trusted** ports are allowed to relay DHCP server responses (offers, acks); **untrusted** ports (the default for all access ports) are only allowed to send DHCP client requests — any DHCP **server** response arriving on an untrusted port is dropped immediately, which is exactly what stops a rogue server connected to an access port.

### Task 11 — Simulate the Rogue DHCP Server Attack
Configure ROGUE DEVICE (connected to Fa0/3, an untrusted port) to act as a DHCP server, and have PC1 request an address.

### Task 12 — Verify the Attack Is Blocked
```
show ip dhcp snooping
show ip dhcp snooping binding
```
Confirm PC1 successfully receives an address from the **legitimate** DHCP1 server (visible in the snooping binding table, tracking the legitimate lease), while ROGUE DEVICE's DHCP responses never reach PC1 at all — dropped at Fa0/3 since that port is untrusted.

---

## Part 4 — Dynamic ARP Inspection (DAI)

### Task 13 — Understand the Threat
Even without a rogue DHCP server, ROGUE DEVICE could send unsolicited, forged ARP replies claiming to be the default gateway (or another host) — a classic ARP spoofing/poisoning attack that redirects traffic without needing any DHCP involvement at all.

### Task 14 — Enable DAI, Using the DHCP Snooping Binding Table as Its Trust Source
```
! On SW-ACCESS
ip arp inspection vlan 10
```
> DAI validates incoming ARP packets on untrusted ports against the **DHCP snooping binding table** built in Part 3 — an ARP reply claiming an IP/MAC pairing that doesn't match a legitimate DHCP-assigned binding is dropped. This is why DHCP Snooping is typically configured **before** DAI: DAI depends on the binding table DHCP Snooping builds.

### Task 15 — Designate Trusted Ports for DAI
```
interface GigabitEthernet0/1
 ip arp inspection trust
```

### Task 16 — Simulate an ARP Spoofing Attempt
Configure ROGUE DEVICE to send a forged ARP reply claiming to be the IP address of SW-DIST's gateway interface (or PC2's address), from a MAC address that doesn't match any legitimate DHCP snooping binding.

### Task 17 — Verify DAI Blocks the Attempt
```
show ip arp inspection interfaces
show ip arp inspection statistics vlan 10
```
Confirm the forged ARP packet is logged as **denied**, and PC1/PC2's ARP tables remain correctly pointing to legitimate addresses rather than being poisoned toward ROGUE DEVICE.

### Task 18 — Final Verification
```
show spanning-tree vlan 10
show spanning-tree inconsistentports
show ip dhcp snooping
show ip dhcp snooping binding
show ip arp inspection interfaces
show ip arp inspection statistics vlan 10
show port-security
show run | section switchport
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 7 — Root Guard | SW-DIST remains root despite SW-ACCESS advertising a superior priority; downstream port shows root-inconsistent, then recovers automatically |
| Task 12 — DHCP Snooping | PC1 receives its address from legitimate DHCP1 only; rogue DHCP responses dropped at the untrusted port |
| Task 17 — DAI | Forged ARP reply from ROGUE DEVICE logged as denied; legitimate ARP mappings preserved |
| `show port-security` | Confirms hardened baseline still active and unaffected by the new controls layered on top |

---

## Verification Checklist (Consolidated Baseline)

| Layer | Control | Verified By |
|---|---|---|
| Physical/L2 identity | Port security, sticky MAC, nonegotiate | `show port-security`, `show interfaces switchport` |
| STP topology integrity | Root Guard, BPDU Guard | `show spanning-tree inconsistentports` |
| DHCP integrity | DHCP Snooping, trusted/untrusted ports | `show ip dhcp snooping binding` |
| ARP integrity | Dynamic ARP Inspection | `show ip arp inspection statistics` |

---

## Challenge (Optional)
- Add **IP Source Guard** (`ip verify source`) on the access ports, which uses the same DHCP snooping binding table to also prevent IP address spoofing (a device using an IP it was never actually assigned) — completing a three-layer binding-table-driven protection (DHCP Snooping, DAI, IP Source Guard) all built on the same underlying trust data.
- Research **Loop Guard** as a complementary STP protection to Root Guard, explaining the different failure scenario it addresses — Root Guard prevents an unauthorized *root* claim, while Loop Guard prevents a port from incorrectly transitioning to forwarding when it stops receiving BPDUs at all (e.g., due to a unidirectional link failure), which Root Guard does not address.
- Write a final consolidated "switch security baseline" checklist (referencing this lab plus the Access Port Configuration, Port Security, and STP labs) suitable for applying to every access-layer switch in a hypothetical organization — and identify which single control in your checklist you believe provides the most protection relative to its configuration effort, with justification.