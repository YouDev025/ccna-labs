# Lab 04: VPN Configuration

## Objective

This lab demonstrates how to design and configure a Site-to-Site VPN topology connecting two remote offices. Each site contains two separate LANs protected by a VPN gateway. The lab focuses on IP addressing, static routing, VPN concepts, and verification of connectivity.

> **Note:** Cisco Packet Tracer has limited support for IPsec VPNs. The IPsec configuration included in this lab is representative of a real Cisco IOS router and may require GNS3, EVE-NG, or physical Cisco devices for full functionality.

---

# Network Topology

```
                    Branch Office                          Headquarters

      PC0 -------- Switch0                    Switch1 -------- PC1
                     |                              |
                     |                              |
                G0/0/0                         G0/0/0
                 Router2========================Router1
                S0/1/0         WAN            S0/1/0
                     |                              |
                     |                              |
                G0/0/1                         G0/0/1
                     |                              |
      PC2 -------- Switch2                    Switch3 -------- PC3
```

---

# Devices Used

- 2 Cisco 2911 Routers
- 4 Cisco 2960 Switches
- 4 PCs
- 1 Serial DCE Cable
- Copper Straight-Through Cables

---

# IP Addressing

## Branch Office

| Device | Interface | IP Address | Mask |
|---------|-----------|------------|------|
| Router2 | G0/0/0 | 192.168.10.1 | 255.255.255.0 |
| Router2 | G0/0/1 | 192.168.11.1 | 255.255.255.0 |
| Router2 | S0/1/0 | 203.0.113.1 | 255.255.255.252 |
| PC0 | Fa0 | 192.168.10.10 | /24 |
| PC2 | Fa0 | 192.168.11.10 | /24 |

---

## Headquarters

| Device | Interface | IP Address | Mask |
|---------|-----------|------------|------|
| Router1 | G0/0/0 | 192.168.20.1 | 255.255.255.0 |
| Router1 | G0/0/1 | 192.168.21.1 | 255.255.255.0 |
| Router1 | S0/1/0 | 203.0.113.2 | 255.255.255.252 |
| PC1 | Fa0 | 192.168.20.10 | /24 |
| PC3 | Fa0 | 192.168.21.10 | /24 |

---

# Default Gateways

| PC | Default Gateway |
|----|-----------------|
| PC0 | 192.168.10.1 |
| PC2 | 192.168.11.1 |
| PC1 | 192.168.20.1 |
| PC3 | 192.168.21.1 |

---

# Lab Tasks

- Build the VPN topology.
- Configure router interfaces.
- Configure IP addressing.
- Configure static routing.
- Configure Site-to-Site IPsec VPN.
- Verify connectivity.
- Verify VPN operation.

---

# Router2 Configuration (Branch Office)

```cisco
enable
configure terminal

hostname Branch

interface GigabitEthernet0/0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

interface GigabitEthernet0/0/1
 ip address 192.168.11.1 255.255.255.0
 no shutdown
exit

interface Serial0/1/0
 ip address 203.0.113.1 255.255.255.252
 clock rate 64000
 no shutdown
exit

ip route 192.168.20.0 255.255.255.0 203.0.113.2
ip route 192.168.21.0 255.255.255.0 203.0.113.2

end
write memory
```

> Configure the `clock rate` only if Router2 is the DCE side.

---

# Router1 Configuration (Headquarters)

```cisco
enable
configure terminal

hostname Headquarters

interface GigabitEthernet0/0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit

interface GigabitEthernet0/0/1
 ip address 192.168.21.1 255.255.255.0
 no shutdown
exit

interface Serial0/1/0
 ip address 203.0.113.2 255.255.255.252
 no shutdown
exit

ip route 192.168.10.0 255.255.255.0 203.0.113.1
ip route 192.168.11.0 255.255.255.0 203.0.113.1

end
write memory
```

---

# IPsec VPN Configuration (Branch)

```cisco
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key Cisco123 address 203.0.113.2

crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac

access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.21.0 0.0.0.255
access-list 110 permit ip 192.168.11.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit ip 192.168.11.0 0.0.0.255 192.168.21.0 0.0.0.255

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set VPN-SET
 match address 110

interface Serial0/1/0
 crypto map VPN-MAP
```

---

# IPsec VPN Configuration (Headquarters)

```cisco
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key Cisco123 address 203.0.113.1

crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac

access-list 110 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip 192.168.20.0 0.0.0.255 192.168.11.0 0.0.0.255
access-list 110 permit ip 192.168.21.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip 192.168.21.0 0.0.0.255 192.168.11.0 0.0.0.255

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set VPN-SET
 match address 110

interface Serial0/1/0
 crypto map VPN-MAP
```

---

# Verification Commands

## Verify Interface Status

```cisco
show ip interface brief
```

## Verify Routing Table

```cisco
show ip route
```

## Verify IKE Phase 1

```cisco
show crypto isakmp sa
```

## Verify IPsec Security Associations

```cisco
show crypto ipsec sa
```

## Verify Active VPN Sessions

```cisco
show crypto session
```

## Verify Crypto Map

```cisco
show crypto map
```

---

# Connectivity Tests

From **PC0**

```text
ping 192.168.20.10
ping 192.168.21.10
```

From **PC2**

```text
ping 192.168.20.10
ping 192.168.21.10
```

From **PC1**

```text
ping 192.168.10.10
ping 192.168.11.10
```

From **PC3**

```text
ping 192.168.10.10
ping 192.168.11.10
```

---

# Expected Results

- Router interfaces are operational (`up/up`).
- Static routes enable communication between all four LANs.
- The VPN tunnel establishes when interesting traffic is generated.
- Traffic between Branch and Headquarters is encrypted.
- Hosts on both sites can communicate securely.
- `show crypto isakmp sa` displays an active security association.
- `show crypto ipsec sa` shows packet encryption and decryption counters increasing.
- `show crypto session` reports the VPN session as active.

---

# Concepts Covered

- Site-to-Site VPN
- IPsec
- ISAKMP / IKE
- Pre-Shared Key Authentication
- Transform Sets
- Crypto ACLs
- Crypto Maps
- Static Routing
- Multiple Protected Subnets
- VPN Verification and Troubleshooting

---

# Skills Practiced

- Design a multi-site VPN topology.
- Configure router interfaces and IP addressing.
- Configure static routes.
- Configure a Site-to-Site IPsec VPN.
- Protect multiple LANs across a single VPN tunnel.
- Verify VPN status using Cisco IOS commands.
- Test secure end-to-end connectivity.

---

# Conclusion

After completing this lab, you will understand how to deploy a Site-to-Site IPsec VPN between two remote offices with multiple protected subnets. You will be able to configure routing, apply VPN policies, verify tunnel establishment, and troubleshoot secure communication using Cisco IOS verification commands.