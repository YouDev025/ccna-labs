# Lab: Packet Filtering

## Objective
Practice the **fundamentals of packet filtering** with Cisco IOS ACLs across a multi-router network: writing correct wildcard masks, understanding **standard vs. extended** ACL behavior, learning **where** an ACL should be placed (a classic source of real-world misconfiguration), observing the **implicit deny**, and confirming how ACL **processing order** (top-down, first-match) affects results.

This lab is intentionally broader and more conceptual than the Extended ACLs lab — it's about building correct filtering habits, not one specific policy.

---

## Topology

```
     192.168.1.0/24                 10.0.12.0/30                 10.0.23.0/30                 192.168.3.0/24
   (Branch A LAN)                  (A <-> B link)                (B <-> C link)                (Branch C LAN)
         |                                                                                              |
       SW-A                                                                                           SW-C
         |                                                                                              |
   Gi0/0 | .1                                                                                    Gi0/1 | .1
   +-----------+     Gi0/1      Gi0/0     +-----------+     Gi0/1      Gi0/0     +-----------+
   |    R-A    |-----------------------------|    R-B    |-----------------------------|    R-C    |
   +-----------+     10.0.12.1  10.0.12.2    +-----------+     10.0.23.2  10.0.23.1     +-----------+
         |                                        |                                            |
       PC-A1                                    PC-B1                                        PC-C1
   192.168.1.10                              (Admin host,                                192.168.3.10
                                              10.0.12... — see table)
```

- **R-A**, **R-B**, **R-C** form a simple three-router chain (Branch A — Branch B — Branch C).
- **PC-A1** and **PC-C1** are ordinary LAN hosts at each end.
- **R-B** (the middle router) also has a locally-attached **admin host, PC-B1**, used later to demonstrate why ACL placement matters.

---

## IP Addressing Table

| Device | Interface | IP Address        | Subnet Mask       | Default Gateway |
|--------|-----------|--------------------|---------------------|-----------------|
| R-A    | Gi0/0     | 192.168.1.1          | 255.255.255.0        | —               |
| R-A    | Gi0/1     | 10.0.12.1              | 255.255.255.252      | —               |
| R-B    | Gi0/0     | 10.0.12.2              | 255.255.255.252      | —               |
| R-B    | Gi0/1     | 10.0.23.2              | 255.255.255.252      | —               |
| R-B    | Gi0/2     | 192.168.99.1          | 255.255.255.0        | —               |
| R-C    | Gi0/0     | 10.0.23.1              | 255.255.255.252      | —               |
| R-C    | Gi0/1     | 192.168.3.1          | 255.255.255.0        | —               |
| PC-A1  | NIC       | 192.168.1.10          | 255.255.255.0        | 192.168.1.1     |
| PC-B1  | NIC       | 192.168.99.10          | 255.255.255.0        | 192.168.99.1    |
| PC-C1  | NIC       | 192.168.3.10          | 255.255.255.0        | 192.168.3.1     |

---

## Tasks

### Task 1 — Build the Topology
1. Place R-A, R-B, R-C, the three switches, and PC-A1, PC-B1, PC-C1 as shown.
2. Cable the inter-router links and each PC to its local switch.
3. Apply the addressing from the table above.

### Task 2 — Enable Routing
This lab focuses on filtering, not routing protocol design — use static routes (or a simple routing protocol like OSPF/EIGRP if your course has already covered it) so that **all three LANs can reach each other** before any ACL is applied:
```
! Example static routes on R-A
ip route 192.168.3.0 255.255.255.0 10.0.12.2
ip route 192.168.99.0 255.255.255.0 10.0.12.2

! Example static routes on R-C
ip route 192.168.1.0 255.255.255.0 10.0.23.2
ip route 192.168.99.0 255.255.255.0 10.0.23.2

! R-B already has all three networks directly or one hop away — add routes as needed
```
Verify full reachability: PC-A1, PC-B1, and PC-C1 can all ping each other with **no ACLs applied**.

### Task 3 — Wildcard Mask Practice (Do This on Paper/Notes First)
Before configuring anything, write the correct wildcard mask for each of the following (answers are provided so you can self-check before moving on):

| Requirement | Network/Host | Wildcard Mask |
|---|---|---|
| Match the entire 192.168.1.0/24 network | 192.168.1.0 | 0.0.0.255 |
| Match only a single host, 192.168.3.10 | 192.168.3.10 | 0.0.0.0 (or use `host 192.168.3.10`) |
| Match any address (all networks) | 0.0.0.0 | 255.255.255.255 (or use keyword `any`) |
| Match the 10.0.12.0/30 link only | 10.0.12.0 | 0.0.0.3 |
| Match 192.168.0.0 through 192.168.3.0 (a /22-style range) | 192.168.0.0 | 0.0.3.255 |

> Getting wildcard masks wrong is one of the most common real-world ACL mistakes — an inverted or miscalculated mask silently matches the wrong traffic (often far too much, or nothing at all) without any error message from the router.

### Task 4 — Standard ACL: Filter by Source Only
Create a standard ACL on **R-C** that blocks PC-A1 specifically from reaching the Branch C LAN, while allowing all other sources:
```
access-list 10 deny   host 192.168.1.10
access-list 10 permit any
```
**Placement rule for standard ACLs:** because they match on source address only, they should be applied **as close to the destination as possible** — otherwise they risk blocking that source's traffic to *other* destinations it should still be allowed to reach. Apply it outbound on R-C's LAN interface:
```
interface GigabitEthernet0/1
 ip access-group 10 out
```

### Task 5 — Test and Observe the Implicit Deny
From **PC-A1**, ping **PC-C1** → should fail (explicitly denied).
From **PC-B1**, ping **PC-C1** → should succeed (matches the `permit any` line).

Now temporarily remove the `permit any` line and re-test:
```
no access-list 10 permit any
```
From **PC-B1**, ping **PC-C1** again → should now **fail**, even though you never wrote a rule denying PC-B1. This demonstrates the **implicit deny** at the end of every ACL — restore the `permit any` line afterward:
```
access-list 10 permit any
```

### Task 6 — Extended ACL: Filter by Source, Destination, and Service
Create an extended ACL on **R-A** that only allows PC-A1 to reach PC-C1 on HTTP (80), denying everything else from PC-A1 to that specific host, while leaving all other traffic unaffected:
```
ip access-list extended PCA1-RESTRICT
 permit tcp host 192.168.1.10 host 192.168.3.10 eq 80
 deny   ip host 192.168.1.10 host 192.168.3.10
 permit ip any any
```
**Placement rule for extended ACLs:** because they can match on source, destination, and service together, they should be applied **as close to the source as possible** — this stops unwanted traffic at the first hop instead of wasting bandwidth carrying it across the whole network only to be dropped later. Apply it inbound on R-A's LAN interface:
```
interface GigabitEthernet0/0
 ip access-group PCA1-RESTRICT in
```

### Task 7 — Test Extended ACL Placement
From **PC-A1**:
- Connect to PC-C1 (or a simulated web server there) on port 80 → should succeed.
- Ping PC-C1 → should fail (blocked by the `deny ip` line before it ever leaves R-A).
- Confirm with `show access-lists PCA1-RESTRICT` on R-A that match counters increment **at R-A**, not further into the network — this is the placement benefit in action.

### Task 8 — Observe Processing Order (Top-Down, First Match)
On R-A, temporarily reorder the ACL to put the broad deny **before** the specific permit:
```
no ip access-list extended PCA1-RESTRICT
ip access-list extended PCA1-RESTRICT
 deny   ip host 192.168.1.10 host 192.168.3.10
 permit tcp host 192.168.1.10 host 192.168.3.10 eq 80
 permit ip any any
```
Retest HTTP from PC-A1 to PC-C1 → it now **fails**, even though the permit line is technically still present, because the router stops at the **first match** — the broad `deny` above it now catches all traffic first. Restore the original order from Task 6 afterward and confirm HTTP works again.
> This task exists specifically to build the habit of ordering ACL entries from **most specific to least specific**.

### Task 9 — Verify
On R-A and R-C:
```
show access-lists
show ip interface GigabitEthernet0/0 | include access list
show ip interface GigabitEthernet0/1 | include access list
show run | section access-list
show ip route
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Baseline (no ACLs) | PC-A1, PC-B1, PC-C1 all ping each other successfully |
| Standard ACL 10 on R-C (Task 4/5) | PC-A1 → PC-C1 blocked; PC-B1 → PC-C1 still works |
| Removing `permit any` from ACL 10 | PC-B1 → PC-C1 now also fails (implicit deny demonstrated), restored afterward |
| Extended ACL on R-A (Task 6/7) | PC-A1 → PC-C1 HTTP succeeds; PC-A1 → PC-C1 ping fails |
| `show access-lists PCA1-RESTRICT` on R-A | Match counters increment locally at R-A, confirming traffic is filtered at the source-side router |
| Reordered ACL (Task 8) | HTTP from PC-A1 → PC-C1 now fails due to the broad deny matching first; restoring original order fixes it |
| PC-B1 and PC-C1 traffic to each other (uninvolved in either ACL) | Unaffected throughout the entire lab |

---

## Challenge (Optional)
- Rebuild the standard ACL from Task 4 as an **extended** ACL that achieves the same source-only filtering result, and discuss in your lab notes why you would still choose the standard ACL for this specific requirement (simplicity, lower resource use, faster to write) despite extended ACLs being more "capable."
- Add a rule to `PCA1-RESTRICT` permitting only HTTPS (443) in addition to HTTP, and verify both work while all other services from PC-A1 to PC-C1 remain blocked.
- Deliberately misconfigure a wildcard mask (e.g., use `0.0.255.0` instead of `0.0.0.255` for a /24) and document exactly which unintended addresses get matched as a result — a hands-on demonstration of why wildcard-mask errors are dangerous in production.