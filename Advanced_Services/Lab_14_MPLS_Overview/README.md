# Lab MPLS Overview

## Objective

This lab introduces the fundamentals of **Multiprotocol Label Switching (MPLS)** and demonstrates how service providers interconnect multiple customer sites through an MPLS-enabled backbone. The lab focuses on understanding the MPLS architecture, provider edge (PE) and provider (P) routers, customer edge (CE) routers, and end-to-end connectivity.

---

## Network Topology

### Devices

- 2 Customer Edge (CE) Routers
- 2 Provider Edge (PE) Routers
- 2 Provider (P) Routers
- 2 Cisco 2960 Switches
- 4 PCs
- Serial and Ethernet Links

---

### Topology Overview

```
                 Customer Site A

      PC0                    PC1
       |                      |
   +--------- Switch0 ---------+
              |
            CE1 Router
              |
        GigabitEthernet
              |
            PE1 Router
              |
         MPLS Backbone
              |
          P1 -------- P2
              |        |
            PE2 Router
              |
        GigabitEthernet
              |
            CE2 Router
              |
   +--------- Switch1 ---------+
      |                      |
     PC2                    PC3

                 Customer Site B
```

---

## IP Addressing

### Customer Site A

| Device | Interface | IP Address |
|---------|-----------|------------|
| CE1 | G0/0 | 192.168.10.1/24 |
| PC0 | Fa0 | 192.168.10.10/24 |
| PC1 | Fa0 | 192.168.10.20/24 |

---

### Customer Site B

| Device | Interface | IP Address |
|---------|-----------|------------|
| CE2 | G0/0 | 192.168.20.1/24 |
| PC2 | Fa0 | 192.168.20.10/24 |
| PC3 | Fa0 | 192.168.20.20/24 |

---

### MPLS Backbone

| Link | Network |
|------|---------|
| CE1 ↔ PE1 | 10.0.0.0/30 |
| PE1 ↔ P1 | 10.0.0.4/30 |
| P1 ↔ P2 | 10.0.0.8/30 |
| P2 ↔ PE2 | 10.0.0.12/30 |
| PE2 ↔ CE2 | 10.0.0.16/30 |

---

## MPLS Architecture

### Customer Edge (CE)

- Connects customer networks to the provider.
- Does not run MPLS.
- Exchanges routes with the Provider Edge router.

### Provider Edge (PE)

- Connects customer sites to the MPLS cloud.
- Maintains customer routing information.
- Assigns and removes MPLS labels.

### Provider (P)

- Core service provider routers.
- Forward packets based on MPLS labels.
- Do not maintain customer routing tables.

---

## Tasks

- Build the MPLS topology.
- Configure IP addressing on all routers.
- Configure provider and customer links.
- Configure static or dynamic routing between CE and PE routers.
- Enable MPLS on provider-facing interfaces (where supported).
- Verify label forwarding.
- Test end-to-end connectivity.

---

## Routing

Possible routing protocols include:

- Static Routing
- OSPF
- EIGRP
- BGP (Provider/Customer)
- MP-BGP (Advanced MPLS VPN deployments)

> **Note:** Cisco Packet Tracer has limited or no support for full MPLS features. This lab is intended to demonstrate the MPLS architecture and traffic flow conceptually. If MPLS commands are unavailable, configure IP routing to simulate provider connectivity.

---

## Verification Commands

### Interface Status

```bash
show ip interface brief
```

### Routing Table

```bash
show ip route
```

### MPLS Interfaces *(if supported)*

```bash
show mpls interfaces
```

### MPLS Forwarding Table *(if supported)*

```bash
show mpls forwarding-table
```

### Label Information *(if supported)*

```bash
show mpls ldp neighbor
```

### Connectivity Test

```bash
ping <destination-ip>
```

### Path Verification

```bash
traceroute <destination-ip>
```

---

## Expected Results

- All interfaces are operational.
- Customer Site A communicates successfully with Customer Site B.
- Provider routers forward traffic correctly.
- Routing tables contain all required routes.
- MPLS labels are exchanged and used for forwarding (when supported).
- End-to-end connectivity is maintained through the provider backbone.

---

## Skills Practiced

- MPLS Fundamentals
- Service Provider Architecture
- CE, PE, and P Router Roles
- IP Addressing
- Static Routing
- Dynamic Routing
- MPLS Concepts
- Network Verification
- Connectivity Testing
- Cisco IOS Commands

---

## Key Concepts

- Multiprotocol Label Switching (MPLS)
- Label Switching
- Label Distribution Protocol (LDP)
- Forwarding Equivalence Class (FEC)
- Provider Backbone
- Provider Edge (PE)
- Provider (P)
- Customer Edge (CE)
- MPLS VPN Concepts
- Traffic Engineering (Overview)

---

## Conclusion

This lab provides an introduction to MPLS concepts by simulating a service provider network interconnecting two customer sites. Although Packet Tracer has limited support for MPLS, the topology and routing configuration illustrate how customer traffic traverses a provider backbone and establish the foundation for more advanced MPLS and Layer 3 VPN deployments in enterprise and service provider environments.