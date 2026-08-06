# Lab: Dynamic NAT

## Objective
Practice configuring and verifying **Dynamic NAT** on a Cisco IOS router — a pool-based, many-to-many translation where multiple inside hosts share a limited pool of public addresses on a first-come, first-served basis (unlike PAT/overload, each translation consumes one full public address, so the pool can be exhausted).

---

## Topology

```
                         ISP / Internet
                               |
                        Gi0/1  |  198.51.100.1/28
                       +---------------+
                       |      R1       |
                       +---------------+
                        Gi0/0  |  192.168.30.1/24
                               |
                             SW1
                +--------------+--------------+
                |              |               |
           +---------+   +---------+     +---------+
           |  PC1    |   |  PC2    |     |  PC3    |
           |.10      |   |.11      |     |.12      |
           +---------+   +---------+     +---------+

              +-----------+
              |   ISP1    |  (simulates the outside world)
              |198.51.100.2/28|
              +-----------+
```

- **R1** is the NAT router: `Gi0/0` inside, `Gi0/1` outside.
- **PC1, PC2, PC3** are inside hosts competing for a small pool of public addresses.
- **ISP1** simulates the outside network used to test and observe translations.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       | Default Gateway |
|--------|-----------|-----------------|--------------------|-----------------|
| R1     | Gi0/0     | 192.168.30.1     | 255.255.255.0      | —               |
| R1     | Gi0/1     | 198.51.100.1       | 255.255.255.240    | —               |
| ISP1   | Gi0/0     | 198.51.100.2       | 255.255.255.240    | —               |
| PC1    | NIC       | 192.168.30.10       | 255.255.255.0      | 192.168.30.1    |
| PC2    | NIC       | 192.168.30.11       | 255.255.255.0      | 192.168.30.1    |
| PC3    | NIC       | 192.168.30.12       | 255.255.255.0      | 192.168.30.1    |

**Dynamic NAT pool (deliberately small, to observe exhaustion):** `198.51.100.5` – `198.51.100.6` (2 addresses for 3 inside hosts).

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, PC1, PC2, PC3, and ISP1 as shown above.
2. Cable R1 Gi0/0 to SW1, SW1 to PC1/PC2/PC3, and R1 Gi0/1 to ISP1.
3. Apply the addressing from the table above to all devices.
4. Confirm each PC can ping R1's inside interface (192.168.30.1) before continuing.

### Task 2 — Basic Router Configuration
On R1:
- Set the hostname to `R1`.
- Configure `Gi0/0` and `Gi0/1` with the addresses above; `no shutdown` both.
- Add a default route toward the ISP:
  ```
  ip route 0.0.0.0 0.0.0.0 198.51.100.2
  ```
- On ISP1, add a return route to the inside network so ping/traceroute tests work end-to-end:
  ```
  ip route 192.168.30.0 255.255.255.0 198.51.100.1
  ```
  > As in the static NAT lab, a real ISP would never route to private space — this is only so you can verify translation from "outside" in the lab.

### Task 3 — Identify NAT Interfaces
```
interface Gi0/0
 ip nat inside
interface Gi0/1
 ip nat outside
```

### Task 4 — Define Interesting Traffic (ACL)
Create a standard ACL matching the inside hosts eligible for translation:
```
access-list 10 permit 192.168.30.0 0.0.0.255
```

### Task 5 — Define the NAT Pool
Create a pool of just two public addresses — smaller than the number of inside hosts, so you can observe pool exhaustion later:
```
ip nat pool DYNPOOL 198.51.100.5 198.51.100.6 netmask 255.255.255.240
```

### Task 6 — Bind the ACL to the Pool
```
ip nat inside source list 10 pool DYNPOOL
```
> Note: this is **not** overloaded — each translation uses a full address, so only two hosts can be translated at once.

### Task 7 — Generate Traffic and Observe
1. From **PC1**, ping ISP1 (198.51.100.2) — should succeed and consume one pool address.
2. From **PC2**, ping ISP1 — should succeed and consume the second pool address.
3. From **PC3**, ping ISP1 — should **fail** (pool exhausted), until an existing translation times out or is cleared.

### Task 8 — Clear and Retest
On R1, clear the dynamic translation table and retest with only PC1 and PC3 active:
```
clear ip nat translation *
```
Ping from PC3 again — it should now succeed since a pool address is free.

### Task 9 — Verify
```
show ip nat translations
show ip nat statistics
show run | section nat
show access-lists 10
show ip nat pool DYNPOOL     (if supported by your IOS version; otherwise use 'show run | section pool')
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show ip interface brief` on R1 | Both interfaces `up/up`, correct IPs |
| PC1 ping ISP1 | Succeeds; `show ip nat translations` shows PC1 mapped to `.5` or `.6` |
| PC2 ping ISP1 (while PC1's translation is active) | Succeeds; consumes the remaining pool address |
| PC3 ping ISP1 (while both pool addresses are in use) | Fails — no free address in the pool |
| `show ip nat statistics` | Shows `pool: DYNPOOL total addresses 2, allocated 2 (100%)` (or similar) when full |
| `clear ip nat translation *` then retest | Frees the pool; PC3 can now be translated |
| Pings between PC1/PC2/PC3 (inside-to-inside) | Succeed regardless of NAT state (NAT doesn't apply to inside-to-inside traffic) |

---

## Challenge (Optional)
- Enlarge the pool to 3 addresses and confirm all three PCs can be translated simultaneously.
- Add `overload` to the pool-based command (`ip nat inside source list 10 pool DYNPOOL overload`) and observe how the same 2-address pool can now support all three PCs at once via port-level multiplexing — compare `show ip nat translations` output before and after.
- Lower the NAT translation timeout (`ip nat translation timeout 60`) and observe how quickly an idle entry is reclaimed, freeing the pool for a waiting host.