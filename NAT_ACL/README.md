# Lab: Dynamic PAT

## Objective
Practice configuring and verifying **Dynamic PAT** (Port Address Translation) — many inside hosts sharing a **small pool** of public addresses, distinguished by port number rather than a single interface address. This sits conceptually between the two NAT labs you may have already done: **Dynamic NAT** (one-to-one from a pool, easily exhausted) and **Static NAT + Overload** (all hosts sharing one interface address). Here, a *pool* of addresses is overloaded, giving more headroom than a single address while still using far fewer public IPs than inside hosts.

---

## Topology

```
                         ISP / Internet
                               |
                        Gi0/1  |  198.51.100.17/28
                       +---------------+
                       |      R1       |
                       +---------------+
                        Gi0/0  |  192.168.50.1/24
                               |
                             SW1
                +--------------+--------------+--------------+
                |              |               |              |
           +---------+   +---------+     +---------+    +---------+
           |  PC1    |   |  PC2    |     |  PC3    |    |  PC4    |
           |.10      |   |.11      |     |.12      |    |.13      |
           +---------+   +---------+     +---------+    +---------+

              +-----------+
              |   ISP1    |  (simulates the outside world)
              |198.51.100.18/28|
              +-----------+
```

- **R1** is the NAT router: `Gi0/0` inside, `Gi0/1` outside.
- **PC1–PC4** are four inside hosts, all sharing a **pool of two public addresses** via PAT.
- **ISP1** simulates the outside network, used to generate and verify translated traffic.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       | Default Gateway |
|--------|-----------|-----------------|---------------------|-----------------|
| R1     | Gi0/0     | 192.168.50.1     | 255.255.255.0        | —               |
| R1     | Gi0/1     | 198.51.100.17     | 255.255.255.240      | —               |
| ISP1   | Gi0/0     | 198.51.100.18     | 255.255.255.240      | —               |
| PC1    | NIC       | 192.168.50.10     | 255.255.255.0        | 192.168.50.1    |
| PC2    | NIC       | 192.168.50.11     | 255.255.255.0        | 192.168.50.1    |
| PC3    | NIC       | 192.168.50.12     | 255.255.255.0        | 192.168.50.1    |
| PC4    | NIC       | 192.168.50.13     | 255.255.255.0        | 192.168.50.1    |

**PAT pool:** `198.51.100.20` – `198.51.100.21` (2 addresses shared by 4 inside hosts, via port-level multiplexing).

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, PC1–PC4, and ISP1 as shown.
2. Cable and address all devices per the table above.
3. Confirm each PC can ping R1's inside interface before continuing.

### Task 2 — Basic Router Configuration
```
interface Gi0/0
 ip address 192.168.50.1 255.255.255.0
 no shutdown
interface Gi0/1
 ip address 198.51.100.17 255.255.255.240
 no shutdown
ip route 0.0.0.0 0.0.0.0 198.51.100.18
```
On ISP1, add a return route for testing:
```
ip route 192.168.50.0 255.255.255.0 198.51.100.17
```

### Task 3 — Identify NAT Interfaces
```
interface Gi0/0
 ip nat inside
interface Gi0/1
 ip nat outside
```

### Task 4 — Define Interesting Traffic
```
access-list 30 permit 192.168.50.0 0.0.0.255
```

### Task 5 — Define the PAT Pool
```
ip nat pool PATPOOL 198.51.100.20 198.51.100.21 netmask 255.255.255.240
```

### Task 6 — Enable Dynamic PAT (Pool + Overload)
This is the key command that distinguishes this lab from the Dynamic NAT lab — adding `overload` to a pool-based NAT statement enables **many-to-few** translation instead of one-to-one:
```
ip nat inside source list 30 pool PATPOOL overload
```

### Task 7 — Generate Traffic from All Four Hosts Simultaneously
From **PC1, PC2, PC3, and PC4**, ping (or open multiple simultaneous sessions to) ISP1 (198.51.100.18) at the same time.

Unlike the Dynamic NAT lab (no overload), all four should succeed **simultaneously**, even though there are only 2 public addresses — because each is now distinguished by source port, not by address alone.

### Task 8 — Verify
```
show ip nat translations
show ip nat statistics
show run | section nat
```

Observe in `show ip nat translations` that multiple inside hosts now map to the **same** pool address, differentiated only by the port number in the translated entry — this is the core concept of PAT.

### Task 9 — Compare Against a Single-Address Equivalent
For contrast, temporarily replace the pool-based PAT with single-address PAT using just the outside interface (as in the Static NAT + Overload lab):
```
no ip nat inside source list 30 pool PATPOOL overload
ip nat inside source list 30 interface GigabitEthernet0/1 overload
```
Retest all four PCs simultaneously — they should still all succeed, now sharing just the **one** interface address instead of a two-address pool. In your lab notes, explain why a pool is still sometimes preferred over a single interface address even with overload enabled (e.g., spreading load, matching an ISP-assigned range larger than one address, or avoiding a single point of translation exhaustion under very high connection counts).

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show ip interface brief` on R1 | Both interfaces `up/up`, correct IPs |
| PC1–PC4 ping ISP1 simultaneously (pool + overload) | All succeed at once, despite only 2 pool addresses |
| `show ip nat translations` | Multiple hosts mapped to the same pool address, unique ports per entry |
| `show ip nat statistics` | Pool shows addresses allocated, with overload multiplexing multiple translations per address |
| Switching to interface-based PAT (Task 9) | All four hosts still succeed, now sharing the single outside interface address |

---

## Challenge (Optional)
- Reduce the PAT pool to a single address and confirm behavior is now identical to interface-based overload — demonstrating that a one-address pool with `overload` and interface-based `overload` are functionally equivalent.
- Generate enough simultaneous connections from all four hosts to approach the practical port-exhaustion limit for a single pool address (many thousands of ports per address in theory, but worth discussing as a theoretical scaling limit), and document why real ISPs sometimes provide multiple public addresses specifically to support very high connection-count environments (e.g., large NAT gateways, CGNAT deployments).