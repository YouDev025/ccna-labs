# Lab: Enhanced Static Routing

## Objective
Go beyond basic static routes (see Lab 01: Static Routing) to practice the techniques that make static routing viable in more realistic networks: **floating static routes** for automatic backup paths, **IP SLA object tracking** so a floating route can react to a failure the interface itself doesn't detect, **route summarization** to reduce routing table size, and **equal-cost static load balancing** across redundant links.

---

## Topology

```
                              192.168.100.0/24
                              (simulated ISP / remote site)
                                       |
                                     R-ISP
                                    /      \
                        10.0.1.0/30         10.0.2.0/30
                       (Primary link)       (Backup link)
                                \            /
                                 +-----------+
                                 |    R-HQ   |
                                 +-----------+
                                       |
                          Gi0/2  |  10.0.10.1/24
                                       |
                                     SW-HQ
                            +----------+----------+
                            |                     |
                       192.168.10.0/24       192.168.11.0/24
                       (Branch subnet 1)     (Branch subnet 2)
                            |                     |
                         PC-A1                 PC-A2
```

- **R-HQ** has two paths to reach `192.168.100.0/24` (representing an ISP or remote site): a **primary** link and a **backup** link.
- **R-ISP** simulates the far side of both links, so you can bring the primary down and confirm failover without needing a full second ISP router.
- Two subnets behind R-HQ (192.168.10.0/24 and 192.168.11.0/24) are used later for the route summarization task.

---

## IP Addressing Table

| Device  | Interface | IP Address       | Subnet Mask       |
|---------|-----------|-------------------|---------------------|
| R-HQ    | Gi0/0     | 10.0.1.1            | 255.255.255.252      |
| R-HQ    | Gi0/1     | 10.0.2.1            | 255.255.255.252      |
| R-HQ    | Gi0/2     | 10.0.10.1            | 255.255.255.0        |
| R-ISP   | Gi0/0     | 10.0.1.2            | 255.255.255.252      |
| R-ISP   | Gi0/1     | 10.0.2.2            | 255.255.255.252      |
| R-ISP   | Gi0/2     | 192.168.100.1        | 255.255.255.0        |
| PC-A1   | NIC       | 192.168.10.10        | 255.255.255.0        |
| PC-A2   | NIC       | 192.168.11.10        | 255.255.255.0        |

> R-HQ's Gi0/2 subnet (10.0.10.0/24) connects to a switch trunked with two VLANs for PC-A1 and PC-A2's subnets — or use two separate physical interfaces/subinterfaces if your platform doesn't support the VLAN setup shown; either is fine, since this detail isn't the focus of the lab.

---

## Tasks

### Task 1 — Build the Topology and Assign Addressing
1. Place R-HQ, R-ISP, SW-HQ, PC-A1, and PC-A2 as shown.
2. Apply the addressing above and `no shutdown` all router interfaces.
3. On R-ISP, add routes back to both HQ subnets so return traffic works for testing:
   ```
   ip route 192.168.10.0 255.255.255.0 10.0.1.1
   ip route 192.168.11.0 255.255.255.0 10.0.1.1
   ```
   > Only routing via the primary link's next hop is needed here — R-ISP doesn't need to know about the backup path for this lab's purposes, since the failover behavior being tested is entirely on R-HQ's side.

---

## Part 1 — Floating Static Routes

### Task 2 — Configure the Primary Static Route
```
ip route 192.168.100.0 255.255.255.0 10.0.1.2
```

### Task 3 — Configure a Floating Static Backup Route
Add a second route to the same destination via the backup link, but with a **higher administrative distance** so it only becomes active if the primary route disappears from the routing table entirely:
```
ip route 192.168.100.0 255.255.255.0 10.0.2.2 200
```
> The default administrative distance for a static route is 1. By setting this backup route's AD to 200 (higher = less preferred), it will never be installed in the routing table while the AD-1 primary route is present and valid — hence "floating": it stays out of the table until needed.

### Task 4 — Verify the Floating Route Is Currently Inactive
```
show ip route 192.168.100.0
show ip route static
```
Confirm only the **primary** route (AD 1, via 10.0.1.2) appears in the active routing table — the floating backup route should not be shown as installed, even though it's configured.

### Task 5 — Test Failover by Shutting Down the Primary Interface
```
interface GigabitEthernet0/0
 shutdown
```
```
show ip route 192.168.100.0
```
Confirm the floating backup route (via 10.0.2.2) is now installed automatically, with no manual intervention. Test reachability:
```
! From PC-A1
ping 192.168.100.1
```
Should still succeed, now via the backup path. Restore the primary link:
```
interface GigabitEthernet0/0
 no shutdown
```
and confirm the primary route resumes priority once it's back up.

---

## Part 2 — IP SLA Object Tracking (Failover Without an Interface Going Down)

### Task 6 — Understand the Gap Interface-Only Failover Leaves
Task 5's failover worked because the interface itself went down — but in real networks, a path can fail **beyond** the directly-connected interface (e.g., the ISP's next hop stops responding, but your local interface stays up). A plain floating static route configured as in Task 3 would **not** detect that kind of failure, since the primary route would still appear valid to R-HQ. IP SLA tracking solves this by actively probing reachability, not just watching interface state.

### Task 7 — Configure an IP SLA Probe
```
ip sla 1
 icmp-echo 10.0.1.2 source-interface GigabitEthernet0/0
 frequency 5
ip sla schedule 1 life forever start-time now
```

### Task 8 — Create a Track Object Tied to the Probe
```
track 1 ip sla 1 reachability
```

### Task 9 — Tie the Primary Static Route to the Track Object
Replace the plain primary route from Task 2 with a tracked version:
```
no ip route 192.168.100.0 255.255.255.0 10.0.1.2
ip route 192.168.100.0 255.255.255.0 10.0.1.2 track 1
```

### Task 10 — Verify Tracking State
```
show track 1
show ip route 192.168.100.0
```
Confirm the track object shows **Up**, and the primary route remains installed.

### Task 11 — Simulate a Failure Beyond the Interface
Instead of shutting down R-HQ's own interface (which Task 5 already tested), simulate a failure where the interface stays up but the next hop stops responding — for example, by shutting down R-ISP's matching interface instead:
```
! On R-ISP
interface GigabitEthernet0/0
 shutdown
```
On R-HQ:
```
show track 1
show ip route 192.168.100.0
```
Confirm the track object transitions to **Down** once the probe fails (allow a few probe cycles based on the `frequency` set in Task 7), and the floating backup route is installed — even though R-HQ's own Gi0/0 interface never went down. This is the specific capability plain floating static routes (Task 3 alone) cannot provide.

Restore R-ISP's interface:
```
interface GigabitEthernet0/0
 no shutdown
```
and confirm the track object returns to **Up** and the primary route is reinstated.

---

## Part 3 — Route Summarization

### Task 12 — Advertise Two Subnets Toward R-ISP as One Summary (Conceptual/Manual)
R-HQ's two branch subnets, 192.168.10.0/24 and 192.168.11.0/24, can be represented by a single summarized route: 192.168.10.0/23 (covering both /24s in one entry). Since this lab uses static routing rather than a dynamic protocol capable of automatic summarization, configure R-ISP with a single summarized static route back to R-HQ instead of two separate ones, to see the effect directly:
```
! On R-ISP — replace the two /24 routes from Task 1 with one summary
no ip route 192.168.10.0 255.255.255.0 10.0.1.1
no ip route 192.168.11.0 255.255.255.0 10.0.1.1
ip route 192.168.10.0 255.255.254.0 10.0.1.1
```
> `255.255.254.0` is the /23 mask covering both 192.168.10.0/24 and 192.168.11.0/24 in a single entry — verify this mask covers exactly (and only) the intended two subnets before using summarization in a real network, since an incorrect summary boundary can accidentally include or exclude unintended addresses.

### Task 13 — Verify the Summary Route
```
! On R-ISP
show ip route static
```
Confirm a single `/23` entry now exists instead of two `/24` entries.
```
! From PC-A1 and PC-A2
ping 192.168.100.1
```
Both should still succeed, confirming the summary correctly covers both subnets.

---

## Part 4 — Equal-Cost Static Load Balancing

### Task 14 — Configure Two Equal-AD Routes to the Same Destination
Temporarily set up both paths to `192.168.100.0/24` at the **same** administrative distance, rather than a primary/backup relationship, to enable load balancing instead of failover:
```
no ip route 192.168.100.0 255.255.255.0 10.0.1.2 track 1
no ip route 192.168.100.0 255.255.255.0 10.0.2.2 200
ip route 192.168.100.0 255.255.255.0 10.0.1.2
ip route 192.168.100.0 255.255.255.0 10.0.2.2
```

### Task 15 — Verify Load Balancing
```
show ip route 192.168.100.0
```
Confirm **both** routes now appear as active, equal-cost paths. Generate traffic and observe traffic being distributed across both (per-destination, by default, on most IOS platforms — meaning a single flow to one destination typically takes one consistent path, while different destinations/flows may take either):
```
show ip cef 192.168.100.1
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 4 | Only primary route active; floating backup configured but not installed |
| Task 5 | Backup route installs automatically when the primary interface is shut down; reachability maintained throughout |
| Task 10 | Track object shows Up; primary route active |
| Task 11 | Track object transitions to Down when the *remote* interface fails (not R-HQ's own); floating route installs automatically — demonstrating detection beyond simple interface status |
| Task 13 | Single `/23` summary route replaces two `/24` routes on R-ISP; both branch subnets remain reachable |
| Task 15 | Both paths to 192.168.100.0/24 appear simultaneously as equal-cost routes |

---

## Challenge (Optional)
- Adjust the IP SLA `frequency` value and measure how it affects failover detection time — document the tradeoff between faster failure detection (lower frequency value) and increased probe traffic/overhead.
- Combine object tracking with a **track list** (tracking multiple SLA probes with boolean AND/OR logic) to only fail over when *both* a local and a more distant reachability check fail, reducing false positives from a single flaky probe.
- Using the summarized route from Task 12, add a **third** branch subnet (e.g., 192.168.12.0/24) that does **not** fall within the existing /23 summary boundary, and determine (before configuring) whether the existing summary needs to change, stay the same with an additional specific route alongside it, or be redesigned entirely — implement your chosen approach and verify it.