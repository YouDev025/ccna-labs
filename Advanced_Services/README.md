# Advanced Services Labs

This section covers more advanced service and infrastructure topics, building on foundational networking knowledge. Each lab focuses on a core enterprise networking concept, with hands-on configuration and verification steps to reinforce learning.

## Overview

These labs are designed to be completed in a virtual or physical lab environment (e.g., GNS3, EVE-NG, Packet Tracer, or physical lab gear). Each lab should be documented with device configurations, topology diagrams, and verification output so results can be reviewed and reproduced later.

## Prerequisites

- Basic understanding of routing and switching concepts
- Familiarity with CLI configuration on your chosen platform (Cisco IOS, Juniper, etc.)
- A working lab topology with at least 2–3 routers/switches and end hosts
- Access to a text editor or lab notebook for recording configurations

## General Instructions

For **every** lab in this section:

1. **Set up the topology** — build or import the suggested topology before configuring services.
2. **Record all configurations** — save the running config for each device (e.g., `show run` output) into the lab's documentation folder.
3. **Verify behavior** — use appropriate `show`/`debug` commands or packet captures to confirm the service is working as intended.
4. **Document results** — note any unexpected behavior, troubleshooting steps taken, and final verified state.
5. **Clean up** — where relevant, remove test-only configuration before moving to the next lab.

---

## Suggested Labs

### 1. QoS Basics

**Objective:** Understand how to classify, mark, and prioritize traffic to ensure critical applications receive adequate bandwidth and low latency.

**Topics covered:**
- Traffic classification and marking (DSCP, CoS)
- Queuing mechanisms (e.g., priority queuing, WFQ, CBWFQ)
- Policing vs. shaping
- Applying QoS policies to interfaces

**Suggested steps:**
1. Identify traffic types to prioritize (e.g., voice, video, best-effort data).
2. Configure a class-map to classify traffic.
3. Configure a policy-map to define QoS actions (marking, queuing, policing).
4. Apply the policy to the relevant interface(s).
5. Generate test traffic and verify queuing/marking behavior.

**Verification:**
- `show policy-map interface`
- `show class-map`
- Packet capture showing DSCP/CoS markings

---

### 2. IPv6

**Objective:** Configure and verify basic IPv6 addressing, routing, and neighbor discovery.

**Topics covered:**
- IPv6 address types (unicast, link-local, multicast)
- Static and dynamic IPv6 addressing (SLAAC, DHCPv6)
- IPv6 routing (static routes, and/or OSPFv3/EIGRPv6)
- Neighbor Discovery Protocol (NDP)

**Suggested steps:**
1. Enable IPv6 on relevant interfaces.
2. Assign IPv6 addresses (manual and/or SLAAC).
3. Configure IPv6 routing between devices.
4. Verify neighbor discovery and reachability.

**Verification:**
- `show ipv6 interface brief`
- `show ipv6 route`
- `show ipv6 neighbors`
- `ping6` / `traceroute6` between hosts

---

### 3. VPN Concepts

**Objective:** Understand and configure a basic site-to-site VPN to secure traffic between two networks.

**Topics covered:**
- VPN types (site-to-site, remote-access)
- IPsec fundamentals (IKE Phase 1 and Phase 2)
- Encryption and authentication methods
- Tunnel establishment and troubleshooting

**Suggested steps:**
1. Define interesting traffic (ACL) to trigger the VPN.
2. Configure IKE Phase 1 (ISAKMP) policy.
3. Configure IPsec Phase 2 (transform set, crypto map).
4. Apply the crypto map to the outbound interface.
5. Generate traffic between sites and confirm the tunnel comes up.

**Verification:**
- `show crypto isakmp sa`
- `show crypto ipsec sa`
- `show crypto session`
- Ping/traceroute across the tunnel with packet capture confirming encryption

---

### 4. High Availability Basics

**Objective:** Implement redundancy mechanisms to ensure network services remain available during a device or link failure.

**Topics covered:**
- First Hop Redundancy Protocols (HSRP, VRRP, or GLBP)
- Redundant links and basic failover
- Preemption and priority tuning
- Failure simulation and recovery testing

**Suggested steps:**
1. Configure two or more gateways with a shared virtual IP (HSRP/VRRP/GLBP).
2. Set priorities to define the active/primary device.
3. Verify end-host traffic flows through the active gateway.
4. Simulate a failure (shut down the active device/interface) and confirm failover.
5. Restore the primary device and verify preemption (if configured).

**Verification:**
- `show standby brief` (HSRP) / `show vrrp brief` (VRRP)
- Continuous ping during failover to observe packet loss (if any)
- Confirm failover and recovery timing

---

## Deliverables

For each lab, submit:
- [ ] Topology diagram
- [ ] Device configuration files (or `show run` output)
- [ ] Verification command output
- [ ] Short write-up of observed behavior and any troubleshooting performed