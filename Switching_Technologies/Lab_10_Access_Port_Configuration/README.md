# Lab: Access Port Configuration

## Objective
Go beyond simply assigning a VLAN to a port (covered in the VLAN Basics lab) to understand **why access ports need to be explicitly and completely configured**, not left at platform defaults. This lab demonstrates the **switch spoofing VLAN hopping attack** — where an attacker's device, plugged into a carelessly-configured "access" port, negotiates a trunk using DTP and gains visibility into VLANs it was never supposed to reach — and shows the specific configuration that prevents it entirely.

---

## Topology

```
                          +-----------+
                          |    SW1    |
                          +-----------+
              Fa0/1        Fa0/2         Fa0/3
                |             |             |
            +-------+     +-------+     +---------+
            |  PC1  |     |  PC2  |     |ATTACKER |
            |VLAN 10|     |VLAN 20|     |(plugs in|
            |(hard- |     |(default,|   | to Fa0/2)|
            | ened) |     | left as |    +---------+
            +-------+     |'lazy'   |
                          | config) |
                          +-------+
```

- **Fa0/1 (PC1)**: configured **correctly and completely** — the hardened baseline this lab is building toward.
- **Fa0/2 (PC2)**: configured the way many real ports end up in practice — VLAN assigned, but left at **default trunking negotiation settings**, representing a common oversight.
- **ATTACKER** will be connected to Fa0/2 (in place of PC2) specifically to demonstrate what that oversight allows.

---

## VLAN Plan

| VLAN | Name    | Subnet            |
|------|---------|---------------------|
| 10   | USERS   | 192.168.10.0/24      |
| 20   | SALES   | 192.168.20.0/24      |
| 999  | UNUSED-NATIVE | (no addressing — parking VLAN) |

---

## Tasks

### Task 1 — Build the Topology
1. Create VLANs 10, 20, and 999 on SW1.
2. Connect PC1 to Fa0/1 (VLAN 10) and PC2 to Fa0/2 (VLAN 20) initially.
3. Apply basic IP addressing to PC1 and PC2 within their respective VLANs.

### Task 2 — Configure Fa0/1 Correctly (The Hardened Baseline)
```
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
```
> Every one of these lines matters: `switchport mode access` **hard-sets** the port (not leaving it to DTP negotiation at all), `switchport access vlan 10` assigns the VLAN, `switchport nonegotiate` stops the port from processing DTP frames entirely, and PortFast/BPDU Guard (from the STP lab) add loop-prevention safety. This is the complete, correct baseline.

### Task 3 — Configure Fa0/2 "Lazily" (VLAN Assigned, Nothing Else)
```
interface FastEthernet0/2
 switchport access vlan 20
```
> Notice what's **missing** compared to Task 2: no explicit `switchport mode access`, no `nonegotiate`. This is deliberately realistic — a busy administrator assigns the VLAN and moves on, without setting the port's trunking behavior explicitly. On many platforms, this leaves the port in a **dynamic** (negotiable) state by default.

### Task 4 — Confirm the Difference in Negotiation State
```
show interfaces FastEthernet0/1 switchport
show interfaces FastEthernet0/2 switchport
```
Confirm Fa0/1 shows `Administrative Mode: static access` with negotiation disabled, while Fa0/2 shows an administrative mode that still permits negotiation (e.g., `dynamic auto`, depending on platform default) — this is the exact gap the attack in the next task exploits.

### Task 5 — Simulate the Switch Spoofing Attack on Fa0/2
Replace PC2 with ATTACKER on Fa0/2. Using a tool capable of generating DTP frames (e.g., Yersinia, or a Linux host with the appropriate capability — treat this conceptually if your lab platform can't generate real DTP frames), have ATTACKER send a **DTP Desirable** frame toward SW1.

### Task 6 — Verify the Attack Succeeds
```
show interfaces FastEthernet0/2 switchport
```
Because Fa0/2 was left in a negotiable state, confirm it has now become a **trunk port** — ATTACKER has successfully negotiated trunking on a port that was only ever supposed to carry a single access VLAN for one PC. Depending on the trunk's resulting allowed-VLAN list (defaulting to all VLANs unless otherwise restricted elsewhere), ATTACKER may now have Layer 2 visibility into VLAN 10 and other VLANs entirely, despite never having been physically connected to a port assigned to those VLANs.

### Task 7 — Understand the Real Impact
In your lab notes, explain concretely what this access gives an attacker beyond "unauthorized network access": specifically, the ability to inject or capture 802.1Q-tagged traffic across multiple VLANs from a single physical connection point, potentially including VLANs whose traffic the attacker was never meant to see at all — an escalation from "connected to one VLAN" to "attached to the trunk itself."

### Task 8 — Remediate Fa0/2
```
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### Task 9 — Verify the Attack No Longer Succeeds
Reconnect ATTACKER to the now-hardened Fa0/2 and repeat the DTP Desirable frame attempt from Task 5.
```
show interfaces FastEthernet0/2 switchport
```
Confirm the port **remains** a static access port in VLAN 20 — `switchport nonegotiate` means DTP frames arriving on this port are simply ignored, regardless of what they request.

### Task 10 — Apply the Same Hardening to Every Remaining Access Port
As a final pass, confirm every access port on SW1 (not just the two used in this specific demonstration) follows the same complete baseline from Task 2/Task 8 — a real hardening pass should never leave some ports "fixed" and others still at the vulnerable default, since an attacker will simply look for whichever port was missed.

### Task 11 — Verify
```
show interfaces status
show interfaces FastEthernet0/1 switchport
show interfaces FastEthernet0/2 switchport
show run | section interface
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 4 — negotiation state comparison | Fa0/1 shows static access, negotiation disabled; Fa0/2 shows a negotiable administrative mode |
| Task 6 — attack against unhardened port | Fa0/2 successfully becomes a trunk after receiving a DTP Desirable frame |
| Task 9 — attack against hardened port | Fa0/2 remains static access; DTP frames are ignored due to `nonegotiate` |
| Task 10 — full pass | Every access port on SW1 follows the identical hardened baseline, none left at defaults |

---

## Challenge (Optional)
- Research the **double-tagging VLAN hopping attack** (a related but distinct technique from switch spoofing, exploiting native VLAN handling rather than DTP) and explain why this lab's `nonegotiate` hardening does **not** protect against it — that specific attack is instead mitigated by the dedicated-unused-native-VLAN practice from the Trunking lab, reinforcing that different attack techniques require different, complementary defenses.
- Write a one-page "access port hardening standard" (a short internal policy document) listing every command from this lab's Task 2/8 baseline, with a one-line justification for each — suitable for a real team to reference when provisioning new switch ports.
- Audit a larger simulated switch (e.g., 24 ports, most unused) and write a script or manual process description for identifying every port that does **not** match the hardened baseline, so a real network team could efficiently find and remediate "lazy" configurations like Fa0/2's before an attacker does.