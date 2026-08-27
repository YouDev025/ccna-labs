# Lab: Advanced Port Security

## Objective
Building on Lab 02: Port Security, this lab covers the techniques needed for a **realistic edge port design**: securing a port that carries **both a voice and a data VLAN** (an IP phone with a PC daisy-chained behind it — a very common real deployment, and one where a simple "maximum 1 MAC" rule breaks immediately), configuring **MAC address aging**, and combining port security with **BPDU Guard** and **storm control** for a genuinely hardened access-layer port.

---

## Topology

```
              +-----------+
              |    SW1    |
              +-----------+
               Fa0/1  |  Fa0/2
                |            |
          +-----------+  +---------+
          | IP-PHONE  |  |  PC2    |
          |    |      |  |(rogue/  |
          |  PC1      |  | extra   |
          | (daisy-   |  | device  |
          |  chained  |  | for     |
          |  behind   |  | testing)|
          |  phone)   |  +---------+
          +-----------+
```

- **Fa0/1** carries both a **voice VLAN** (for IP-PHONE) and a **data VLAN** (for PC1, physically connected through the phone's built-in switch port — the standard IP telephony deployment pattern).
- **PC2** on Fa0/2 is used to simulate an unauthorized additional device for testing violations in this more complex scenario.

---

## VLAN and Addressing Plan

| VLAN | Purpose | Subnet            |
|------|---------|---------------------|
| 10   | Data (PCs)     | 192.168.10.0/24     |
| 20   | Voice (phones) | 192.168.20.0/24     |

| Device    | Port  | VLAN | IP Address       |
|-----------|-------|------|--------------------|
| IP-PHONE  | Fa0/1 (voice) | 20 | 192.168.20.10  |
| PC1       | Fa0/1 (data, via phone) | 10 | 192.168.10.10 |
| PC2       | Fa0/2 | 10 | 192.168.10.11 |

---

## Tasks

### Task 1 — Build the Topology and VLANs
1. Create VLAN 10 (data) and VLAN 20 (voice) on SW1.
2. Connect IP-PHONE to Fa0/1, with PC1 daisy-chained into the phone's own PC port (or simulate this connection pattern per your lab platform's capabilities).
3. Connect PC2 to Fa0/2.
4. Apply the addressing above.

### Task 2 — Configure the Voice/Data Port
```
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
```
> `switchport voice vlan 20` is what allows this single physical port to correctly carry both the phone's voice traffic (tagged, VLAN 20) and the PC's data traffic (untagged, VLAN 10) simultaneously — without this, only one device/VLAN could realistically use the port.

### Task 3 — Attempt Basic Port Security (Observe the Problem)
Apply the same simple configuration style as the basic Port Security lab:
```
interface FastEthernet0/1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
```
Generate traffic from both the phone and PC1:
```
! From PC1
ping 192.168.10.1  (switch management IP)
```
Check:
```
show port-security interface FastEthernet0/1
```
Depending on which device's MAC was learned first, the **second** device (whichever generates traffic second) will trigger a violation — a maximum of 1 clearly isn't correct for a port that legitimately carries two devices. This is the problem this lab's remaining tasks fix properly.

### Task 4 — Correct the Maximum for a Voice+Data Port
```
interface FastEthernet0/1
 switchport port-security maximum 2
```
> Two is correct here: one MAC for the phone (VLAN 20) and one for PC1 (VLAN 10). Some platforms also support `switchport port-security maximum <N> vlan access` and `switchport port-security maximum <N> vlan voice` to set **per-VLAN** limits explicitly on the same port — check your platform's support for this more precise option, which is generally preferable to a single combined maximum when available.

### Task 5 — Verify Both Devices Are Now Learned Correctly
```
! From PC1 and from IP-PHONE (generate traffic from both)
show port-security address
```
Confirm **two** sticky MAC entries now exist on Fa0/1 — one associated with VLAN 10 (PC1) and one with VLAN 20 (IP-PHONE).

### Task 6 — Configure MAC Address Aging
By default, sticky/dynamic port security entries don't expire on their own. Configure aging so stale entries (e.g., if a PC is disconnected and replaced much later) are eventually removed automatically rather than permanently consuming one of the port's MAC slots:
```
interface FastEthernet0/1
 switchport port-security aging time 60
 switchport port-security aging type inactivity
```
> `aging type inactivity` ages out an entry based on how long it's been since that specific MAC was last seen sending traffic (more useful for this scenario) rather than `absolute`, which ages out exactly N minutes after the entry was first learned regardless of ongoing activity — inactivity-based aging is usually the better choice for ports where legitimate devices might occasionally be idle but shouldn't be aged out just because a fixed clock expired.

### Task 7 — Verify Aging Configuration
```
show port-security interface FastEthernet0/1
```
Confirm the aging time and type are reflected in the output.

### Task 8 — Configure Fa0/2 (PC2's Port) with Its Own Security
```
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

### Task 9 — Harden Both Access Ports with BPDU Guard
Access ports connecting to end devices (phones, PCs) should never receive Spanning Tree BPDUs — if one does, it's a strong signal of either a misconfiguration (someone plugged a switch into this port) or a deliberate attack attempting to manipulate the spanning tree topology. BPDU Guard immediately disables the port if this happens:
```
interface FastEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable

interface FastEthernet0/2
 spanning-tree portfast
 spanning-tree bpduguard enable
```
> `spanning-tree portfast` is a prerequisite in most designs (it marks the port as an edge port, skipping the normal STP listening/learning delay), and BPDU Guard is the security control layered on top of that assumption — if the port ever does receive a BPDU despite being marked as an edge port, something is wrong, and the port should shut down rather than participate in STP.

### Task 10 — Add Storm Control as a Complementary Layer
Port security limits **which** devices can use a port; storm control limits **how much** broadcast/multicast/unicast traffic any single connected device can flood, protecting against both malicious floods and simple malfunctioning NICs:
```
interface FastEthernet0/1
 storm-control broadcast level 5.00
interface FastEthernet0/2
 storm-control broadcast level 5.00
```
> This example limits broadcast traffic to 5% of port bandwidth before storm control begins suppressing it — adjust the specific threshold based on your actual traffic baseline in a real deployment rather than treating 5% as a universal default.

### Task 11 — Test the Full Hardened Port (Fa0/2)
Simulate an unauthorized device attempt by connecting a different device's MAC in place of PC2 (or use PC2 itself if it wasn't the originally-learned sticky address):
```
! Trigger a port-security violation
```
```
show interfaces FastEthernet0/2 status
show port-security interface FastEthernet0/2
```
Confirm the port is `err-disabled` due to the security violation, consistent with `violation shutdown` from Task 8.

### Task 12 — Recover and Configure Automatic Errdisable Recovery
```
errdisable recovery cause psecure-violation
errdisable recovery cause bpduguard
errdisable recovery interval 300
```
Reconnect the legitimate device and confirm the port recovers automatically after the configured interval, without requiring a manual `shutdown`/`no shutdown`.

### Task 13 — Final Verification
```
show port-security
show port-security interface FastEthernet0/1
show port-security interface FastEthernet0/2
show port-security address
show spanning-tree interface FastEthernet0/1 detail
show errdisable recovery
show interfaces status
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — maximum 1 on voice+data port | Demonstrates the problem: second device triggers a violation unnecessarily |
| Task 5 — maximum 2, both devices learned | Two sticky entries present, one per VLAN, both legitimate devices working simultaneously |
| Task 7 — aging | Configured aging time/type reflected in `show port-security interface` |
| Task 9 — BPDU Guard | Configured on both access ports; would trigger err-disable if a BPDU were ever received |
| Task 11 — Fa0/2 violation | Port enters `err-disabled` on an unauthorized MAC, consistent with `violation shutdown` |
| Task 12 — automatic recovery | Port re-enables automatically after the configured interval, no manual intervention |
| `show port-security` (switch-wide) | Confirms correct maximums, violation modes, and current counts for both ports |

---

## Challenge (Optional)
- Configure **per-VLAN** maximums on Fa0/1 if your platform supports it (`switchport port-security maximum 1 vlan access` and `switchport port-security maximum 1 vlan voice`), and compare the resulting behavior/flexibility against the single combined maximum-of-2 approach used in this lab.
- Research and briefly compare port security (Layer 2, MAC-based) against **802.1X** (identity-based network access control) — explain a scenario where 802.1X would be clearly preferable (e.g., a shared conference-room port used by many different, unpredictable devices, where a fixed MAC allowlist isn't practical).
- Extend this lab's hardened-port design with **DHCP snooping** and **Dynamic ARP Inspection** (if covered in your course), and explain how these additional controls address threats (rogue DHCP servers, ARP spoofing) that port security and BPDU Guard alone do not cover.