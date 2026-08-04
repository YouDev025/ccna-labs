# Lab Firewall Basics

## 📌 Objective

The objective of this lab is to understand the **fundamentals of network firewalls** and practice basic traffic filtering and security policies.

In this lab, you will:

* Build a basic network topology with an internal LAN, firewall, and external network.
* Configure IP addressing and routing.
* Configure the firewall interfaces.
* Define trusted and untrusted network zones.
* Configure basic firewall rules.
* Allow legitimate traffic.
* Block unauthorized traffic.
* Test connectivity and firewall behavior.
* Verify the configuration using diagnostic commands.

---

# 🧠 Lab Overview

A **firewall** is a security device or software component that controls network traffic according to predefined security rules.

A basic firewall can:

* Allow traffic.
* Deny traffic.
* Filter traffic based on source IP.
* Filter traffic based on destination IP.
* Filter TCP/UDP ports.
* Control traffic between security zones.
* Log security events.

Basic architecture:

```text
                 Internet / WAN
                      |
                      |
                 Untrusted Zone
                      |
                      ↓
              +---------------+
              |   FIREWALL    |
              |               |
              | WAN      LAN   |
              +---------------+
                         |
                         |
                  Trusted Zone
                         |
                         ↓
                      Switch
                         |
              +----------+----------+
              |                     |
           PC-CLIENT             SERVER
```

---

# 🏗️ Topology

The recommended topology contains:

* 1 Internet/ISP router
* 1 Firewall
* 1 LAN switch
* 1 Client PC
* 1 Internal server

Example:

```text
                           INTERNET
                              |
                              |
                        +-----------+
                        | ISP Router|
                        +-----------+
                              |
                         203.0.113.0/30
                              |
                              |
                       +--------------+
                       |   FIREWALL   |
                       |              |
                       | WAN      LAN  |
                       +--------------+
                                  |
                             192.168.10.0/24
                                  |
                              +-------+
                              |  SW1  |
                              +-------+
                               /     \
                              /       \
                             ↓         ↓
                           PC1       SERVER
                       .10.10       .10.20
```

---

# 🌐 IP Addressing Plan

| Device     | Interface | IP Address    | Mask | Role                |
| ---------- | --------- | ------------- | ---- | ------------------- |
| ISP Router | WAN       | 203.0.113.1   | /30  | External Gateway    |
| Firewall   | WAN       | 203.0.113.2   | /30  | Untrusted Interface |
| Firewall   | LAN       | 192.168.10.1  | /24  | Internal Gateway    |
| PC1        | NIC       | 192.168.10.10 | /24  | Internal Client     |
| Server     | NIC       | 192.168.10.20 | /24  | Internal Server     |

Default gateway for internal devices:

```text
192.168.10.1
```

---

# 🔐 Security Zones

The firewall separates the network into different security zones.

## Untrusted Zone

The external/Internet network:

```text
203.0.113.0/30
```

This network should be considered **untrusted**.

---

## Trusted Zone

The internal LAN:

```text
192.168.10.0/24
```

This network contains internal clients and servers.

---

## Basic Security Principle

The firewall should follow:

```text
UNTRUSTED
    ↓
 FIREWALL
    ↓
 TRUSTED
```

Traffic should only be allowed when it is explicitly authorized.

---

# 🔧 Tasks

## Task 1 — Build the Topology

Create the following topology:

```text
ISP Router
     |
     |
Firewall
     |
     |
   Switch
   /    \
 PC1    Server
```

Connect the devices using Ethernet links.

Verify that all physical interfaces are connected and operational.

---

# Task 2 — Configure the ISP Router

Configure the external interface:

```text
IP Address: 203.0.113.1
Subnet Mask: 255.255.255.252
```

Example Cisco configuration:

```text
enable
configure terminal

interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
 no shutdown

end
```

Verify:

```text
show ip interface brief
```

Expected:

```text
GigabitEthernet0/0    203.0.113.1    up    up
```

---

# Task 3 — Configure the Firewall WAN Interface

Configure the firewall's external interface:

```text
IP Address: 203.0.113.2
Mask:       255.255.255.252
Gateway:    203.0.113.1
```

The WAN interface connects the firewall to the external network.

Logical representation:

```text
Firewall WAN
203.0.113.2
     |
     |
203.0.113.1
ISP Router
```

---

# Task 4 — Configure the Firewall LAN Interface

Configure the internal interface:

```text
IP Address: 192.168.10.1
Mask:       255.255.255.0
```

This address will be the default gateway for internal hosts.

```text
PC1
192.168.10.10
     |
     |
192.168.10.1
Firewall
```

---

# Task 5 — Configure PC1

Configure:

```text
IP Address:      192.168.10.10
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.10.1
```

Test the firewall's LAN interface:

```bash
ping 192.168.10.1
```

Expected:

```text
Reply from 192.168.10.1
```

---

# Task 6 — Configure the Internal Server

Configure:

```text
IP Address:      192.168.10.20
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.10.1
```

Enable an HTTP service on the server.

Example:

```text
HTTP
TCP Port: 80
```

---

# Task 7 — Configure Routing

The firewall needs a default route toward the ISP.

Example:

```text
Destination: 0.0.0.0/0
Next Hop:    203.0.113.1
```

Conceptually:

```text
Firewall
    |
    | Default Route
    ↓
203.0.113.1
    |
    ↓
Internet
```

Verify the routing table:

```text
show ip route
```

---

# 🛡️ Task 8 — Configure Basic Firewall Policy

The firewall should control traffic between:

```text
TRUSTED LAN
     |
     ↓
 FIREWALL
     |
     ↓
UNTRUSTED WAN
```

A basic security policy can be:

| Source | Destination | Service    | Action         |
| ------ | ----------- | ---------- | -------------- |
| LAN    | WAN         | ICMP       | Allow          |
| LAN    | WAN         | HTTP       | Allow          |
| LAN    | WAN         | HTTPS      | Allow          |
| WAN    | LAN         | Any        | Deny           |
| WAN    | Server      | HTTP       | Deny initially |
| LAN    | Firewall    | Management | Allow          |

The basic principle is:

```text
Allow legitimate traffic
Block unauthorized traffic
```

---

# Task 9 — Configure ICMP Filtering

ICMP can be used for connectivity testing.

Allow internal clients to send ICMP toward the external network.

Example:

```text
LAN → WAN
ICMP → ALLOW
```

But external devices should not automatically be allowed to ping internal hosts:

```text
WAN → LAN
ICMP → DENY
```

This demonstrates basic traffic filtering.

---

# Task 10 — Configure HTTP Filtering

Allow internal clients to access external HTTP services:

```text
LAN → WAN
TCP/80 → ALLOW
```

Block unsolicited external HTTP connections toward the internal LAN:

```text
WAN → LAN
TCP/80 → DENY
```

---

# Task 11 — Configure HTTPS Filtering

Allow secure web traffic from the internal network:

```text
LAN → WAN
TCP/443 → ALLOW
```

HTTPS uses:

```text
TCP Port 443
```

---

# Task 12 — Configure DNS Filtering

If DNS is used in the topology, allow DNS requests from the internal network.

Typical DNS:

```text
UDP/53
TCP/53
```

Example:

```text
LAN → DNS Server
UDP/53 → ALLOW
TCP/53 → ALLOW
```

---

# 🧱 Task 13 — Default Deny Policy

A firewall should generally have a restrictive policy.

Conceptually:

```text
ALLOW required traffic
DENY everything else
```

Example:

```text
LAN → WAN → HTTP   → ALLOW
LAN → WAN → HTTPS  → ALLOW
LAN → WAN → DNS    → ALLOW
LAN → WAN → ICMP   → ALLOW

WAN → LAN → ANY    → DENY
```

This is an important security principle.

---

# 🔍 Task 14 — Verify Firewall Interfaces

Check interface status:

```text
show ip interface brief
```

Verify that interfaces are:

```text
up
up
```

Example:

```text
Interface              IP Address       Status
---------------------------------------------------
WAN                    203.0.113.2      up
LAN                    192.168.10.1     up
```

---

# 🔎 Task 15 — Verify Routing

Use:

```text
show ip route
```

Verify the presence of:

```text
C 192.168.10.0/24
C 203.0.113.0/30
S* 0.0.0.0/0
```

The default route should point toward:

```text
203.0.113.1
```

---

# 🧪 Task 16 — Test Internal Connectivity

From PC1:

```bash
ping 192.168.10.1
```

Expected:

```text
SUCCESS
```

Then test the internal server:

```bash
ping 192.168.10.20
```

Expected:

```text
SUCCESS
```

---

# 🌐 Task 17 — Test External Connectivity

From the internal client, test the external gateway:

```bash
ping 203.0.113.1
```

Expected:

```text
SUCCESS
```

If routing and firewall policies are correctly configured, the request should pass.

---

# 🚫 Task 18 — Test Blocked Traffic

Attempt to initiate a connection from the untrusted network toward the internal network.

Example:

```text
WAN → LAN
```

Expected:

```text
DENIED
```

This verifies that the firewall is protecting the internal network.

---

# 🌍 Task 19 — Test HTTP Service

If an external HTTP server exists, test:

```text
http://<external-server-ip>
```

From the internal PC.

Expected:

```text
HTTP connection → ALLOWED
```

Then attempt an unsolicited HTTP connection from WAN toward the internal server.

Expected:

```text
WAN → Internal Server
TCP/80 → BLOCKED
```

---

# 📊 Task 20 — Verify Firewall Rules

Use platform-specific commands to inspect firewall policies.

Possible commands include:

```text
show running-config
show access-lists
show access-list
show firewall
show policy
show logging
show connections
show sessions
```

> The exact commands depend on the firewall platform.

---

# 📝 Example ACL Logic

A basic ACL can conceptually look like:

```text
1. Allow LAN → HTTP
2. Allow LAN → HTTPS
3. Allow LAN → DNS
4. Allow LAN → ICMP
5. Deny WAN → LAN
6. Log denied traffic
```

Example logical policy:

```text
             FIREWALL
                 |
      +----------+----------+
      |                     |
    TRUSTED              UNTRUSTED
192.168.10.0/24       203.0.113.0/30
      |                     |
      ↓                     ↓
     LAN                INTERNET

LAN → WAN
HTTP      ALLOW
HTTPS     ALLOW
DNS       ALLOW
ICMP      ALLOW

WAN → LAN
ANY       DENY
```

---

# 🔥 Stateful Firewall Concept

A stateful firewall keeps track of active connections.

Example:

```text
PC1
 |
 | TCP SYN
 ↓
Firewall
 |
 ↓
Internet Server
```

If the firewall allows the connection, it maintains state information.

Return traffic:

```text
Internet Server
       |
       | TCP response
       ↓
    Firewall
       |
       ↓
      PC1
```

The return traffic can be permitted because it belongs to an established connection.

---

# 🧪 Verification Checklist

| Test                     | Expected Result |
| ------------------------ | --------------- |
| Firewall LAN interface   | ✅ UP            |
| Firewall WAN interface   | ✅ UP            |
| PC → Firewall LAN        | ✅ Allowed       |
| PC → Internal Server     | ✅ Allowed       |
| Firewall → ISP           | ✅ Allowed       |
| LAN → HTTP               | ✅ Allowed       |
| LAN → HTTPS              | ✅ Allowed       |
| LAN → DNS                | ✅ Allowed       |
| LAN → ICMP               | ✅ Allowed       |
| WAN → LAN                | ❌ Denied        |
| Unsolicited WAN → Server | ❌ Denied        |
| Firewall rules           | ✅ Correct       |
| Routing table            | ✅ Correct       |
| Logs                     | ✅ Available     |

---

# 🛠️ Troubleshooting

## Problem 1 — PC cannot reach the firewall

Check:

```text
IP address
Subnet mask
Default gateway
LAN interface
Cable/link
```

Verify:

```text
ping 192.168.10.1
```

---

## Problem 2 — PC cannot access the Internet

Check:

```text
Default gateway
Firewall default route
WAN interface
ISP router
Firewall policy
NAT configuration if required
```

Verify:

```text
show ip route
```

---

## Problem 3 — Traffic is being blocked unexpectedly

Check:

```text
Firewall rules
ACL order
Source address
Destination address
Protocol
TCP/UDP port
```

Remember:

```text
Firewall rules are usually evaluated in order.
```

---

## Problem 4 — External traffic reaches the internal network

Check:

```text
WAN → LAN rules
Default deny policy
ACL configuration
NAT configuration
Firewall state table
```

This is a critical security issue.

---

# 🧠 Important Concepts

| Concept             | Description                           |
| ------------------- | ------------------------------------- |
| Firewall            | Controls network traffic              |
| Trusted Zone        | Internal network                      |
| Untrusted Zone      | External network                      |
| ACL                 | Access Control List                   |
| Rule                | Defines allowed/denied traffic        |
| Stateful Inspection | Tracks active connections             |
| Inbound Traffic     | Traffic entering an interface         |
| Outbound Traffic    | Traffic leaving an interface          |
| Default Deny        | Blocks traffic not explicitly allowed |
| Logging             | Records firewall events               |
| NAT                 | Translates IP addresses               |
| DMZ                 | Isolated network for public services  |

---

# 🎯 Learning Outcomes

After completing this lab, you should be able to:

* Explain the role of a firewall.
* Identify trusted and untrusted zones.
* Configure firewall interfaces.
* Configure basic routing.
* Create basic firewall policies.
* Allow authorized traffic.
* Block unauthorized traffic.
* Understand ACLs.
* Understand stateful firewall behavior.
* Test firewall rules.
* Analyze firewall logs.
* Troubleshoot connectivity problems.

---

# 📌 Firewall Security Rules to Remember

### Rule 1

```text
Do not trust external traffic by default.
```

### Rule 2

```text
Allow only required services.
```

### Rule 3

```text
Block unauthorized traffic.
```

### Rule 4

```text
Use logging to monitor denied traffic.
```

### Rule 5

```text
Review firewall rules regularly.
```

---

# ✅ Final Verification

Before completing the lab:

```text
[✓] Topology built
[✓] IP addresses configured
[✓] Firewall WAN configured
[✓] Firewall LAN configured
[✓] Routing configured
[✓] Security zones configured
[✓] Firewall rules configured
[✓] HTTP tested
[✓] HTTPS tested
[✓] DNS tested
[✓] ICMP tested
[✓] Unauthorized traffic blocked
[✓] show commands executed
[✓] Firewall behavior verified
[✓] Troubleshooting performed
```

---

# 🏁 Conclusion

This lab introduces the fundamental concepts of **network firewall security**.

The main architecture is:

```text
             UNTRUSTED
              INTERNET
                  |
                  ↓
            +-----------+
            | FIREWALL  |
            +-----------+
                  |
                  ↓
              TRUSTED
                 LAN
                  |
            +-----+-----+
            |           |
           PC         SERVER
```

The firewall acts as a security boundary between the trusted internal network and the untrusted external network.

The key principle is:

```text
        ALLOW what is required
                 +
        DENY what is unauthorized
                 +
        LOG suspicious traffic
                 =
        BASIC NETWORK SECURITY
```
