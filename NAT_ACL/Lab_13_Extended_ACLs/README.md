# Lab: Extended ACLs

## Objective
Practice configuring and verifying **extended access control lists** on a Cisco IOS router. Unlike standard ACLs (source only), extended ACLs filter on source address, destination address, protocol, and port — allowing precise, service-specific traffic control.

---

## Topology

```
                          +-----------+
                          |   SRV1    |   Web server (HTTP/HTTPS)
                          |192.168.40.20/24|
                          +-----------+
                                |
                              SW-DMZ
                                |
                        Gi0/2   |  192.168.40.1/24
                       +---------------+
                       |      R1       |
                       +---------------+
                Gi0/0  |               |  Gi0/1
      192.168.10.1/24  |               |  192.168.20.1/24
                        |               |
                      SW-A            SW-B
                +-------+-------+       |
                |               |   +---------+
           +---------+     +---------+ |  PC3  |
           |  PC1    |     |  PC2    | |(Server |
           | .10     |     | .11     | | subnet)|
           +---------+     +---------+ +---------+

  VLAN 10 (192.168.10.0/24) = Users     — Gi0/0
  VLAN 20 (192.168.20.0/24) = Admin/IT  — Gi0/1
  VLAN 40 (192.168.40.0/24) = DMZ/Server— Gi0/2
```

- **R1** is a router-on-a-stick / multi-interface router separating three subnets: **Users**, **Admin**, and **DMZ**.
- **PC1** and **PC2** are regular user workstations (Users subnet).
- **PC3** represents an IT/admin workstation (Admin subnet) that needs broader access.
- **SRV1** is a DMZ web server offering HTTP (80) and HTTPS (443), and also SSH (22) for admin management only.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       | Default Gateway |
|--------|-----------|-------------------|--------------------|-----------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0      | —               |
| R1     | Gi0/1     | 192.168.20.1       | 255.255.255.0      | —               |
| R1     | Gi0/2     | 192.168.40.1       | 255.255.255.0      | —               |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0      | 192.168.10.1    |
| PC2    | NIC       | 192.168.10.11       | 255.255.255.0      | 192.168.10.1    |
| PC3    | NIC       | 192.168.20.10       | 255.255.255.0      | 192.168.20.1    |
| SRV1   | NIC       | 192.168.40.20       | 255.255.255.0      | 192.168.40.1    |

---

## Scenario / Requirements
The security policy for R1 states:

1. Users (VLAN 10) may reach SRV1 on **HTTP (80) and HTTPS (443) only** — no SSH, no ping.
2. Admin (VLAN 20) may reach SRV1 on **HTTP, HTTPS, and SSH (22)**, and may also **ping** SRV1 for troubleshooting.
3. Users must **not** be able to initiate any traffic directly to the Admin subnet (VLAN 20).
4. All other traffic not explicitly addressed above should be **permitted** (this lab focuses on the DMZ and inter-VLAN restriction only).

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW-A, SW-B, SW-DMZ, PC1, PC2, PC3, and SRV1 as shown above.
2. Cable each switch to its respective router interface (or subinterface if using router-on-a-stick with trunking — either physical or VLAN-based design is acceptable).
3. Apply the IP addressing from the table above to all devices.
4. Verify baseline connectivity with **no ACLs applied**: every device can ping every other device.

### Task 2 — Basic Router Configuration
On R1:
- Set hostname `R1`.
- Configure `Gi0/0`, `Gi0/1`, and `Gi0/2` with the addresses above; `no shutdown` all three.
- No dynamic routing is required — all subnets are directly connected to R1.

### Task 3 — Build the Extended ACL for User → DMZ Traffic
Create a named extended ACL that permits only HTTP/HTTPS from Users to SRV1, and denies everything else from Users toward the DMZ:
```
ip access-list extended USERS-TO-DMZ
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 80
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 443
 deny   ip 192.168.10.0 0.0.0.255 host 192.168.40.20
 permit ip any any
```
> The final `deny` line blocks anything else (e.g., ping, SSH) from Users to SRV1 specifically, while the trailing `permit ip any any` allows unrelated traffic to keep flowing.

### Task 4 — Build the Extended ACL for Admin → DMZ Traffic
Create a second ACL permitting Admin full service access plus ICMP to SRV1:
```
ip access-list extended ADMIN-TO-DMZ
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 80
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 443
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 22
 permit icmp 192.168.20.0 0.0.0.255 host 192.168.40.20
 permit ip any any
```

### Task 5 — Block Users from Reaching the Admin Subnet
Add a rule (either as its own ACL applied on Gi0/0, or merged into `USERS-TO-DMZ` if applied on the same interface/direction) that denies Users-to-Admin traffic:
```
ip access-list extended USERS-TO-DMZ
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
```
> Insert this **above** the trailing `permit ip any any` line using `sequence numbers` if your IOS requires it (e.g., `show access-lists USERS-TO-DMZ` to see line numbers, then `ip access-list extended USERS-TO-DMZ` followed by `15 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255`).

### Task 6 — Apply the ACLs
Apply each ACL **inbound** on the interface where the source traffic enters R1:
```
interface GigabitEthernet0/0
 ip access-group USERS-TO-DMZ in

interface GigabitEthernet0/1
 ip access-group ADMIN-TO-DMZ in
```

### Task 7 — Verify
From **PC1** or **PC2** (Users):
- Browse or `curl`/telnet to SRV1 on port 80 and 443 → should succeed.
- Ping SRV1 → should fail (ICMP not permitted, hits the deny).
- Attempt SSH to SRV1 (port 22) → should fail.
- Ping or attempt any connection to PC3 (192.168.20.10) → should fail.

From **PC3** (Admin):
- Ping SRV1 → should succeed.
- Browse to SRV1 on 80/443 → should succeed.
- SSH to SRV1 → should succeed.

On R1:
```
show access-lists
show ip interface GigabitEthernet0/0 | include access list
show ip interface GigabitEthernet0/1 | include access list
show run | section access-list
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show ip interface brief` on R1 | All three interfaces `up/up`, correct IPs |
| PC1/PC2 → SRV1 HTTP (80) | Succeeds |
| PC1/PC2 → SRV1 HTTPS (443) | Succeeds |
| PC1/PC2 → SRV1 ping (ICMP) | Fails |
| PC1/PC2 → SRV1 SSH (22) | Fails |
| PC1/PC2 → PC3 (any traffic) | Fails |
| PC3 → SRV1 HTTP/HTTPS/SSH/ping | All succeed |
| `show access-lists USERS-TO-DMZ` | Match counters increment on the `permit tcp ... eq 80/443`, `deny ip ... 192.168.40.20`, and `deny ip ... 192.168.20.0` lines as traffic is tested |
| `show access-lists ADMIN-TO-DMZ` | Match counters increment on the service-specific `permit` lines as PC3 traffic is tested |

---

## Challenge (Optional)
- Add a rule so SRV1 itself can **initiate** DNS queries (UDP/53) outbound to an external resolver, while all other outbound traffic from SRV1 remains blocked by a new ACL applied on `Gi0/2 in`.
- Use `established` (or reflexive ACLs) so that **return traffic** from SRV1 to Users on ports 80/443 is explicitly permitted, and observe how removing `established` breaks nothing here (since ACLs on Cisco IOS are stateless by default for basic extended ACLs) versus what changes if you experiment with reflexive ACLs.
- Convert the plain extended ACLs into **named ACLs with remarks** (`remark`) documenting the purpose of each line, and use `show access-lists` to confirm the remarks appear alongside each rule.