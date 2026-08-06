# Lab: Static NAT + NAT Overload (PAT)

## Objective
Practice configuring and verifying two forms of Network Address Translation on a Cisco IOS router:
- **Static NAT** — a permanent one-to-one mapping so an inside server is reachable from the outside world on a fixed public address.
- **NAT Overload (PAT)** — many inside hosts sharing a single public IP address, distinguished by port number, for general outbound Internet access.

You will also configure an **extended ACL** to control which inside traffic is eligible for translation, and verify everything with `show` commands and end-to-end connectivity tests.

---

## Topology

```
                         ISP / Internet
                               |
                        Gi0/1  |  209.165.201.1/29
                       +---------------+
                       |      R1       |   (border/NAT router)
                       +---------------+
                        Gi0/0  |  192.168.10.1/24
                               |
                    +----------+-----------+
                    |                      |
              +-----------+          +-----------+
              |   SW1     |          |  SRV1     |
              +-----------+          |192.168.10.10/24|
                    |                +-----------+
         +----------+----------+
         |                     |
    +---------+           +---------+
    |  PC1    |           |  PC2    |
    |DHCP/    |           |DHCP/    |
    |Static   |           |Static   |
    +---------+           +---------+

              +-----------+
              |   ISP1    |  (simulates the outside world)
              |209.165.201.2/29|
              +-----------+
```

- **R1** is the NAT router: `Gi0/0` faces the inside LAN, `Gi0/1` faces the ISP.
- **SW1** is a Layer 2 switch connecting PC1, PC2, and SRV1 to R1.
- **SRV1** is an internal server (e.g., web/FTP) that must be reachable from outside.
- **ISP1** is a router simulating the Internet, used only to test reachability/translation from "outside."

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       | Default Gateway |
|--------|-----------|-------------------|--------------------|-----------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0      | —               |
| R1     | Gi0/1     | 209.165.201.1       | 255.255.255.248    | —               |
| ISP1   | Gi0/0     | 209.165.201.2       | 255.255.255.248    | —               |
| SRV1   | NIC       | 192.168.10.10       | 255.255.255.0      | 192.168.10.1    |
| PC1    | NIC       | 192.168.10.11       | 255.255.255.0      | 192.168.10.1    |
| PC2    | NIC       | 192.168.10.12       | 255.255.255.0      | 192.168.10.1    |

**Public address pool for overload (PAT):** use the router's outside interface address `209.165.201.1` (recommended for `ip nat inside source ... overload`).

**Static NAT mapping:** `192.168.10.10` (SRV1 inside) ↔ `209.165.201.10` (public address for the server).

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, ISP1, SW1, SRV1, PC1, and PC2 as shown above.
2. Cable:
   - R1 Gi0/0 → SW1
   - SW1 → PC1, PC2, SRV1
   - R1 Gi0/1 → ISP1 Gi0/0
3. Apply the IP addressing from the table above to all devices.
4. Confirm each device can ping its **default gateway** before continuing.

### Task 2 — Basic Router Configuration
On R1:
- Set hostname `R1`.
- Configure `Gi0/0` and `Gi0/1` with the addresses above and `no shutdown` both interfaces.
- Add a default route pointing to the ISP so outbound traffic has somewhere to go:
  ```
  ip route 0.0.0.0 0.0.0.0 209.165.201.2
  ```
- On ISP1, add a static (or summary) route back to the inside network so return traffic for the static NAT test works:
  ```
  ip route 192.168.10.0 255.255.255.0 209.165.201.1
  ```
  > In real life the ISP wouldn't route to a private network — here it's only so you can test the static NAT mapping from "outside."

### Task 3 — Identify NAT Interfaces
On R1, mark which interface is inside and which is outside:
```
interface Gi0/0
 ip nat inside
interface Gi0/1
 ip nat outside
```

### Task 4 — Configure Static NAT for SRV1
Create a permanent, one-to-one translation so SRV1 is reachable from outside as `209.165.201.10`:
```
ip nat inside source static 192.168.10.10 209.165.201.10
```

### Task 5 — Define Interesting Traffic for Overload (ACL)
Create a standard ACL that identifies which inside hosts are allowed to be translated via PAT. Exclude the server, since it already has its own static mapping:
```
access-list 1 deny   host 192.168.10.10
access-list 1 permit 192.168.10.0 0.0.0.255
```

### Task 6 — Configure NAT Overload (PAT)
Enable dynamic NAT with overload, using the outside interface's address as the shared public IP:
```
ip nat inside source list 1 interface GigabitEthernet0/1 overload
```

### Task 7 — Verify
From PC1 and PC2:
- Ping ISP1's address (209.165.201.2) to confirm PAT is translating outbound traffic.
- Ping SRV1 (192.168.10.10) to confirm inside-to-inside connectivity is unaffected by NAT.

From ISP1:
- Ping `209.165.201.10` to confirm the static NAT mapping delivers traffic to SRV1.

On R1, use these commands to confirm behavior:
```
show ip nat translations
show ip nat statistics
show run | section nat
show ip interface brief
show access-lists
debug ip nat        (optional, generate traffic then 'undebug all')
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show ip interface brief` on R1 | Both interfaces `up/up`, correct IPs |
| PC1/PC2 ping ISP1 (209.165.201.2) | Success; `show ip nat translations` shows a dynamic entry with an overloaded port for each PC |
| ISP1 ping 209.165.201.10 | Success; reaches SRV1 via the static entry |
| `show ip nat translations` | One **static** entry for SRV1 (always present) + **dynamic** entries for PC1/PC2 only while traffic is active |
| `show ip nat statistics` | Shows hits, active translations, and the correct inside/outside interfaces |
| SRV1 traffic | Never appears as a dynamic/overloaded entry (excluded by ACL 1) |

---

## Challenge (Optional)
- Add a second internal server and give it its own static NAT entry on a different public IP.
- Restrict PAT to only PC1 (deny PC2 in the ACL) and verify PC2 loses outside connectivity while PC1 keeps it.
- Convert the static NAT entry into a **static NAT with port forwarding** (e.g., only forward TCP/80) using:
  ```
  ip nat inside source static tcp 192.168.10.10 80 209.165.201.10 80
  ```