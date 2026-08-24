# Lab: Routing Summarization

## Objective
This lab isolates **one specific skill**: correctly calculating and configuring **route summarization**, and understanding why it matters (smaller routing tables, faster lookups, better fault isolation). You'll practice the **binary math** that determines whether a summary is valid and tight, then configure the same summarization goal **two different ways** — manually via EIGRP interface summarization, and via OSPF area range — so you can compare the mechanics of achieving the same result in a distance-vector versus link-state protocol.

---

## Topology

```
                            192.168.16.0/24
                            192.168.17.0/24
                            192.168.18.0/24
                            192.168.19.0/24
                            (four contiguous branch subnets)
                                    |
                                  SW-BR
                                    |
                            Gi0/0-Gi0/3 | .1 each
                            +---------------+
                            |    R-BR       |   Branch router
                            +---------------+
                             Gi0/4 | 10.0.1.1/30
                                    |
                            +---------------+
                            |   R-CORE      |   Core/summarization point
                            +---------------+
```

- **R-BR** has four contiguous branch LANs directly connected.
- **R-CORE** is where the summarized route should appear — instead of four separate /24 entries, it should see (at most) one summary route, once configured correctly.
- This lab is run in two passes on the same topology: once with **EIGRP**, once with **OSPF** — configure, verify, and tear down one before starting the other, so each protocol's summarization behavior is observed cleanly.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|--------------------|
| R-BR   | Gi0/0     | 192.168.16.1        | 255.255.255.0       |
| R-BR   | Gi0/1     | 192.168.17.1        | 255.255.255.0       |
| R-BR   | Gi0/2     | 192.168.18.1        | 255.255.255.0       |
| R-BR   | Gi0/3     | 192.168.19.1        | 255.255.255.0       |
| R-BR   | Gi0/4     | 10.0.1.1              | 255.255.255.252     |
| R-CORE | Gi0/0     | 10.0.1.2              | 255.255.255.252     |

---

## Part 1 — The Math (Do This Before Configuring Anything)

### Task 1 — Convert Each Subnet to Binary
Write out the third octet of each subnet in binary:

| Subnet | Third Octet (Decimal) | Third Octet (Binary) |
|---|---|---|
| 192.168.16.0/24 | 16 | 00010000 |
| 192.168.17.0/24 | 17 | 00010001 |
| 192.168.18.0/24 | 18 | 00010010 |
| 192.168.19.0/24 | 19 | 00010011 |

### Task 2 — Identify the Common Prefix Bits
Line up the four binary values:
```
00010000
00010001
00010010
00010011
```
The first **6 bits** (`000100`) are identical across all four; the last 2 bits vary (`00`, `01`, `10`, `11` — covering all four combinations exactly, with no gaps and no extra addresses). This means these four /24s summarize perfectly into a single block.

### Task 3 — Calculate the Summary Prefix Length
- Fixed bits so far: 24 (the first three octets are fully fixed at 192.168.X where the third octet's first 6 bits are also fixed) — more precisely: 16 bits for `192.168`, plus 6 more fixed bits in the third octet = **22 bits total**.
- Summary: **192.168.16.0/22**

### Task 4 — Verify the Summary Boundary Is Exact
A summary is only "safe" if it covers **exactly** the intended subnets — no more, no less. Confirm:
- `192.168.16.0/22` covers addresses from `192.168.16.0` through `192.168.19.255` — exactly the four /24s in this lab, nothing extra.
- If there were only **three** subnets (16, 17, 18) instead of four, a /22 summary would still technically "work" for reachability, but would also **incorrectly claim ownership** of the unused 192.168.19.0/24 block — a real-world summarization mistake that can cause traffic for an address that doesn't actually exist behind this router to be silently routed there anyway (and black-holed, since nothing on that subnet exists to receive it). This is exactly why the boundary-checking habit from Task 2 matters, not just finding *a* prefix length that happens to include everything.

---

## Part 2 — EIGRP Summarization

### Task 5 — Build the Topology and Configure Basic EIGRP
```
! R-BR
router eigrp 100
 network 192.168.16.0 0.0.0.255
 network 192.168.17.0 0.0.0.255
 network 192.168.18.0 0.0.0.255
 network 192.168.19.0 0.0.0.255
 network 10.0.1.0 0.0.0.3
 no auto-summary
 passive-interface GigabitEthernet0/0
 passive-interface GigabitEthernet0/1
 passive-interface GigabitEthernet0/2
 passive-interface GigabitEthernet0/3

! R-CORE
router eigrp 100
 network 10.0.1.0 0.0.0.3
 no auto-summary
```

### Task 6 — Verify the Unsummarized Baseline
On **R-CORE**:
```
show ip route eigrp
```
Confirm **four** separate `/24` entries — this is your "before" state.

### Task 7 — Configure EIGRP Manual Summarization
EIGRP summarization is configured **on the interface facing the direction you want the summary advertised**, using the prefix calculated in Task 3:
```
! On R-BR
interface GigabitEthernet0/4
 ip summary-address eigrp 100 192.168.16.0 255.255.252.0
```

### Task 8 — Verify EIGRP Summarization
On **R-CORE**:
```
show ip route eigrp
```
Confirm the four `/24` entries are now replaced by a **single** `192.168.16.0/22` entry. On **R-BR**:
```
show ip route
```
Note R-BR itself still shows all four individual /24s as directly connected — summarization only affects what's **advertised to neighbors**, not the originating router's own table.

### Task 9 — Verify Reachability and Tear Down for Part 3
```
! From a host on any of the four branch subnets (or from R-CORE itself)
ping 192.168.16.1
ping 192.168.19.1
```
Both should succeed. Remove EIGRP entirely before moving to Part 3:
```
! On both R-BR and R-CORE
no router eigrp 100
```

---

## Part 3 — OSPF Summarization

### Task 10 — Configure Basic OSPF Across Two Areas
To use `area range` summarization (rather than EIGRP's per-interface approach), the branch subnets need to be in a **different area** than R-CORE:
```
! R-BR
router ospf 1
 router-id 1.1.1.1
 network 192.168.16.0 0.0.0.255 area 1
 network 192.168.17.0 0.0.0.255 area 1
 network 192.168.18.0 0.0.0.255 area 1
 network 192.168.19.0 0.0.0.255 area 1
 network 10.0.1.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
 passive-interface GigabitEthernet0/1
 passive-interface GigabitEthernet0/2
 passive-interface GigabitEthernet0/3

! R-CORE
router ospf 1
 router-id 2.2.2.2
 network 10.0.1.0 0.0.0.3 area 0
```

### Task 11 — Verify the Unsummarized Baseline
On **R-CORE**:
```
show ip route ospf
```
Confirm four separate `O IA` entries.

### Task 12 — Configure OSPF Area Range Summarization
Unlike EIGRP's interface-level command, OSPF summarization is configured **on the ABR, for the area being summarized**, using the same /22 prefix from Task 3:
```
! On R-BR (the ABR between Area 1 and Area 0)
router ospf 1
 area 1 range 192.168.16.0 255.255.252.0
```

### Task 13 — Verify OSPF Summarization
On **R-CORE**:
```
show ip route ospf
```
Confirm the four `O IA` entries are replaced by a single `192.168.16.0/22` `O IA` entry.

### Task 14 — Verify Reachability
```
ping 192.168.16.1
ping 192.168.19.1
```
Both should succeed, confirming the OSPF summary is functioning identically to the EIGRP summary from Part 2, despite being configured completely differently.

### Task 15 — Final Verification and Comparison
```
show ip route
show ip protocols
show run | section router
```
In your lab notes, write a short comparison table covering: where each summarization command is configured (interface vs. area/ABR), what triggers it to take effect, and any wording/syntax differences you noticed between the two protocols for expressing the same /22 summary.

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 1–4 (math) | Correctly identifies /22 as the exact, tight summary boundary for the four given subnets |
| Task 6 — EIGRP baseline | Four separate /24 entries on R-CORE |
| Task 8 — EIGRP summarized | Single /22 entry on R-CORE; R-BR's own table still shows all four /24s individually |
| Task 9 — reachability | Succeeds to both the first and last subnet in the summarized range |
| Task 11 — OSPF baseline | Four separate `O IA` /24 entries on R-CORE |
| Task 13 — OSPF summarized | Single /22 `O IA` entry on R-CORE |
| Task 14 — reachability | Succeeds identically to the EIGRP test |

---

## Challenge (Optional)
- Add a **fifth** subnet, 192.168.20.0/24, which does **not** fall within the 192.168.16.0/22 boundary (it needs its own bit pattern check — do the binary math yourself to confirm this before assuming). Determine whether the existing summary can be extended to include it cleanly, needs a second separate summary, or should remain unsummarized on its own — implement your conclusion and verify it.
- Deliberately configure a **too-broad** summary (e.g., a /21 instead of the correct /22) and use `ping` to a nearby but genuinely non-existent address within the falsely-claimed range to observe the black-holing behavior described conceptually in Task 4 — document what you observe (likely a timeout, since the summarizing router will accept the traffic as matching its summary route but have nowhere real to forward it).
- Research and document how **static route summarization** (from the Enhanced Static Routing lab), **EIGRP interface summarization**, and **OSPF area range summarization** all ultimately rely on the exact same underlying binary-boundary math from Part 1 of this lab — despite being configured completely differently in each protocol.