# Lab: ACL Placement

## Objective
This lab isolates **one specific skill**: deciding **where** to apply an ACL in a multi-router, multi-path network. You will deliberately apply the *same* filtering intent at a **wrong** location first, observe the collateral damage it causes, then move it to the **correct** location and confirm the problem is resolved. The goal is to internalize the standard rule of thumb — **standard ACLs close to the destination, extended ACLs close to the source** — by seeing it fail and succeed side by side, rather than just being told the rule.

---

## Topology

```
   192.168.1.0/24                                                 192.168.3.0/24
   (Sales LAN)                                                    (Finance LAN)
        |                                                               |
      SW-1                                                            SW-3
        |                                                               |
  Gi0/0 | .1                                                     Gi0/1 | .1
  +-----------+   Gi0/1        Gi0/0   +-----------+   Gi0/1        Gi0/0  +-----------+
  |    R1     |------------------------|    R2     |------------------------|    R3     |
  +-----------+ 10.0.12.1    10.0.12.2 +-----------+ 10.0.23.2    10.0.23.1 +-----------+
                                              |
                                        Gi0/2 | .1
                                              |
                                            SW-2
                                              |
                                        192.168.2.0/24
                                        (IT LAN)
```

- **R1, R2, R3** form a chain; **R2** additionally has a third LAN (**IT**) directly attached.
- **Sales LAN** (behind R1) and **Finance LAN** (behind R3) are the two networks between which we want to restrict traffic.
- **IT LAN** (behind R2, in the middle) represents a third party whose traffic must **not** be affected by the Sales/Finance policy — this is what makes placement mistakes visible.

---

## IP Addressing Table

| Device  | Interface | IP Address     | Subnet Mask       | Default Gateway |
|---------|-----------|-----------------|---------------------|-----------------|
| R1      | Gi0/0     | 192.168.1.1      | 255.255.255.0        | —               |
| R1      | Gi0/1     | 10.0.12.1          | 255.255.255.252      | —               |
| R2      | Gi0/0     | 10.0.12.2          | 255.255.255.252      | —               |
| R2      | Gi0/1     | 10.0.23.2          | 255.255.255.252      | —               |
| R2      | Gi0/2     | 192.168.2.1      | 255.255.255.0        | —               |
| R3      | Gi0/0     | 10.0.23.1          | 255.255.255.252      | —               |
| R3      | Gi0/1     | 192.168.3.1      | 255.255.255.0        | —               |
| PC-Sales| NIC       | 192.168.1.10      | 255.255.255.0        | 192.168.1.1     |
| PC-IT   | NIC       | 192.168.2.10      | 255.255.255.0        | 192.168.2.1     |
| PC-Fin  | NIC       | 192.168.3.10      | 255.255.255.0        | 192.168.3.1     |

---

## Policy Requirement
**Sales must not be able to reach Finance.** IT's traffic to Finance (and everywhere else) must be completely unaffected.

This single, simple requirement is the whole lab — the point is discovering that **where** you enforce it matters as much as the rule itself.

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, R2, R3, the three switches, and PC-Sales, PC-IT, PC-Fin as shown.
2. Cable all links and apply the addressing from the table above.
3. Configure static routes (or a routing protocol already covered in your course) so all three LANs can reach each other:
   ```
   ! R1
   ip route 192.168.2.0 255.255.255.0 10.0.12.2
   ip route 192.168.3.0 255.255.255.0 10.0.12.2

   ! R2 has Sales and Finance one hop away in each direction; add routes as needed
   ip route 192.168.1.0 255.255.255.0 10.0.12.1
   ip route 192.168.3.0 255.255.255.0 10.0.23.1

   ! R3
   ip route 192.168.1.0 255.255.255.0 10.0.23.2
   ip route 192.168.2.0 255.255.255.0 10.0.23.2
   ```
4. Verify full reachability with **no ACLs applied**: PC-Sales, PC-IT, and PC-Fin can all ping each other.

### Task 2 — Attempt 1 (Deliberately Wrong Placement)
Build a standard ACL matching Sales as the source, and apply it **outbound on R2's Gi0/2 (the IT LAN interface)** — a plausible-looking but incorrect choice, since it happens to sit "in the middle" of the network:
```
access-list 10 deny   192.168.1.0 0.0.0.255
access-list 10 permit any
```
```
interface GigabitEthernet0/2
 ip access-group 10 out
```

### Task 3 — Test Attempt 1 and Observe the Problem
From **PC-Sales**, ping **PC-Fin** → fails, as intended.
From **PC-IT**, ping **PC-Fin** → should still succeed (this interface only filters traffic *leaving toward IT*, not toward Finance, so this particular test may still pass — check it anyway and note the result).
From **PC-Sales**, ping **PC-IT** → this traffic also egresses Gi0/2 outbound, so it is **also blocked** — this is the collateral damage: Sales was never supposed to be restricted from reaching IT, only Finance, but this placement caught both because it filters *all* outbound traffic on that interface regardless of final destination.

Confirm with:
```
show access-lists 10
```
and note that the match counter increments for **both** the Sales→Finance and Sales→IT test pings — proof this ACL cannot distinguish between them from this location.

### Task 4 — Remove the Incorrect Placement
```
interface GigabitEthernet0/2
 no ip access-group 10 out
```

### Task 5 — Attempt 2 (Correct Standard ACL Placement)
Standard ACLs match source only, so they must sit **as close to the destination as possible** to avoid catching traffic bound for other, unintended destinations. Apply the same ACL logic outbound on **R3's Gi0/1 (the Finance-facing interface)** instead:
```
interface GigabitEthernet0/1
 ip access-group 10 out
```

### Task 6 — Test Attempt 2
From **PC-Sales**, ping **PC-Fin** → fails, as intended.
From **PC-Sales**, ping **PC-IT** → now **succeeds**, since this ACL only evaluates traffic as it exits toward Finance specifically.
From **PC-IT**, ping **PC-Fin** → succeeds, confirming IT is completely unaffected.

Confirm with:
```
show access-lists 10
```
Now the match counter should increment **only** for the Sales→Finance test, not for Sales→IT.

### Task 7 — Repeat the Exercise with an Extended ACL (Opposite Rule)
Extended ACLs match source, destination, and service together, so — unlike standard ACLs — they should sit **as close to the source as possible**: this stops unwanted traffic immediately rather than letting it consume bandwidth across the whole network only to be dropped near the destination.

Remove the standard ACL from R3, then build and test an equivalent extended ACL misplaced at the destination first:
```
! On R3 — remove Task 5/6 config
interface GigabitEthernet0/1
 no ip access-group 10 out
```
```
! Extended ACL, same intent: Sales cannot reach Finance
ip access-list extended SALES-TO-FIN
 deny   ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
 permit ip any any
```
```
! Misplaced: applied at R3 (the destination) instead of R1 (the source)
interface GigabitEthernet0/1
 ip access-group SALES-TO-FIN out
```
This technically **works** for the Sales→Finance test (extended ACLs are precise enough that placement mistakes don't always cause collateral damage the way standard ACLs do) — but it is still **inefficient**: the denied traffic travels across the entire network (R1 → R2 → R3) only to be dropped at the very last hop, wasting bandwidth on every link along the way.

### Task 8 — Move the Extended ACL to the Correct (Source-Side) Location
```
! Remove from R3
interface GigabitEthernet0/1
 no ip access-group SALES-TO-FIN out

! Apply at R1 instead — as close to the source as possible
interface GigabitEthernet0/0
 ip access-group SALES-TO-FIN in
```

### Task 9 — Verify Efficient Placement
From **PC-Sales**, ping **PC-Fin** → still fails, as intended.
On **R1**:
```
show access-lists SALES-TO-FIN
```
Confirm the match counter increments **at R1**, meaning the traffic is dropped at the very first hop — never even reaching R2 or consuming bandwidth on the R2↔R3 link. Compare this conceptually to Task 7, where the same denied packets would have crossed two extra links before being dropped.

### Task 10 — Final Verification Across All Devices
```
show access-lists
show run | section access-list
show ip interface GigabitEthernet0/0 | include access list
show ip route
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Baseline (no ACLs) | All three PCs can ping each other |
| Standard ACL misplaced on R2 Gi0/2 (Task 2/3) | Sales→Finance blocked (correct), but Sales→IT **also** blocked (collateral damage — the bug this lab demonstrates) |
| Standard ACL correctly placed on R3 Gi0/1 (Task 5/6) | Sales→Finance blocked; Sales→IT succeeds; IT→Finance succeeds |
| Extended ACL misplaced on R3 Gi0/1 (Task 7) | Sales→Finance blocked correctly, but traffic unnecessarily traverses the entire path before being dropped |
| Extended ACL correctly placed on R1 Gi0/0 (Task 8/9) | Sales→Finance blocked at the very first hop; `show access-lists` match counter increments on R1, not R3 |
| IT traffic throughout the entire lab | Never affected by any configuration change, in any task |

---

## Challenge (Optional)
- Add a fourth LAN behind a new router attached to R3, and determine (before configuring anything) where a **new** standard ACL protecting that LAN should be placed — then configure it and prove your placement choice is correct using the same collateral-damage testing method from this lab.
- Using `show access-lists` match counters as your evidence, write a one-paragraph explanation (in your lab notes) of why the "wrong" extended ACL placement in Task 7 is a **bandwidth/efficiency problem** rather than a **security problem** — while the "wrong" standard ACL placement in Task 2/3 is a **correctness problem** (it blocks traffic it was never supposed to block).
- Combine both ACLs simultaneously (extended at R1, and a separate, correctly-scoped standard ACL elsewhere for a different requirement) and confirm they operate independently without interfering with each other.