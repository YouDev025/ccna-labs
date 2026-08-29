# Lab: Device Hardening

## Objective
This is a **capstone lab** for the device security series: apply a complete hardening checklist to a single router and switch, combining techniques from the SSH, Port Security, AAA, HTTPS, Banner and Login, Advanced Port Security, and SSH Key Management labs into one cohesive baseline configuration — plus several hardening items not yet covered individually: **disabling unused services**, **shutting down unused physical ports**, and general configuration hygiene (`service password-encryption`, disabling source routing, etc.). By the end, you'll have a single device configuration representing a reasonable real-world security baseline, along with a checklist you could reuse on future devices.

---

## Topology

```
                          +-----------+
                          |    R1     |   Router being hardened
                          +-----------+
                        Gi0/0  |  192.168.10.1/24
                                |
                              SW1        Switch being hardened
                    +----------+----------+----------+
                    |                     |          |
              +-----------+         +-----------+  (unused ports:
              |   PC1     |         |  SRV1     |   Fa0/3, Fa0/4 —
              |(admin)    |         |(protected |    should be shut
              +-----------+         | server)   |    down entirely)
                                     +-----------+
```

- **R1** and **SW1** are the two devices being hardened in this lab.
- **PC1** is the trusted admin workstation used throughout for verification.
- **SRV1** occupies one active port; **Fa0/3** and **Fa0/4** on SW1 are deliberately left unused, representing the common real-world oversight of ports that are patched but have no device connected — a genuine attack surface if left enabled.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |
| SRV1   | NIC       | 192.168.10.20       | 255.255.255.0        |

---

## Part 1 — Foundational Access Hardening (Combining Prior Labs)

### Task 1 — Build the Topology
Place R1, SW1, PC1, and SRV1 as shown, and apply the addressing above.

### Task 2 — Apply SSH Hardening (from the SSH and SSH Key Management labs)
```
hostname R1
ip domain-name lab.local
crypto key generate rsa modulus 2048 label R1-HOSTKEY
username netadmin privilege 15 secret StrongAdminPass123
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0
line console 0
 exec-timeout 10 0
```

### Task 3 — Apply Banner Hardening (from the Banner and Login lab)
```
banner motd #
***************************************************************
  WARNING: Authorized access only. All activity is logged and
  monitored. Unauthorized use is prohibited and may result in
  legal action.
***************************************************************
#
```

### Task 4 — Apply Brute-Force Protection (from the Banner and Login lab)
```
login block-for 120 attempts 3 within 60
ip access-list standard MGMT-EXCEPTION
 permit 192.168.10.10
login quiet-mode access-class MGMT-EXCEPTION
```

### Task 5 — Restrict Management Access by Source (Tie-In from ACL Labs)
```
access-list 10 permit host 192.168.10.10
line vty 0 4
 access-class 10 in
```

### Task 6 — Verify Part 1
```
! From PC1 (should succeed)
ssh -l netadmin 192.168.10.1
! From SRV1 (should fail — not in the permitted access-class)
ssh -l netadmin 192.168.10.1
```

---

## Part 2 — Disable Unused and Insecure Services

### Task 7 — Encrypt Locally Stored Passwords
```
service password-encryption
```
> This applies a weak, reversible obfuscation (type 7) to plaintext passwords stored in the configuration — it is **not** strong encryption and should never be relied on as the sole protection for sensitive credentials (use `secret`, which applies real hashing, wherever possible instead), but it does prevent passwords from being trivially readable by anyone glancing at an unlocked terminal or a config backup file.

### Task 8 — Disable Unused Management Protocols
Disable services that are enabled by default (or easily left enabled) but rarely needed, and each represent unnecessary attack surface if left on:
```
no ip http server
no ip bootp server
no service finger
no service pad
no service config
```
> `no ip http server` assumes you are **not** relying on the router's web management interface at all — if you completed the HTTP Service Verification or HTTPS Configuration labs and specifically want HTTPS management retained, use `no ip http server` (disabling plaintext only) while keeping `ip http secure-server` enabled instead, rather than disabling both.

### Task 9 — Disable IP Source Routing
```
no ip source-route
```
> IP source routing allows a packet to specify its own path through the network, which can be abused to bypass normal routing/security controls — there is essentially never a legitimate reason to leave this enabled on a modern network.

### Task 10 — Disable CDP on External-Facing Interfaces
CDP (Cisco Discovery Protocol) is useful for internal troubleshooting but leaks detailed device information (platform, IOS version, IP address) to anything connected on that segment — disable it specifically on any interface facing outside your trust boundary, while leaving it enabled internally if it's still useful for your own operations:
```
! Example: if Gi0/0 in this lab represented an untrusted/external-facing segment
interface GigabitEthernet0/0
 no cdp enable
```
> In this lab's simple topology, Gi0/0 faces the internal LAN, so disabling CDP here is somewhat illustrative rather than strictly necessary — the important habit is identifying which interfaces on a real device face untrusted segments and disabling CDP (and LLDP, similarly) specifically there.

### Task 11 — Verify Part 2
```
show ip http server status
show run | include ip source-route
show cdp interface
show run | section service
```
Confirm each disabled service no longer appears as active/enabled.

---

## Part 3 — Switch Hardening (SW1)

### Task 12 — Apply Port Security to All Active Ports (from the Port Security labs)
```
interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable

interface FastEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### Task 13 — Shut Down All Unused Physical Ports
This is one of the single highest-value, lowest-effort hardening steps available on any switch, and one of the most commonly skipped in practice:
```
interface range FastEthernet0/3 - 24
 shutdown
 switchport mode access
 switchport access vlan 999
```
> Assigning unused ports to an unused, non-routed "parking" VLAN (999 in this example) in addition to shutting them down provides defense in depth — even if a port were accidentally re-enabled later without careful review, it wouldn't land on a production VLAN by default.

### Task 14 — Configure Errdisable Recovery
```
errdisable recovery cause psecure-violation
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

### Task 15 — Verify Part 3
```
show interfaces status
show port-security
show vlan brief
```
Confirm active ports (Fa0/1, Fa0/2) show correct port security, and unused ports (Fa0/3–24) show as administratively down and assigned to the parking VLAN.

---

## Part 4 — Final Full-Device Verification

### Task 16 — Confirm Nothing Was Broken by the Hardening Pass
```
! From PC1
ssh -l netadmin 192.168.10.1
ping 192.168.10.20   (SRV1, to confirm normal traffic still flows)
```
Both should succeed — hardening should never be verified solely by confirming restrictions work; always also confirm legitimate, intended functionality survived the changes.

### Task 17 — Build a Hardening Checklist from This Lab
In your lab notes, compile everything configured in this lab into a single reusable checklist (item, command, and one-line rationale) that you could apply to a new device in the future without needing to re-derive each step from scratch. This checklist itself is a deliverable of the lab, not just the configuration.

### Task 18 — Final Verification
```
show run
show ip ssh
show login
show port-security
show interfaces status
show cdp interface
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 6 — SSH + ACL restriction | PC1 succeeds; SRV1 (not in access-class) fails |
| Task 11 — disabled services | `ip http server`, bootp, finger, pad, and config service all confirmed disabled |
| Task 11 — CDP | Disabled on the designated interface |
| Task 15 — active ports | Fa0/1/Fa0/2 show correct port security configuration |
| Task 15 — unused ports | Fa0/3–24 administratively down, assigned to parking VLAN 999 |
| Task 16 — functionality preserved | Legitimate SSH access and normal traffic (PC1 → SRV1) both still work after all hardening changes |

---

## Challenge (Optional)
- Add the AAA (RADIUS/TACACS+), HTTPS/PKI trustpoint, and SNMPv3 configurations from their respective earlier labs into this same device, producing a genuinely complete hardened baseline, and confirm all layers coexist correctly without conflicting.
- Research the concept of a **hardening baseline/benchmark** (e.g., CIS Benchmarks) for network devices, and compare at least three items from a real published benchmark against what this lab covered — note any significant gaps this lab didn't address.
- Write a short prioritization argument: if you could only implement **three** of this lab's hardening items on a resource- and time-constrained real network, which three would you choose and why? There's no single correct answer, but your reasoning should reference the actual risk each control addresses, not just personal preference.