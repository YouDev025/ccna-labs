# Lab 01: Connectivity Troubleshooting

## Objective
Practice a **systematic, bottom-up troubleshooting method** for basic connectivity problems: checking physical/interface status first, then IP addressing and subnetting, then using `ping` and `traceroute` to isolate exactly where a failure occurs. As in the ACL Troubleshooting lab, you'll start from a topology that's **already configured but broken** — four distinct, realistic mistakes have been introduced across different layers. Find and fix each one methodically, rather than guessing.

---

## Topology

```
  192.168.1.0/24                 10.0.12.0/30                 192.168.2.0/24
  (PC1's LAN)                    (R1 <-> R2 link)              (PC2's LAN)
        |                                                             |
      SW1                                                           SW2
        |                                                             |
  Gi0/0 | .1                                                   Gi0/1 | .1
  +-----------+     Gi0/1              Gi0/0     +-----------+
  |    R1     |------------------------------------|    R2     |
  +-----------+     10.0.12.1        10.0.12.2     +-----------+
        |                                                             |
      PC1                                                           PC2
  192.168.1.10                                                192.168.2.10
```

- A simple two-router topology, deliberately kept small so the troubleshooting method — not topology complexity — is the focus.
- **Intended result**: PC1 should be able to ping PC2 successfully. It currently cannot, for four separate, unrelated reasons.

---

## IP Addressing Table (Intended — What the Network *Should* Look Like)

| Device | Interface | IP Address     | Subnet Mask       |
|--------|-----------|-----------------|---------------------|
| R1     | Gi0/0     | 192.168.1.1      | 255.255.255.0        |
| R1     | Gi0/1     | 10.0.12.1          | 255.255.255.252      |
| R2     | Gi0/0     | 10.0.12.2          | 255.255.255.252      |
| R2     | Gi0/1     | 192.168.2.1      | 255.255.255.0        |
| PC1    | NIC       | 192.168.1.10      | 255.255.255.0        |
| PC2    | NIC       | 192.168.2.10      | 255.255.255.0        |

---

## Task 1 — Build the Topology and Load the Broken Configuration
1. Build the topology exactly as shown.
2. Apply the following **starting configuration**, exactly as written — this is the broken state you must diagnose:

```
! On R1
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface GigabitEthernet0/1
 ip address 10.0.12.1 255.255.255.252
 shutdown
ip route 192.168.2.0 255.255.255.0 10.0.12.2
```

```
! On R2
interface GigabitEthernet0/0
 ip address 10.0.12.2 255.255.255.0
 no shutdown
interface GigabitEthernet0/1
 ip address 192.168.2.1 255.255.255.0
 no shutdown
ip route 192.168.1.0 255.255.255.0 10.0.12.1
```

```
! PC1: IP 192.168.1.10, mask 255.255.255.0, gateway 192.168.1.1
! PC2: IP 192.168.2.11 (note: NOT .10), mask 255.255.255.0, gateway 192.168.2.1
```

3. Do not fix anything yet — apply exactly as written, then begin the systematic diagnosis in Task 2.

---

## Task 2 — Establish the Symptom
```
! From PC1
ping 192.168.2.10
```
Confirm this fails. This is your starting point — everything from here follows a structured method, not random guessing.

---

## Task 3 — Layer 1/Physical: Check Interface Status First

### Why This Comes First
Before checking IP addressing or routing, always confirm the physical/data-link layer is actually up — no amount of correct IP configuration matters if the interface itself is down.

```
! On R1 and R2
show ip interface brief
```

### Find Bug #1
Confirm R1's Gi0/1 shows **administratively down** — it was explicitly shut down in the starting configuration.

### Fix Bug #1
```
! On R1
interface GigabitEthernet0/1
 no shutdown
```
Re-check:
```
show ip interface brief
```
Confirm Gi0/1 now shows `up/up`.

### Retest
```
! From PC1
ping 192.168.2.10
```
Still fails — there are more problems to find. This confirms Bug #1 was real but not the only issue, which is an important habit: don't assume one fix solved everything just because it was clearly wrong.

---

## Task 4 — Layer 3: Verify IP Addressing and Subnetting

### Why This Comes Next
With Layer 1/2 confirmed healthy, the next most common source of connectivity problems is incorrect IP addressing — including subnet masks, which are easy to overlook since the interface can still come up `up/up` even with a wrong mask.

```
! On R2
show ip interface brief
show run interface GigabitEthernet0/0
```

### Find Bug #2
Compare R2's Gi0/0 configuration against the intended addressing table: it's configured with **255.255.255.0** instead of the intended **255.255.255.252** for the R1–R2 point-to-point link. This doesn't necessarily break the interface itself, but it does mean R2 has an incorrect understanding of what network 10.0.12.0 actually covers — with a /24 instead of a /30, R2 believes a much larger range of addresses is directly connected to this interface than is actually true.

### Fix Bug #2
```
! On R2
interface GigabitEthernet0/0
 ip address 10.0.12.2 255.255.255.252
```

### Retest
```
! From PC1
ping 192.168.2.10
```
Still fails.

---

## Task 5 — Verify End-Device Addressing (Not Just Router Interfaces)
It's easy to focus troubleshooting entirely on routers and forget to double-check the end devices themselves.
```
! On PC2
ipconfig /all   (or platform equivalent)
```

### Find Bug #3
Confirm PC2 is actually configured with **192.168.2.11**, not **192.168.2.10** as the addressing table intended — PC1 has been pinging an address that simply isn't assigned to anything.

### Fix Bug #3
Reconfigure PC2 with the correct address, 192.168.2.10.

### Retest
```
! From PC1
ping 192.168.2.10
```
Still fails, but note: also try
```
ping 192.168.2.11
```
This should now succeed — proving R1, R2, and the routing between them are actually working correctly at this point; PC1 was simply pinging an address that didn't exist. This is a useful diagnostic technique in its own right: if a specific address doesn't respond, try pinging something else on the same segment to determine if the problem is that one address, or the whole path.

---

## Task 6 — Use Traceroute to Confirm the Path (Before Assuming Everything Is Fixed)
```
! From PC1
traceroute 192.168.2.10
```

### Find Bug #4
Depending on exactly how Bug #3 was fixed and whether PC2 retained any stale configuration, you may find the traceroute stops at R2 or shows unexpected behavior at the final hop, if PC2's gateway or mask wasn't also corrected consistently. Carefully re-verify **all** of PC2's settings (address, mask, gateway) — not just the one field identified in Task 5 — since a single deliberately-introduced bug can sometimes have follow-on effects if only partially corrected. Confirm PC2's full configuration now exactly matches the addressing table.

### Retest
```
! From PC1
ping 192.168.2.10
traceroute 192.168.2.10
```
Both should now succeed, with `traceroute` showing the expected two-hop path: PC1 → R1 → R2 → PC2.

---

## Task 7 — Full Regression Test
```
! From PC1
ping 192.168.2.10

! From PC2
ping 192.168.1.10
```
Both directions should now succeed.

---

## Task 8 — Final Verification
```
show ip interface brief    (on both routers)
show ip route              (on both routers)
show run interface <each interface>
```
On PC1 and PC2:
```
ipconfig /all
```

---

## Verification Checklist

| Bug | Layer | Symptom | Root Cause | Fix |
|---|---|---|---|---|
| #1 | Physical/Interface | R1's link to R2 never comes up | Gi0/1 administratively shut down | `no shutdown` |
| #2 | IP Addressing/Subnetting | Interface up, but not necessarily obvious without checking | Wrong subnet mask on R2's Gi0/0 (/24 instead of /30) | Corrected to 255.255.255.252 |
| #3 | End-device addressing | Ping to the "expected" address fails even though the path works | PC2 configured with .11 instead of .10 | Reconfigured PC2 to .10 |
| #4 | Consistency check | Residual/incomplete fix | PC2's other settings not fully re-verified after Bug #3's fix | Full re-check of PC2's address, mask, and gateway together |
| Final regression test | — | — | — | Both directions succeed |

---

## Verification Checklist (Method)

| Step | Confirms |
|---|---|
| `show ip interface brief` | Interface administrative and line-protocol status |
| `show run interface <if>` | Configured IP address and mask, compared against the intended addressing table |
| End-device `ipconfig` | Address, mask, and gateway actually applied at the host, not just assumed |
| `ping` to a *different* address on the same segment | Distinguishes "this one address doesn't exist" from "the whole path is broken" |
| `traceroute` | Confirms the actual hop-by-hop path matches what's expected, catching problems a simple ping success/failure might not fully reveal |

---

## Challenge (Optional)
- Reintroduce Bug #2 (the wrong subnet mask) on a different interface of your own choosing, hand the configuration to a lab partner without telling them what's wrong, and have them apply this lab's bottom-up method (physical → addressing → end-device → path verification) to find it independently.
- Write a personal "connectivity troubleshooting checklist" based on this lab's four bug categories (interface status, subnet mask correctness, end-device addressing, and full-configuration consistency after a partial fix) that you can reuse on future labs or real equipment.
- Deliberately introduce a **default gateway** misconfiguration on PC1 (point it at the wrong address) instead of any of this lab's four bugs, and diagnose it using the same method — note specifically how the symptom differs from the four bugs in this lab (e.g., what does `ping` show when the gateway itself is wrong, versus when the destination doesn't exist, versus when an interface is down)?