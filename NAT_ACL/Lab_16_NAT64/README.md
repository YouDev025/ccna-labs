# Lab: NAT64

## Objective
Practice configuring and verifying **stateful NAT64** on a Cisco IOS router — translation that allows an **IPv6-only client** to reach an **IPv4-only server**, using a well-known or configured NAT64 prefix to represent IPv4 destinations as IPv6 addresses. This lab also covers **DNS64**, which is required in practice so IPv6-only clients can resolve IPv4-only names into NAT64-mapped addresses automatically.

---

## Topology

```
                 IPv6-only network                    IPv4-only network
             2001:DB8:60::/64                         192.168.50.0/24
                       |                                       |
                     SW-A                                    SW-B
                       |                                       |
              Gi0/0    |                              Gi0/1    |
      2001:DB8:60::1/64|                       192.168.50.1/24 |
                       +---------------+
                       |      R1       |   (NAT64 + DNS64 router)
                       +---------------+
                       |
                  +---------+                         +-----------+
                  |  PC1    |                          |   SRV1    |
                  |(IPv6-only)|                        |(IPv4-only)|
                  |2001:DB8:60::10/64|                 |192.168.50.20/24|
                  +---------+                          +-----------+
```

- **R1** performs stateful NAT64 translation between the IPv6-only segment and the IPv4-only segment, and optionally acts as a **DNS64** resolver.
- **PC1** is an IPv6-only host with no IPv4 stack at all — it can only reach `SRV1` through NAT64.
- **SRV1** is an ordinary IPv4-only server (e.g., running a simple web service) with no awareness that NAT64 exists.

---

## Addressing Table

| Device | Interface | Address                          | Prefix/Mask         | Notes |
|--------|-----------|------------------------------------|-----------------------|-------|
| R1     | Gi0/0     | 2001:DB8:60::1                     | /64                    | IPv6-only side |
| R1     | Gi0/1     | 192.168.50.1                        | 255.255.255.0          | IPv4-only side |
| PC1    | NIC       | 2001:DB8:60::10                     | /64                    | IPv6 only — **no** IPv4 address configured |
| SRV1   | NIC       | 192.168.50.20                       | 255.255.255.0          | IPv4 only |

**NAT64 well-known prefix:** `64:FF9B::/96` (the IANA-reserved prefix used to algorithmically embed an IPv4 address inside an IPv6 address).

With this prefix, `192.168.50.20` is represented to IPv6 clients as:
```
64:FF9B::192.168.50.20
```
(IOS also accepts/display this in pure hex form, e.g. `64:FF9B::C0A8:3214`, which is the same address.)

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW-A, SW-B, PC1, and SRV1 as shown above.
2. Cable R1 Gi0/0 to SW-A → PC1, and R1 Gi0/1 to SW-B → SRV1.
3. Apply the addressing from the table above.
4. On **PC1**, configure **only** an IPv6 address — do not assign any IPv4 address or gateway, so the lab genuinely proves NAT64 (not regular NAT or dual-stack routing) is doing the work.
5. On **SRV1**, configure only an IPv4 address, as it normally would be.
6. Verify baseline reachability: PC1 can ping R1's Gi0/0 (2001:DB8:60::1), and SRV1 can ping R1's Gi0/1 (192.168.50.1). PC1 and SRV1 **cannot** yet reach each other.

### Task 2 — Enable IPv6 Routing and Basic Interfaces
On R1:
```
ipv6 unicast-routing
interface Gi0/0
 ipv6 address 2001:DB8:60::1/64
 no shutdown
interface Gi0/1
 ip address 192.168.50.1 255.255.255.0
 no shutdown
```

### Task 3 — Enable NAT64 Globally and on Interfaces
Mark which interface faces the IPv6-only network and which faces the IPv4-only network:
```
interface Gi0/0
 nat64 enable
interface Gi0/1
 nat64 enable
```

### Task 4 — Configure the NAT64 Prefix
Define the well-known NAT64 prefix R1 will use to represent IPv4 addresses to IPv6 clients:
```
nat64 prefix stateful 64:FF9B::/96
```

### Task 5 — Configure Stateful NAT64 Translation
Enable stateful (dynamic, PAT-like) translation from the IPv6 side to the IPv4 side, using R1's own IPv4 outbound interface as the shared IPv4 address pool (similar in spirit to `overload` in IPv4-to-IPv4 PAT):
```
nat64 v6v4 list NAT64-SOURCE interface GigabitEthernet0/1 overload
```
Where `NAT64-SOURCE` is an IPv6 ACL identifying which IPv6-only hosts are eligible for translation:
```
ipv6 access-list NAT64-SOURCE
 permit ipv6 2001:DB8:60::/64 any
```

### Task 6 — (Optional but Recommended) Configure DNS64
In production, IPv6-only clients rely on **DNS64** to automatically receive a NAT64-mapped AAAA record when they query a name that only has an IPv4 (A) record. Configure R1 as a DNS64-capable resolver, pointing to an upstream IPv4 DNS server reachable via NAT64 or directly if R1 has IPv4 reachability to it:
```
ip dns server
ip name-server 192.168.50.1
ipv6 dns view-list DNS64-VIEW
 view VIEW1
  view-list DNS64-VIEW
  domain-lookup source-interface GigabitEthernet0/1
```
> Full DNS64 configuration varies by IOS train and is often done on a dedicated DNS64 device in real networks; for this lab, the essential concept is: **DNS64 synthesizes AAAA records so client applications never need to know NAT64 exists** — they just resolve a name and connect over IPv6 as normal. You may substitute a static `/etc/hosts`-style entry on PC1 if your platform doesn't support DNS64, so the lab can focus on the translation itself.

### Task 7 — Test Translation Directly (Without DNS64)
From **PC1**, connect directly to SRV1's NAT64-mapped address (bypassing DNS64 to isolate the translation function itself):
```
ping 64:FF9B::192.168.50.20
```
This should succeed — R1 receives the IPv6 packet, recognizes the destination falls within the NAT64 prefix, extracts the embedded IPv4 address (`192.168.50.20`), performs the translation, and forwards an IPv4 packet to SRV1.

### Task 8 — Test with DNS64 (If Configured)
If DNS64 is working, PC1 should be able to simply resolve SRV1's IPv4-only hostname and connect over IPv6 without ever typing the NAT64 prefix manually — confirm this if your DNS64 setup is complete.

### Task 9 — Verify
On R1:
```
show nat64 translations
show nat64 statistics
show nat64 prefix stateful
show run | section nat64
show ipv6 interface brief
show ip interface brief
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show ipv6 interface brief` / `show ip interface brief` on R1 | Gi0/0 shows IPv6 address, Gi0/1 shows IPv4 address, both `up/up` |
| PC1 has no IPv4 address configured | Confirms any successful PC1→SRV1 traffic is genuinely NAT64, not regular dual-stack routing |
| PC1 ping to `64:FF9B::192.168.50.20` | Succeeds |
| `show nat64 translations` | Shows an active mapping: PC1's IPv6 address ↔ R1's IPv4 address (with a port, since `overload` is used), destination = SRV1 |
| `show nat64 statistics` | Translation/packet counters increment as traffic passes |
| SRV1-side capture/`show` (if available) | Traffic from R1 arrives as ordinary IPv4 — SRV1 has no idea NAT64 is involved |
| PC1 ping to SRV1's plain IPv4 address (192.168.50.20) directly | Fails — PC1 has no IPv4 stack; this must go through the NAT64 prefix |

---

## Challenge (Optional)
- Restrict `NAT64-SOURCE` to a single host (PC1's specific /128) instead of the whole /64, and confirm a second IPv6-only host on the same subnet cannot be translated.
- Configure **static NAT64** (`nat64 v6v4 static 2001:DB8:60::10 192.168.50.30`) to give PC1 a fixed, predictable IPv4-mapped identity, and compare its `show nat64 translations` entry (always present) to the dynamic entries created for other hosts (present only while active).
- Research and document, without configuring it in this lab, how **NAT64 differs from 464XLAT and NAT66**, and when a network would choose NAT64 versus simply running dual-stack.