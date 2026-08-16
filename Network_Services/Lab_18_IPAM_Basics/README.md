# Lab IPAM Basics

## 1. Objective

This lab introduces the basic concepts of **IP Address Management (IPAM)** in an enterprise network.

IPAM helps administrators organize, assign, track, and manage IP addresses and network information.

At the end of this lab, you should be able to:

* Build a basic enterprise IPAM topology.
* Create an IP addressing plan.
* Configure IPv4 addresses on network devices.
* Configure DHCP for automatic IP address assignment.
* Reserve addresses for network infrastructure.
* Verify DHCP leases.
* Test connectivity between devices.
* Identify used and available IP addresses.
* Understand the relationship between IPAM, DHCP, and DNS.
* Troubleshoot common IP addressing problems.

---

# 2. What Is IPAM?

**IPAM** means:

> Internet Protocol Address Management

IPAM is the process of managing IP addresses in a network.

It helps administrators answer questions such as:

```text
Which IP addresses are available?
Which IP address is assigned to a PC?
Which addresses are reserved?
Which subnet belongs to which department?
Which device owns a particular IP address?
```

A simple IPAM structure can be:

```text
Network
   │
   ├── Subnet
   │      │
   │      ├── Used IPs
   │      ├── Reserved IPs
   │      └── Available IPs
   │
   └── DHCP
          │
          └── IP Leases
```

---

# 3. Network Topology

The lab uses a small enterprise network:

```text
                         ┌─────────────────┐
                         │      R1         │
                         │    Router       │
                         │ 192.168.10.1    │
                         │      DHCP       │
                         └────────┬────────┘
                                  │
                                  │
                           ┌──────┴──────┐
                           │     SW1     │
                           │   Switch    │
                           └──────┬──────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    │             │             │
               ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
               │   PC1   │   │   PC2   │   │ Server  │
               │ DHCP    │   │ DHCP    │   │ Static  │
               └─────────┘   └─────────┘   └─────────┘
```

---

# 4. Devices

| Device     | Quantity | Role                       |
| ---------- | -------: | -------------------------- |
| Router R1  |        1 | Gateway + DHCP             |
| Switch SW1 |        1 | Layer 2 connectivity       |
| PC1        |        1 | DHCP client                |
| PC2        |        1 | DHCP client                |
| Server     |        1 | Static IP / Infrastructure |

---

# 5. IP Addressing Plan

Network:

```text
192.168.10.0/24
```

Subnet mask:

```text
255.255.255.0
```

Usable range:

```text
192.168.10.1 → 192.168.10.254
```

Broadcast:

```text
192.168.10.255
```

### IPAM Table

| IP Address        | Device         | Type           | Status    |
| ----------------- | -------------- | -------------- | --------- |
| 192.168.10.1      | R1             | Gateway        | Reserved  |
| 192.168.10.2      | SW1            | Management     | Reserved  |
| 192.168.10.10     | Server         | Infrastructure | Reserved  |
| 192.168.10.11–20  | Infrastructure | Reserved       | Reserved  |
| 192.168.10.21–254 | Clients        | DHCP           | Available |
| 192.168.10.255    | Broadcast      | Broadcast      | Reserved  |

---

# 6. IP Address Categories

In this lab, addresses are divided into three categories.

### Infrastructure

```text
192.168.10.1
192.168.10.2
192.168.10.10
```

### DHCP range

```text
192.168.10.21
        ↓
192.168.10.254
```

### Broadcast

```text
192.168.10.255
```

The gateway and infrastructure addresses should not be assigned dynamically to clients.

---

# 7. Step 1 — Configure R1

Enter configuration mode:

```cisco
enable
configure terminal
```

Configure the router interface:

```cisco
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit
```

Verify:

```cisco
show ip interface brief
```

Expected:

```text
Interface              IP-Address      Status
GigabitEthernet0/0     192.168.10.1    up
```

---

# 8. Step 2 — Reserve Infrastructure Addresses

We do not want DHCP to assign infrastructure addresses.

Reserve:

```text
192.168.10.1 → Router
192.168.10.2 → Switch
192.168.10.10 → Server
192.168.10.11–20 → Future infrastructure
```

On R1:

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.20
```

This prevents DHCP from assigning these addresses.

---

# 9. Step 3 — Configure DHCP

Create a DHCP pool:

```cisco
ip dhcp pool LAN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.10.10
exit
```

The DHCP server will now provide:

```text
IP address
Subnet mask
Default gateway
DNS server
```

to DHCP clients.

---

# 10. Step 4 — Configure SW1

Configure the management interface:

```cisco
enable
configure terminal

interface vlan 1
 ip address 192.168.10.2 255.255.255.0
 no shutdown
exit
```

Configure the default gateway:

```cisco
ip default-gateway 192.168.10.1
```

Save:

```cisco
end
write memory
```

---

# 11. Step 5 — Configure the Server

Configure the server manually.

```text
IP Address:       192.168.10.10
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
DNS Server:       192.168.10.10
```

This server represents an infrastructure resource managed by the IPAM plan.

---

# 12. Step 6 — Configure PC1

Set PC1 to DHCP.

In Packet Tracer:

```text
PC1
→ Desktop
→ IP Configuration
→ DHCP
```

PC1 should receive an address such as:

```text
IP Address:       192.168.10.21
Subnet Mask:      255.255.255.0
Gateway:          192.168.10.1
DNS:              192.168.10.10
```

The exact DHCP address may be different depending on the order of requests.

---

# 13. Step 7 — Configure PC2

Set PC2 to DHCP:

```text
PC2
→ Desktop
→ IP Configuration
→ DHCP
```

It should receive another available address.

For example:

```text
IP Address:       192.168.10.22
Subnet Mask:      255.255.255.0
Gateway:          192.168.10.1
DNS:              192.168.10.10
```

---

# 14. Step 8 — Verify DHCP

On R1:

```cisco
show ip dhcp binding
```

Example:

```text
Bindings from all pools:

IP address       Client-ID
192.168.10.21    ...
192.168.10.22    ...
```

This command shows which IP addresses are currently assigned.

---

# 15. Step 9 — Verify DHCP Pool

Use:

```cisco
show ip dhcp pool
```

You should see information about:

```text
Pool name
Network
Default router
Allocated addresses
Excluded addresses
```

Example:

```text
Pool LAN
Network: 192.168.10.0/24
Default router: 192.168.10.1
```

---

# 16. Step 10 — Verify Excluded Addresses

Use:

```cisco
show running-config | section dhcp
```

You should see:

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.20
```

This confirms that infrastructure addresses are protected from DHCP assignment.

---

# 17. Step 11 — Test Connectivity

From PC1:

```bash
ping 192.168.10.1
```

Expected:

```text
Reply from 192.168.10.1
```

Test the switch:

```bash
ping 192.168.10.2
```

Test the server:

```bash
ping 192.168.10.10
```

Test PC2:

```bash
ping 192.168.10.22
```

The exact address of PC2 depends on the DHCP lease.

---

# 18. Step 12 — Verify the Default Gateway

On PC1:

```text
ipconfig
```

Verify:

```text
Default Gateway: 192.168.10.1
```

The default gateway allows the PC to communicate with networks outside its local subnet.

---

# 19. Step 13 — Verify the Subnet

The subnet is:

```text
192.168.10.0/24
```

Important addresses:

```text
Network:        192.168.10.0
First host:     192.168.10.1
Last host:      192.168.10.254
Broadcast:      192.168.10.255
```

Number of usable host addresses:

```text
2^8 - 2 = 254
```

Therefore:

```text
192.168.10.0/24
→ 254 usable host addresses
```

---

# 20. IPAM Address Inventory

An administrator can maintain an inventory such as:

| Address       | Host   | Type           | Status    |
| ------------- | ------ | -------------- | --------- |
| 192.168.10.1  | R1     | Router         | Used      |
| 192.168.10.2  | SW1    | Switch         | Used      |
| 192.168.10.3  | —      | Infrastructure | Available |
| 192.168.10.10 | Server | Server         | Used      |
| 192.168.10.11 | —      | Reserved       | Available |
| 192.168.10.12 | —      | Reserved       | Available |
| 192.168.10.21 | PC1    | DHCP           | Used      |
| 192.168.10.22 | PC2    | DHCP           | Used      |
| 192.168.10.23 | —      | DHCP           | Available |

This is the basic idea behind IPAM:

```text
IP Address
    ↓
Device
    ↓
Owner / Role
    ↓
Status
    ↓
Network / Subnet
```

---

# 21. DHCP vs IPAM

DHCP and IPAM are related but different.

### DHCP

DHCP automatically gives clients IP configuration.

```text
Client
  ↓
DHCP Request
  ↓
DHCP Server
  ↓
IP Address
```

### IPAM

IPAM manages the overall address space.

```text
IPAM
 │
 ├── Networks
 ├── Subnets
 ├── Addresses
 ├── Reservations
 ├── DHCP information
 └── DNS information
```

### Important

```text
DHCP → Assigns IP addresses

IPAM → Manages IP addresses
```

---

# 22. Step 14 — Check the ARP Table

On R1:

```cisco
show ip arp
```

You may see:

```text
Protocol  Address          Age
Internet  192.168.10.2    ...
Internet  192.168.10.10   ...
Internet  192.168.10.21   ...
Internet  192.168.10.22   ...
```

ARP allows the router to associate:

```text
IP Address ↔ MAC Address
```

This can help an administrator identify active devices.

---

# 23. Step 15 — Check the MAC Address Table

On SW1:

```cisco
show mac address-table
```

This shows learned MAC addresses and their switch ports.

Example:

```text
VLAN    MAC Address       Port
----    -----------       ----
1       AAAA.BBBB.CCCC   Fa0/1
1       DDDD.EEEE.FFFF   Fa0/2
```

This can help correlate:

```text
IP
 ↓
MAC
 ↓
Switch Port
 ↓
Device
```

---

# 24. Troubleshooting

## Problem 1 — PC does not receive an IP address

Check the router:

```cisco
show ip interface brief
```

Check DHCP:

```cisco
show ip dhcp pool
```

Check bindings:

```cisco
show ip dhcp binding
```

Check configuration:

```cisco
show running-config | section dhcp
```

---

## Problem 2 — PC receives an incorrect address

Verify:

```cisco
ipconfig
```

Check whether the address is inside the correct subnet:

```text
192.168.10.0/24
```

Check excluded addresses:

```cisco
show running-config | section dhcp
```

---

## Problem 3 — PC cannot reach the gateway

Test:

```bash
ping 192.168.10.1
```

Then check:

```cisco
show ip interface brief
```

Make sure the router interface is:

```text
up/up
```

---

## Problem 4 — Duplicate IP address

A duplicate IP can occur when:

```text
Static IP
     +
DHCP IP
     ↓
Same address
```

To avoid this, reserve static addresses with:

```cisco
ip dhcp excluded-address
```

Example:

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.20
```

---

# 25. Useful Verification Commands

### Router interfaces

```cisco
show ip interface brief
```

### DHCP leases

```cisco
show ip dhcp binding
```

### DHCP pool

```cisco
show ip dhcp pool
```

### DHCP configuration

```cisco
show running-config | section dhcp
```

### ARP table

```cisco
show ip arp
```

### Routing table

```cisco
show ip route
```

### Switch MAC table

```cisco
show mac address-table
```

### VLANs

```cisco
show vlan brief
```

### Switch interfaces

```cisco
show interfaces status
```

---

# 26. IPAM Workflow

A simple enterprise IPAM workflow is:

```text
1. Create network
       ↓
2. Divide network into subnets
       ↓
3. Reserve infrastructure addresses
       ↓
4. Configure DHCP ranges
       ↓
5. Assign static addresses
       ↓
6. Assign dynamic addresses
       ↓
7. Track addresses
       ↓
8. Verify connectivity
       ↓
9. Update IPAM documentation
```

---

# 27. Example Enterprise IPAM Plan

For a larger enterprise, the network could be divided into several subnets:

| VLAN | Department     | Network         | Gateway      |
| ---- | -------------- | --------------- | ------------ |
| 10   | Administration | 192.168.10.0/24 | 192.168.10.1 |
| 20   | Sales          | 192.168.20.0/24 | 192.168.20.1 |
| 30   | IT             | 192.168.30.0/24 | 192.168.30.1 |
| 40   | Servers        | 192.168.40.0/24 | 192.168.40.1 |
| 50   | Voice          | 192.168.50.0/24 | 192.168.50.1 |

This provides better organization and makes IP management easier.

---

# 28. Lab Verification Checklist

* [ ] R1 has IP `192.168.10.1`.
* [ ] SW1 has management IP `192.168.10.2`.
* [ ] Server has static IP `192.168.10.10`.
* [ ] DHCP pool `LAN` exists.
* [ ] Infrastructure addresses are excluded from DHCP.
* [ ] PC1 receives an IP address automatically.
* [ ] PC2 receives an IP address automatically.
* [ ] DHCP bindings are visible on R1.
* [ ] PC1 can ping the gateway.
* [ ] PC2 can ping the gateway.
* [ ] PCs can communicate with the server.
* [ ] ARP entries are visible.
* [ ] MAC addresses are visible on SW1.
* [ ] IPAM inventory is updated.
* [ ] Configuration is saved.

---

# 29. Lab Questions

### Question 1

What does IPAM mean?

### Question 2

What is the difference between IPAM and DHCP?

### Question 3

Why should infrastructure addresses be excluded from DHCP?

### Question 4

What is the purpose of the default gateway?

### Question 5

How many usable addresses exist in a `/24` network?

### Question 6

What command displays DHCP bindings?

### Question 7

What command displays the ARP table?

### Question 8

What command displays the switch MAC address table?

### Question 9

Why is an IP address inventory useful for an administrator?

### Question 10

What problem can occur if a static IP address is inside the DHCP pool?

---

# 30. Final Architecture

The complete IPAM concept can be represented as:

```text
                    IPAM
                     │
        ┌────────────┼────────────┐
        │            │            │
      IPs          DHCP          DNS
        │            │            │
        │            │            │
        ▼            ▼            ▼
   Address       Automatic      Name
   Inventory     Assignment     Resolution
        │            │            │
        └────────────┼────────────┘
                     │
                     ▼
                  Network
                  Devices
```

The basic relationship is:

```text
IPAM
 │
 ├── Network planning
 │
 ├── Subnet management
 │
 ├── IP address inventory
 │
 ├── DHCP management
 │
 └── DNS management
```

### Key idea

```text
IPAM → Organize and manage IP addresses
DHCP → Automatically assign IP addresses
DNS  → Resolve names to IP addresses
ARP  → Resolve IP addresses to MAC addresses
```
