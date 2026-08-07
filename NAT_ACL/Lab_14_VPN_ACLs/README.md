# Lab: VPN ACLs

## Objective
Practice configuring and verifying the **ACLs that support a site-to-site IPsec VPN** on Cisco IOS routers. This lab focuses specifically on the three roles ACLs play in a VPN deployment:
1. A **crypto ACL** ("interesting traffic") that tells the router which traffic should be encrypted and sent through the tunnel.
2. A **NAT exemption ACL** that prevents VPN traffic from being translated by an existing NAT/PAT configuration (so it isn't broken before it reaches the tunnel).
3. A **perimeter ACL** on the outside interface that explicitly permits the IPsec/ISAKMP protocols themselves, so the tunnel can form in the first place.

Full IPsec phase 1/phase 2 configuration (ISAKMP policy, transform sets, crypto maps) is included for context, but the emphasis of this lab is the ACL logic around the VPN, not deep IPsec tuning.

---

## Topology

```
   Site A (HQ)                                          Site B (Branch)
 192.168.10.0/24                                        192.168.20.0/24
        |                                                        |
      SW-A                                                      SW-B
        |                                                        |
  Gi0/0 | 192.168.10.1/24                        192.168.20.1/24 | Gi0/0
   +-----------+                                          +-----------+
   |    R1     |  Gi0/1                          Gi0/1     |    R2     |
   +-----------+  203.0.113.1/30 ---- ISP1 ---- 203.0.113.2/30 +-----------+
        |                                                        |
      PC1                                                      PC2
  192.168.10.10                                           192.168.20.10
```

- **R1** (HQ) and **R2** (Branch) build an IPsec site-to-site VPN across the "Internet" (simulated by ISP1, or a direct back-to-back link between the two `Gi0/1` interfaces if you don't have a spare router for ISP1).
- **PC1** (HQ) and **PC2** (Branch) represent hosts on each private LAN that should communicate over the encrypted tunnel.
- **R1** and **R2** also each have a PAT configuration for normal Internet-bound traffic (e.g., general web browsing) — this is what makes the NAT-exemption ACL necessary.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       | Default Gateway |
|--------|-----------|-------------------|--------------------|-----------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0      | —               |
| R1     | Gi0/1     | 203.0.113.1         | 255.255.255.252    | —               |
| R2     | Gi0/0     | 192.168.20.1       | 255.255.255.0      | —               |
| R2     | Gi0/1     | 203.0.113.2         | 255.255.255.252    | —               |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0      | 192.168.10.1    |
| PC2    | NIC       | 192.168.20.10       | 255.255.255.0      | 192.168.20.1    |

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, R2, SW-A, SW-B, PC1, and PC2 as shown above (insert ISP1 between R1 and R2 if you want a more realistic "Internet" segment, or connect `Gi0/1` on both routers back-to-back for simplicity).
2. Apply the IP addressing from the table above to all devices.
3. Confirm each PC can ping its own gateway, and each router can ping the other router's `Gi0/1` address, **before** configuring VPN or NAT.

### Task 2 — Baseline Routing
Add default/static routes so each router can reach the other's public interface and, later, the tunnel can form:
```
! On R1
ip route 0.0.0.0 0.0.0.0 203.0.113.2   (or via ISP1's address, if used)

! On R2
ip route 0.0.0.0 0.0.0.0 203.0.113.1   (or via ISP1's address, if used)
```

### Task 3 — Configure PAT for General Internet Traffic (Pre-existing Condition)
This simulates each site already having Internet access before the VPN is added — the key setup that makes NAT exemption necessary:
```
! On R1
interface Gi0/0
 ip nat inside
interface Gi0/1
 ip nat outside
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/1 overload

! On R2 (mirror)
interface Gi0/0
 ip nat inside
interface Gi0/1
 ip nat outside
access-list 1 permit 192.168.20.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/1 overload
```

### Task 4 — Build the Crypto ACL (Interesting Traffic)
Define exactly which traffic should be sent through the VPN tunnel: HQ-to-Branch subnet traffic only. This ACL must **mirror** on both routers (source/destination reversed).
```
! On R1
ip access-list extended VPN-INTERESTING-TRAFFIC
 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255

! On R2
ip access-list extended VPN-INTERESTING-TRAFFIC
 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```
> This ACL is referenced later by the crypto map. Only traffic matching this ACL is encrypted; everything else (e.g., general web browsing) continues to use the PAT configuration from Task 3.

### Task 5 — Build the NAT Exemption ACL
Traffic destined for the far-end LAN must **bypass** NAT/PAT, or it will be translated to the public IP before it ever reaches the crypto engine — breaking the tunnel's routing. Create a **separate** ACL for this and reference it in the NAT configuration with a `deny` (meaning "do not translate this") followed by a `permit` for everything else that should still be PATted:
```
! On R1
ip access-list extended NAT-EXEMPT
 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any

! Replace the NAT statement from Task 3 to reference this ACL instead of ACL 1
no ip nat inside source list 1 interface GigabitEthernet0/1 overload
ip nat inside source list NAT-EXEMPT interface GigabitEthernet0/1 overload
```
```
! On R2 (mirror)
ip access-list extended NAT-EXEMPT
 deny   ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any

no ip nat inside source list 1 interface GigabitEthernet0/1 overload
ip nat inside source list NAT-EXEMPT interface GigabitEthernet0/1 overload
```
> The `deny` line here does **not** block traffic — in the context of a NAT ACL, `deny` simply means "don't translate this," while the `permit` line means "do translate everything else." This is a common point of confusion versus firewall ACLs, where `deny` actually blocks traffic.

### Task 6 — Configure IPsec (Phase 1 and Phase 2)
Minimal ISAKMP/IPsec configuration so the crypto ACL has a tunnel to attach to:
```
! On R1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key VpnLabKey123 address 203.0.113.2

crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac

crypto map VPNMAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TSET
 match address VPN-INTERESTING-TRAFFIC

interface Gi0/1
 crypto map VPNMAP
```
```
! On R2 (mirror, peer = 203.0.113.1)
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key VpnLabKey123 address 203.0.113.1

crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac

crypto map VPNMAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TSET
 match address VPN-INTERESTING-TRAFFIC

interface Gi0/1
 crypto map VPNMAP
```

### Task 7 — Build the Perimeter ACL Permitting IPsec/ISAKMP
If the outside interface has (or later gets) an inbound security ACL, it must explicitly permit the protocols the VPN needs, or the tunnel will never form. Build this ACL now and apply it so students see the requirement even in a lab without a pre-existing firewall policy:
```
! On R1 (mirror on R2 with source/destination swapped)
ip access-list extended OUTSIDE-IN
 permit udp host 203.0.113.2 host 203.0.113.1 eq isakmp
 permit udp host 203.0.113.2 host 203.0.113.1 eq non500-isakmp
 permit esp  host 203.0.113.2 host 203.0.113.1
 permit icmp any host 203.0.113.1 echo-reply
 deny   ip any any log

interface Gi0/1
 ip access-group OUTSIDE-IN in
```
> `isakmp` = UDP 500 (tunnel negotiation), `non500-isakmp` = UDP 4500 (NAT-Traversal, used if a NAT device sits between the peers), and `esp` is the encrypted payload protocol itself (IP protocol 50). All three must be permitted for the VPN to establish and pass traffic.

### Task 8 — Verify
From **PC1**, ping **PC2** (192.168.20.10) to trigger interesting traffic and bring the tunnel up.

On **R1** and **R2**:
```
show crypto isakmp sa          (Phase 1 — should show QM_IDLE once established)
show crypto ipsec sa           (Phase 2 — check encaps/decaps counters increasing)
show crypto map
show access-lists VPN-INTERESTING-TRAFFIC
show access-lists NAT-EXEMPT
show access-lists OUTSIDE-IN
show ip nat translations
```

From **PC1**, browse to a simulated external site (or ping ISP1/R2's `Gi0/1` public address, representing "the Internet") to confirm normal PAT still works for non-tunnel traffic.

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show crypto isakmp sa` | Shows `QM_IDLE` state between 203.0.113.1 and 203.0.113.2 after PC1 pings PC2 |
| `show crypto ipsec sa` | `#pkts encaps` / `#pkts decaps` counters increment as PC1 ↔ PC2 traffic flows |
| `show access-lists VPN-INTERESTING-TRAFFIC` | Match counter increments only for HQ ↔ Branch subnet traffic |
| `show ip nat translations` while pinging PC2 | **No** dynamic/overloaded entry created for the 192.168.10.0→192.168.20.0 traffic (correctly exempted) |
| `show ip nat translations` while browsing to an external address | A dynamic/overloaded entry **is** created (PAT still applies to non-VPN traffic) |
| `show access-lists OUTSIDE-IN` | `permit udp ... eq isakmp` and `permit esp` lines show hit counts; the tunnel fails to form if this ACL is missing or misconfigured |
| PC1 ↔ PC2 ping | Succeeds, and traffic is confirmed encrypted via the IPsec SA counters above |

---

## Challenge (Optional)
- Narrow the crypto ACL to only permit a specific host-to-host pair (PC1 ↔ PC2) instead of full subnet-to-subnet, and confirm traffic from a different host on the same subnet no longer triggers the tunnel or gets NAT-exempted.
- Add a third site (Site C) and expand the NAT exemption and crypto ACLs to cover a full hub-and-spoke topology, verifying that Site A's exemption ACL correctly excludes traffic to **both** Site B and Site C while still PATting general Internet traffic.
- Temporarily remove the `non500-isakmp` (UDP 4500) permit line from `OUTSIDE-IN`, simulate a NAT device between the two peers (or note the effect in this lab if no NAT-T is actually in play), and explain in your lab notes why NAT-Traversal support matters when one or both VPN peers sit behind a NAT device.