# Lab Cloud Connectivity

## 📌 Objective

The objective of this lab is to understand and practice the fundamentals of **Cloud Connectivity** between an on-premises network and a cloud environment.

In this lab, you will:

* Build an on-premises network connected to a simulated cloud environment.
* Configure IP addressing and routing.
* Configure a WAN or Internet connection.
* Configure cloud-side networking.
* Establish connectivity between on-premises and cloud resources.
* Configure static routes or dynamic routing when supported.
* Test connectivity between local and cloud resources.
* Verify network paths and service availability.
* Troubleshoot common cloud connectivity problems.

---

# 🧠 Lab Overview

**Cloud Connectivity** refers to the network connection between an organization's local infrastructure and resources hosted in a cloud environment.

A basic architecture is:

```text
                 ON-PREMISES NETWORK
                        |
                        |
                    +-------+
                    | LAN   |
                    +-------+
                        |
                     Router
                        |
                        |
                   Internet/WAN
                        |
                        |
                +---------------+
                | Cloud Network |
                +---------------+
                     /       \
                    /         \
                   ↓           ↓
               Cloud VM 1   Cloud VM 2
```

The objective is to allow systems in the local network to communicate with resources hosted in the cloud.

---

# 🏗️ Lab Architecture

The lab consists of two main environments:

### On-Premises

```text
Client
   |
   |
Switch
   |
Router
   |
WAN / Internet
```

### Cloud

```text
Cloud Gateway
     |
     |
Cloud Network
     |
 +---+---+
 |       |
VM1     VM2
```

Complete architecture:

```text
                           INTERNET / WAN
                                 |
                                 |
                    +----------------------+
                    |   On-Premises Router |
                    +----------------------+
                                 |
                              LAN
                                 |
                              +----+
                              | SW1|
                              +----+
                               |
                              PC1

                                 |
                                 |
                         WAN / INTERNET
                                 |
                                 ↓
                    +----------------------+
                    |    Cloud Gateway     |
                    +----------------------+
                                 |
                         Cloud Network
                         10.20.0.0/24
                            /      \
                           /        \
                          ↓          ↓
                     Cloud VM1   Cloud VM2
                    10.20.0.10   10.20.0.20
```

---

# 🌐 IP Addressing Plan

| Device         | Interface | IP Address    | Mask | Role          |
| -------------- | --------- | ------------- | ---- | ------------- |
| PC1            | NIC       | 192.168.10.10 | /24  | Local Client  |
| On-Prem Router | LAN       | 192.168.10.1  | /24  | LAN Gateway   |
| On-Prem Router | WAN       | 203.0.113.2   | /30  | WAN           |
| Cloud Gateway  | WAN       | 203.0.113.1   | /30  | Cloud Gateway |
| Cloud Gateway  | LAN       | 10.20.0.1     | /24  | Cloud Gateway |
| Cloud VM1      | NIC       | 10.20.0.10    | /24  | Cloud Server  |
| Cloud VM2      | NIC       | 10.20.0.20    | /24  | Cloud Server  |

> **Note:** The addressing scheme can be modified according to the cloud platform or simulator used.

---

# ☁️ Cloud Networking Concepts

## Virtual Network

A cloud provider normally offers a logical network such as:

```text
Virtual Network
      |
      +--- Subnet
      |
      +--- Route Table
      |
      +--- Security Rules
      |
      +--- Virtual Machines
```

Examples of technologies/concepts include:

* VPC
* VNet
* Virtual Subnet
* Route Tables
* Security Groups
* Network ACLs
* Internet Gateway
* NAT Gateway
* VPN Gateway

---

# 🔑 Important Concepts

## 1. On-Premises Network

The infrastructure located inside the organization.

Example:

```text
192.168.10.0/24
```

---

## 2. Cloud Network

The virtual network hosted by the cloud provider.

Example:

```text
10.20.0.0/24
```

---

## 3. Cloud Gateway

The gateway provides connectivity between the local network and cloud network.

```text
On-Premises
     |
     ↓
Cloud Gateway
     |
     ↓
Cloud Network
```

---

## 4. Route Table

A route table determines where packets should be forwarded.

Example:

```text
Destination       Next Hop
--------------------------------
10.20.0.0/24      Cloud Gateway
0.0.0.0/0         Internet Gateway
```

---

# 🔧 Tasks

## Task 1 — Build the Topology

Create the following topology:

```text
              ON-PREMISES
                  |
                 PC1
                  |
                 SW1
                  |
              Router R1
                  |
                  |
              WAN / ISP
                  |
                  |
             Cloud Gateway
                  |
             Cloud Network
              /          \
             /            \
          VM1              VM2
```

Verify that all physical links are operational.

---

# Task 2 — Configure the Local Client

Configure PC1:

```text
IP Address:      192.168.10.10
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.10.1
```

Test the local gateway:

```bash
ping 192.168.10.1
```

Expected:

```text
Reply from 192.168.10.1
```

---

# Task 3 — Configure the On-Premises Router

Configure the LAN interface:

```text
IP Address: 192.168.10.1
Mask:       255.255.255.0
```

Example Cisco configuration:

```text
enable
configure terminal

interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown

end
```

Verify:

```text
show ip interface brief
```

---

# Task 4 — Configure the WAN Interface

Configure the WAN interface:

```text
IP Address: 203.0.113.2
Mask:       255.255.255.252
```

Example:

```text
interface GigabitEthernet0/1
 ip address 203.0.113.2 255.255.255.252
 no shutdown
```

Verify:

```text
show ip interface brief
```

---

# Task 5 — Configure the Cloud Gateway

Configure the simulated cloud gateway:

```text
WAN:
203.0.113.1/30

Cloud LAN:
10.20.0.1/24
```

Logical architecture:

```text
              Cloud Gateway
             /              \
     203.0.113.1          10.20.0.1
          |                    |
          |                    |
        WAN              Cloud Network
                              |
                         10.20.0.0/24
```

---

# Task 6 — Configure Cloud VM1

Configure:

```text
Hostname:        CLOUD-VM-01
IP Address:      10.20.0.10
Subnet Mask:     255.255.255.0
Default Gateway: 10.20.0.1
```

---

# Task 7 — Configure Cloud VM2

Configure:

```text
Hostname:        CLOUD-VM-02
IP Address:      10.20.0.20
Subnet Mask:     255.255.255.0
Default Gateway: 10.20.0.1
```

---

# Task 8 — Configure Routing

The on-premises router needs a route toward the cloud network.

Configure:

```text
Destination: 10.20.0.0/24
Next Hop:    203.0.113.1
```

Conceptually:

```text
192.168.10.0/24
       |
       ↓
      R1
       |
       | 203.0.113.0/30
       |
       ↓
Cloud Gateway
       |
       ↓
10.20.0.0/24
```

Example static route:

```text
ip route 10.20.0.0 255.255.255.0 203.0.113.1
```

Verify:

```text
show ip route
```

Expected:

```text
S 10.20.0.0/24 via 203.0.113.1
```

---

# Task 9 — Configure Cloud-Side Routing

The cloud gateway needs to know how to reach the on-premises network.

Configure a route:

```text
Destination: 192.168.10.0/24
Next Hop:    203.0.113.2
```

Logical routing:

```text
Cloud VM
   |
   ↓
Cloud Gateway
   |
   ↓
203.0.113.2
   |
   ↓
On-Prem Router
   |
   ↓
192.168.10.0/24
```

---

# Task 10 — Configure Cloud Security Rules

Cloud environments commonly use security rules to control traffic.

Allow:

```text
Source: 192.168.10.0/24
Destination: 10.20.0.0/24
ICMP: ALLOW
```

For an HTTP service:

```text
Source: 192.168.10.0/24
Destination: 10.20.0.10
TCP/80: ALLOW
```

For HTTPS:

```text
Source: 192.168.10.0/24
Destination: 10.20.0.10
TCP/443: ALLOW
```

The general principle is:

```text
ALLOW required traffic
DENY unnecessary traffic
```

---

# Task 11 — Test Local Connectivity

From PC1:

```bash
ping 192.168.10.1
```

Expected:

```text
SUCCESS
```

Then test the WAN gateway:

```bash
ping 203.0.113.1
```

Expected:

```text
SUCCESS
```

---

# Task 12 — Test Cloud Connectivity

From PC1:

```bash
ping 10.20.0.10
```

Then:

```bash
ping 10.20.0.20
```

Expected:

```text
Reply from 10.20.0.10
Reply from 10.20.0.20
```

This verifies end-to-end connectivity:

```text
PC1
 ↓
LAN
 ↓
R1
 ↓
WAN
 ↓
Cloud Gateway
 ↓
Cloud Network
 ↓
Cloud VM
```

---

# Task 13 — Test Reverse Connectivity

From Cloud VM1, test the local network:

```bash
ping 192.168.10.10
```

Expected:

```text
SUCCESS
```

This confirms that routing works in both directions.

---

# Task 14 — Configure an Application Service

Configure a web service on Cloud VM1.

Example:

```text
Server: CLOUD-VM-01
IP: 10.20.0.10
Protocol: HTTP
Port: 80
```

From PC1:

```text
http://10.20.0.10
```

Expected:

```text
Cloud VM 1 Web Service
```

---

# Task 15 — Test HTTPS

If HTTPS is configured:

```text
https://10.20.0.10
```

Verify that TCP port 443 is permitted by the cloud security rules.

---

# 🔍 Verification Commands

On Cisco routers, use:

```text
show ip interface brief
show ip route
show arp
show interfaces
show running-config
```

Test connectivity:

```text
ping 192.168.10.10
ping 203.0.113.1
ping 10.20.0.10
ping 10.20.0.20
```

Use traceroute:

```text
traceroute 10.20.0.10
```

On Linux-based cloud VMs:

```bash
ip addr
ip route
ping 10.20.0.1
ping 192.168.10.10
ss -tuln
```

---

# 🧭 Traceroute Verification

Use:

```text
traceroute 10.20.0.10
```

Expected logical path:

```text
PC1
 ↓
192.168.10.1
 ↓
203.0.113.1
 ↓
10.20.0.1
 ↓
10.20.0.10
```

This helps identify routing problems.

---

# 🧪 Connectivity Test Matrix

| Source               | Destination   | Protocol | Expected |
| -------------------- | ------------- | -------- | -------- |
| PC1                  | Local Gateway | ICMP     | ✅ Allow  |
| PC1                  | WAN Gateway   | ICMP     | ✅ Allow  |
| PC1                  | Cloud VM1     | ICMP     | ✅ Allow  |
| PC1                  | Cloud VM2     | ICMP     | ✅ Allow  |
| Cloud VM1            | PC1           | ICMP     | ✅ Allow  |
| PC1                  | Cloud VM1     | HTTP     | ✅ Allow  |
| PC1                  | Cloud VM1     | HTTPS    | ✅ Allow  |
| Unauthorized Network | Cloud VM      | Any      | ❌ Deny   |

---

# 🔐 Security Considerations

Cloud connectivity should not mean that all traffic is automatically permitted.

Use the principle of **least privilege**.

For example:

```text
                    CLOUD
                      |
              +---------------+
              | Security Rule |
              +---------------+
                 /         \
                /           \
             ALLOW          DENY
               |              |
        Required Traffic   Unnecessary
```

Only expose required services.

For example:

```text
HTTP     TCP/80     → Required
HTTPS    TCP/443    → Required
SSH      TCP/22     → Admin only
ICMP                → Troubleshooting only
```

---

# 🛡️ VPN-Based Cloud Connectivity

In real-world environments, cloud connectivity is often secured using a VPN.

Example:

```text
               ON-PREMISES
                   |
                Router
                   |
              VPN Gateway
                   |
             Encrypted VPN
                   |
                   ↓
             Cloud VPN Gateway
                   |
              Cloud Network
                   |
             +-----+-----+
             |           |
            VM1         VM2
```

The VPN provides an encrypted tunnel between the two networks.

Typical technologies include:

* IPsec
* Site-to-Site VPN
* SSL VPN
* GRE over IPsec
* Cloud VPN Gateway

---

# 🔐 Site-to-Site VPN Concept

The objective is to connect:

```text
On-Premises:
192.168.10.0/24
```

with:

```text
Cloud:
10.20.0.0/24
```

through an encrypted tunnel.

Conceptually:

```text
192.168.10.0/24
       |
       ↓
VPN Gateway
       |
       |==== ENCRYPTED VPN ====|
                              |
                              ↓
                         VPN Gateway
                              |
                              ↓
                       10.20.0.0/24
```

---

# 🛠️ Troubleshooting

## Problem 1 — PC cannot reach the cloud VM

Check:

```text
IP addressing
Default gateway
On-premises routing
Cloud routing
Security rules
Firewall rules
```

Verify:

```text
show ip route
```

---

## Problem 2 — PC reaches gateway but not cloud

Check the route:

```text
10.20.0.0/24
```

The on-premises router should have:

```text
S 10.20.0.0/24 via 203.0.113.1
```

---

## Problem 3 — Cloud VM can reach its gateway but not PC1

Check the reverse route:

```text
192.168.10.0/24
```

The cloud gateway must know how to reach the local network.

---

## Problem 4 — Ping works but HTTP does not

Check:

```text
HTTP service
TCP port 80
Cloud security rules
Firewall
Server listening ports
```

On Linux:

```bash
ss -tuln
```

---

## Problem 5 — HTTP works but HTTPS does not

Check:

```text
HTTPS service
TCP/443
TLS configuration
Cloud security rules
Firewall
```

---

# 📊 Troubleshooting Flow

```text
              Connectivity Problem
                       |
                       ↓
               Check IP Address
                       |
                       ↓
                Check Gateway
                       |
                       ↓
                Check Routing
                       |
                       ↓
              Check Security Rules
                       |
                       ↓
                 Check Firewall
                       |
                       ↓
               Check Application
```

---

# 🧠 Important Concepts

| Concept            | Meaning                                                 |
| ------------------ | ------------------------------------------------------- |
| Cloud Connectivity | Network connection between local and cloud resources    |
| VPC/VNet           | Virtual cloud network                                   |
| Subnet             | Logical IP network inside a cloud network               |
| Route Table        | Controls packet forwarding                              |
| Cloud Gateway      | Connects networks                                       |
| Internet Gateway   | Provides Internet connectivity                          |
| NAT Gateway        | Provides outbound Internet access for private resources |
| Security Group     | Controls traffic to cloud resources                     |
| Network ACL        | Controls subnet-level traffic                           |
| VPN                | Secure encrypted network connection                     |
| Site-to-Site VPN   | Connects two networks through an encrypted tunnel       |
| Hybrid Cloud       | Combination of on-premises and cloud infrastructure     |

---

# 🎯 Learning Outcomes

After completing this lab, you should be able to:

* Explain cloud connectivity.
* Understand on-premises and cloud networks.
* Configure IP addressing.
* Configure WAN connectivity.
* Configure static routes.
* Understand cloud route tables.
* Configure cloud security rules.
* Test connectivity between local and cloud resources.
* Use ping and traceroute.
* Verify HTTP/HTTPS services.
* Troubleshoot routing problems.
* Understand the role of VPNs in cloud connectivity.
* Understand basic hybrid-cloud architecture.

---

# 📌 Key Concepts to Memorize

### Cloud Connectivity

```text
Cloud Connectivity = Connecting local infrastructure to cloud resources.
```

### VPC / VNet

```text
VPC/VNet = Virtual network created inside a cloud environment.
```

### Route Table

```text
Route Table = Determines where network traffic is forwarded.
```

### Security Group

```text
Security Group = Controls allowed and denied traffic to cloud resources.
```

### VPN

```text
VPN = Encrypted tunnel used to securely connect networks.
```

### Hybrid Cloud

```text
Hybrid Cloud = Combination of on-premises infrastructure and cloud infrastructure.
```

---

# 📋 Final Architecture

```text
                    ON-PREMISES
                 192.168.10.0/24
                        |
                       PC1
                    .10.10
                        |
                       SW1
                        |
                       R1
                  .10.1 / WAN
                        |
                        |
                 203.0.113.0/30
                        |
                        |
                Cloud Gateway
                        |
                        |
                 10.20.0.1/24
                        |
                 CLOUD NETWORK
                 10.20.0.0/24
                    /       \
                   /         \
                  ↓           ↓
             Cloud VM1    Cloud VM2
             .20.0.10     .20.0.20
```

---

# 🧪 Final Verification

Before completing the lab:

```text
[✓] Topology built
[✓] On-premises LAN configured
[✓] Client configured
[✓] Router LAN configured
[✓] WAN connectivity configured
[✓] Cloud gateway configured
[✓] Cloud network configured
[✓] Cloud VMs configured
[✓] Routing configured
[✓] Cloud security rules configured
[✓] Local connectivity tested
[✓] Cloud connectivity tested
[✓] Reverse connectivity tested
[✓] HTTP tested
[✓] HTTPS tested
[✓] Ping tested
[✓] Traceroute tested
[✓] show commands executed
[✓] Troubleshooting completed
```

---

# 🏁 Conclusion

This lab demonstrates the fundamentals of **Cloud Connectivity** and how an on-premises network can communicate with resources hosted in a cloud environment.

The main communication path is:

```text
On-Premises Client
        ↓
Local Network
        ↓
On-Premises Router
        ↓
WAN / Internet / VPN
        ↓
Cloud Gateway
        ↓
Cloud Network
        ↓
Cloud Resources
```

The key principle is:

```text
        ON-PREMISES
             |
             ↓
      Secure Connectivity
             |
      +------+------+
      |             |
     WAN           VPN
      |             |
      +------+------+
             |
             ↓
           CLOUD
             |
       +-----+-----+
       |           |
      VM1         VM2
```

A properly designed cloud connection must provide **connectivity, routing, security, availability, and controlled access** between on-premises and cloud resources.
