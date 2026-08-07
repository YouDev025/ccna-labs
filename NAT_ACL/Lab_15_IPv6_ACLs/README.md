# Lab: IPv6 ACLs

## Objective
Practice configuring and verifying **IPv6 access control lists** on a Cisco IOS router. IPv6 ACLs are always "extended" in capability (no numbered standard/extended split like IPv4) and have IPv6-specific behavior — most importantly, they must implicitly permit **Neighbor Discovery Protocol (NDP)** messages, or basic IPv6 operation (address resolution, router discovery) breaks even when the ACL "looks" correct.

---

## Topology

```
                          +-----------+
                          |   SRV1    |   Web/SSH server
                          |2001:DB8:40::20/64|
                          +-----------+
                                |
                              SW-DMZ
                                |
                        Gi0/2   |  2001:DB8:40::1/64
                       +---------------+
                       |      R1       |
                       +---------------+
                Gi0/0  |               |  Gi0/1
   2001:DB8:10::1/64   |               |  2001:DB8:20::1/64
                        |               |
                      SW-A            SW-B
                +-------+-------+       |
                |               |   +---------+
           +---------+     +---------+ |  PC3  |
           |  PC1    |     |  PC2    | |(Admin) |
           |::10     |     |::11     | +---------+
           +---------+     +---------+

  VLAN 10 = Users  (2001:DB8:10::/64) — Gi0/0
  VLAN 20 = Admin  (2001:DB8:20::/64) — Gi0/1
  VLAN 40 = DMZ    (2001:DB8:40::/64) — Gi0/2
```

- **R1** separates three IPv6 subnets: Users, Admin, and DMZ — same logical layout as the Extended ACLs lab, but entirely in IPv6.
- **PC1/PC2** are user workstations; **PC3** is an admin workstation; **SRV1** is a DMZ server offering HTTP/HTTPS and SSH.

---

## IPv6 Addressing Table

| Device | Interface | IPv6 Address              | Prefix Length | Default Gateway   |
|--------|-----------|-----------------------------|----------------|--------------------|
| R1     | Gi0/0     | 2001:DB8:10::1               | /64            | —                  |
| R1     | Gi0/1     | 2001:DB8:20::1               | /64            | —                  |
| R1     | Gi0/2     | 2001:DB8:40::1               | /64            | —                  |
| PC1    | NIC       | 2001:DB8:10::10               | /64            | fe80:: (R1 Gi0/0 link-local, via SLAAC/RA) |
| PC2    | NIC       | 2001:DB8:10::11               | /64            | fe80:: (R1 Gi0/0 link-local) |
| PC3    | NIC       | 2001:DB8:20::10               | /64            | fe80:: (R1 Gi0/1 link-local) |
| SRV1   | NIC       | 2001:DB8:40::20               | /64            | fe80:: (R1 Gi0/2 link-local) |

> Use `ipv6 unicast-routing` on R1, and either static IPv6 addressing on each PC or `ipv6 address autoconfig` with router advertisements enabled on each R1 interface.

---

## Scenario / Requirements
Same policy intent as the IPv4 Extended ACLs lab, translated to IPv6:

1. Users (VLAN 10) may reach SRV1 on **HTTP (80) and HTTPS (443) only** — no SSH, no ping.
2. Admin (VLAN 20) may reach SRV1 on **HTTP, HTTPS, and SSH (22)**, and may **ping6** SRV1 for troubleshooting.
3. Users must **not** be able to initiate traffic directly to the Admin subnet (VLAN 20).
4. **Neighbor Discovery (NDP) must keep working everywhere** — this is the IPv6-specific twist this lab focuses on.

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW-A, SW-B, SW-DMZ, PC1, PC2, PC3, and SRV1 as shown above.
2. Cable each switch to its respective R1 interface.
3. Enable IPv6 routing on R1:
   ```
   ipv6 unicast-routing
   ```
4. Apply the addressing from the table to all interfaces/devices, enabling Router Advertisements (default behavior once an IPv6 address is applied, unless `ipv6 nd ra suppress` is set).
5. Verify baseline connectivity with **no ACLs applied**: every device can ping6 every other device, and each PC has correctly auto-configured or statically set its IPv6 address.

### Task 2 — Understand the NDP Requirement (Read Before Configuring)
IPv6 relies on ICMPv6 for functions IPv4 handled differently (ARP, etc.):
- **Neighbor Solicitation (NS)** / **Neighbor Advertisement (NA)** — equivalent to ARP; resolves link-layer addresses.
- **Router Solicitation (RS)** / **Router Advertisement (RA)** — used for SLAAC and default gateway discovery.

If you write a restrictive IPv6 ACL and forget to permit these, hosts on that link can silently lose the ability to resolve neighbors or learn a default gateway — even though your ACL "looks" like it only blocks the traffic you intended. Cisco IOS **automatically inserts implicit permit entries** for NDP at the end of every IPv6 ACL, but it's best practice (and required on some platforms/older IOS versions) to permit it **explicitly** near the top of the ACL so it isn't affected by anything else you add later.

### Task 3 — Build the Extended IPv6 ACL for Users → DMZ
```
ipv6 access-list USERS-TO-DMZ-V6
 permit icmp any any nd-ns
 permit icmp any any nd-na
 permit tcp 2001:DB8:10::/64 host 2001:DB8:40::20 eq 80
 permit tcp 2001:DB8:10::/64 host 2001:DB8:40::20 eq 443
 deny   ipv6 2001:DB8:10::/64 host 2001:DB8:40::20
 deny   ipv6 2001:DB8:10::/64 2001:DB8:20::/64
 permit ipv6 any any
```
> `nd-ns` / `nd-na` explicitly permit Neighbor Discovery. The two `deny` lines block Users→SRV1 traffic other than HTTP/HTTPS, and Users→Admin traffic entirely. The trailing `permit ipv6 any any` allows everything else (e.g., Internet-bound traffic, if this router had an outside path) to continue.

### Task 4 — Build the Extended IPv6 ACL for Admin → DMZ
```
ipv6 access-list ADMIN-TO-DMZ-V6
 permit icmp any any nd-ns
 permit icmp any any nd-na
 permit tcp 2001:DB8:20::/64 host 2001:DB8:40::20 eq 80
 permit tcp 2001:DB8:20::/64 host 2001:DB8:40::20 eq 443
 permit tcp 2001:DB8:20::/64 host 2001:DB8:40::20 eq 22
 permit icmp 2001:DB8:20::/64 host 2001:DB8:40::20 echo-request
 permit icmp 2001:DB8:20::/64 host 2001:DB8:40::20 echo-reply
 permit ipv6 any any
```

### Task 5 — Apply the ACLs
Apply each ACL **inbound** on the interface where its source traffic enters R1:
```
interface GigabitEthernet0/0
 ipv6 traffic-filter USERS-TO-DMZ-V6 in

interface GigabitEthernet0/1
 ipv6 traffic-filter ADMIN-TO-DMZ-V6 in
```
> Note the IPv6 command is `ipv6 traffic-filter`, not `ip access-group` — a common typo/mixup when working across both address families.

### Task 6 — Verify
From **PC1/PC2** (Users):
- Confirm they can still resolve neighbors and reach their default gateway (`show ipv6 neighbors` on R1, or `ping6` to the R1 link-local/global address) — NDP should be unaffected.
- Browse or connect to SRV1 on TCP 80/443 → should succeed.
- `ping6` SRV1 → should fail.
- SSH to SRV1 (port 22) → should fail.
- Attempt any connection to PC3 (2001:DB8:20::10) → should fail.

From **PC3** (Admin):
- `ping6` SRV1 → should succeed.
- Browse to SRV1 on 80/443 → should succeed.
- SSH to SRV1 → should succeed.

On **R1**:
```
show ipv6 access-list
show ipv6 interface GigabitEthernet0/0
show ipv6 interface GigabitEthernet0/1
show ipv6 neighbors
show ipv6 route
show run | section ipv6 access-list
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show ipv6 interface brief` on R1 | All three interfaces show a link-local and global address, state `up/up` |
| `show ipv6 neighbors` | Populated entries for PC1, PC2, PC3, SRV1 — confirms NDP still functions after the ACL is applied |
| PC1/PC2 → SRV1 HTTP/HTTPS | Succeeds |
| PC1/PC2 → SRV1 ping6 | Fails |
| PC1/PC2 → SRV1 SSH | Fails |
| PC1/PC2 → PC3 (any) | Fails |
| PC3 → SRV1 HTTP/HTTPS/SSH/ping6 | All succeed |
| `show ipv6 access-list USERS-TO-DMZ-V6` | Match counters increment on the `permit tcp ... eq 80/443` and `deny ipv6 ...` lines as traffic is tested; `nd-ns`/`nd-na` lines increment continuously in the background |
| `show ipv6 access-list ADMIN-TO-DMZ-V6` | Match counters increment on the service-specific permits as PC3 traffic is tested |

---

## Challenge (Optional)
- Remove the explicit `permit icmp any any nd-ns` / `nd-na` lines and observe whether IOS's **implicit** NDP permit still keeps neighbor discovery working on your platform/IOS version — document the difference in behavior (or lack thereof) in your lab notes.
- Add an ACL entry using an **object-like approach**: reference a specific host via `host 2001:DB8:10::10` instead of the whole /64, and confirm only PC1 (not PC2) is affected by a new, more granular rule.
- Extend the lab to dual-stack: apply an equivalent IPv4 extended ACL (as in the Extended ACLs lab) on the same interfaces alongside these IPv6 ACLs, and verify that IPv4 and IPv6 traffic filtering operate independently — an interface can have both `ip access-group` and `ipv6 traffic-filter` applied at the same time.