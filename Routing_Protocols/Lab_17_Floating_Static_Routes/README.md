# Lab: Floating Static Routes

## Objective
This lab isolates **one specific skill**, going deeper than the brief floating-static task in the Enhanced Static Routing lab: using floating static routes as a **backup to a dynamic routing protocol** (not just a backup to another static route), including **multiple tiers** of backup paths at increasing administrative distances, and testing failure scenarios beyond a simple interface shutdown — specifically, a scenario where the primary **link stays up** but the **routing protocol itself stops working**, which a naive "watch the interface" approach cannot detect.

---

## Topology

```
                          192.168.1.0/24
                          (Branch LAN)
                                |
                              SW-A
                                |
                        Gi0/0   |  192.168.1.1/24
                       +---------------+
                       |      R-A      |
                       +---------------+
                Gi0/1  |       Gi0/2  |       Gi0/3
        10.0.1.0/30    |   10.0.2.0/30 |   10.0.3.0/30
        (Primary,      |   (Backup 1,  |   (Backup 2,
         OSPF)          |    static)   |    static)
                        |               |
                        +-------+-------+-------+
                                |
                       +---------------+
                       |      R-B      |
                       +---------------+
                        Gi0/1 |  192.168.100.0/24
                                (destination LAN, simulated remote site)
```

- **R-A** and **R-B** are connected by **three** separate links: a **primary** link running OSPF, and two additional **static-only** backup links at different administrative distances, forming a tiered failover chain.
- Only the primary link runs a dynamic routing protocol — the two backup links exist purely to carry floating static routes.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|--------------------|
| R-A    | Gi0/0     | 192.168.1.1        | 255.255.255.0       |
| R-A    | Gi0/1     | 10.0.1.1              | 255.255.255.252     |
| R-A    | Gi0/2     | 10.0.2.1              | 255.255.255.252     |
| R-A    | Gi0/3     | 10.0.3.1              | 255.255.255.252     |
| R-B    | Gi0/1     | 10.0.1.2              | 255.255.255.252     |
| R-B    | Gi0/2     | 10.0.2.2              | 255.255.255.252     |
| R-B    | Gi0/3     | 10.0.3.2              | 255.255.255.252     |
| R-B    | Gi0/4     | 192.168.100.1        | 255.255.255.0       |

---

## Tasks

### Task 1 — Build the Topology
1. Place R-A, R-B, and all three links between them as shown, plus R-A's LAN.
2. Apply the addressing above and `no shutdown` all interfaces.

### Task 2 — Configure OSPF on the Primary Link Only
```
! R-A
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.1.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0

! R-B
router ospf 1
 router-id 2.2.2.2
 network 10.0.1.0 0.0.0.3 area 0
 network 192.168.100.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/4
```
> Note the backup links (Gi0/2 and Gi0/3 on both routers) are **not** included in OSPF at all — they exist purely for the static routes you'll configure next.

### Task 3 — Verify OSPF Baseline
```
show ip ospf neighbor
show ip route ospf
```
Confirm the OSPF adjacency is `FULL` and R-A has learned `192.168.100.0/24` via OSPF (administrative distance 110).

### Task 4 — Configure Tier 1 (First) Floating Static Backup
Add a static route via the first backup link, with an AD higher than OSPF's 110 so it stays out of the routing table while OSPF is healthy:
```
! On R-A
ip route 192.168.100.0 255.255.255.0 10.0.2.2 150
```

### Task 5 — Configure Tier 2 (Second) Floating Static Backup
Add a second, even-less-preferred static route via the third link:
```
! On R-A
ip route 192.168.100.0 255.255.255.0 10.0.3.2 200
```

### Task 6 — Verify the Tiered Configuration Is Present but Inactive
```
show ip route 192.168.100.0
show ip route static
```
Confirm only the OSPF route (AD 110) is currently installed and active — both floating static routes are configured but neither appears in the active routing table yet.

### Task 7 — Test Tier 1 Failover via Interface Shutdown
```
! On R-A
interface GigabitEthernet0/1
 shutdown
```
```
show ip route 192.168.100.0
```
Confirm the **Tier 1** floating static route (AD 150, via 10.0.2.2) is now installed. Verify reachability:
```
ping 192.168.100.1 source 192.168.1.1
```
Should succeed via the Tier 1 backup. Restore the primary link:
```
interface GigabitEthernet0/1
 no shutdown
```
and confirm the OSPF route resumes priority once the adjacency reconverges.

### Task 8 — Test the Scenario a Simple Interface Check Cannot Detect
This is the core exercise of the lab. Simulate a failure where the **primary link's interface stays up**, but the routing protocol itself stops functioning correctly — for example, an OSPF process crash, a misconfigured area mismatch introduced by mistake, or an authentication failure introduced without the interface itself going down. Simulate this by deliberately breaking the OSPF area assignment on R-B's primary-facing interface without touching the interface's link state:
```
! On R-B — introduce an area mismatch (Gi0/1 was area 0, now incorrectly area 5)
router ospf 1
 no network 10.0.1.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 5
```
```
! On R-A
show ip ospf neighbor
```
Confirm the OSPF adjacency has dropped — **even though `show ip interface brief` on both routers still shows Gi0/1 as `up/up`**. This is exactly the scenario a naive "assume the link works if the interface is up" design would miss entirely.

### Task 9 — Verify Automatic Failover Despite the Interface Being Up
```
! On R-A
show ip route 192.168.100.0
show ip interface brief
```
Confirm the Tier 1 floating static route has been installed automatically — because OSPF removed the now-invalid route from R-A's routing table (since the adjacency dropped) even though Gi0/1 itself remains up. Verify reachability:
```
ping 192.168.100.1 source 192.168.1.1
```
Should still succeed via Tier 1.

### Task 10 — Test Tier 2 by Also Failing the Tier 1 Backup
```
! On R-A
interface GigabitEthernet0/2
 shutdown
```
```
show ip route 192.168.100.0
```
Confirm the **Tier 2** floating static route (AD 200, via 10.0.3.2) is now installed — the full three-tier chain (OSPF → Tier 1 static → Tier 2 static) has now been exercised, one failure at a time.

### Task 11 — Restore Everything and Confirm Recovery Order
Restore in this order and verify the expected route re-takes priority after each step:
```
! Step 1: fix the OSPF area mismatch on R-B
router ospf 1
 no network 10.0.1.0 0.0.0.3 area 5
 network 10.0.1.0 0.0.0.3 area 0
```
```
! On R-A, check after Step 1
show ip route 192.168.100.0
```
Confirm the OSPF route (AD 110) resumes priority automatically, even though Gi0/1 was never shut down in this failure — recovery is just as automatic as failover was.
```
! Step 2: restore the Tier 1 backup interface
interface GigabitEthernet0/2
 no shutdown
```

### Task 12 — Verify
```
show ip route 192.168.100.0
show ip route static
show ip ospf neighbor
show run | section ip route
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — OSPF baseline | Adjacency `FULL`; route to 192.168.100.0/24 via OSPF (AD 110) |
| Task 6 — floating routes configured but inactive | Only the OSPF route is installed; both static backups exist in config but not in the active table |
| Task 7 — Tier 1 via interface shutdown | Tier 1 static route (AD 150) installs; reachability maintained |
| Task 8 — area mismatch introduced | OSPF adjacency drops; **interface remains `up/up`** on both routers |
| Task 9 — Tier 1 via routing-protocol failure (not interface failure) | Tier 1 static route installs automatically despite the interface being up — the key demonstration of this lab |
| Task 10 — Tier 1 also down | Tier 2 static route (AD 200) installs |
| Task 11 — recovery | OSPF route resumes priority automatically once the area mismatch is fixed, without needing the interface to be touched |

---

## Challenge (Optional)
- Combine this lab with IP SLA object tracking (from the Enhanced Static Routing lab) on one of the backup links, so that even the floating static routes themselves can react to a failure beyond their own directly-connected interface, not just OSPF's.
- Reorder the AD values so Tier 2's path is actually the physically shorter/faster link, and discuss in your lab notes why administrative distance ordering should reflect your actual preferred failover order, not just be assigned arbitrarily.
- Write a short incident-response-style note explaining why Task 8's scenario (routing protocol failure with the interface remaining up) is a category of failure worth specifically testing for in a real network design review, rather than assuming "if the interface is up, the path works."