# CCNA 200-301 Complete Lab Manual
> 100 Hands-On Labs for Packet Tracer & GNS3 — From Beginner to Advanced

---

## Overview

This lab manual is designed for CCNA 200-301 candidates seeking hands-on practice aligned with the official Cisco exam blueprint. It covers 100 structured labs progressing from basic device configuration to complex troubleshooting scenarios.

---

## Prerequisites

- **Cisco Packet Tracer** v8.0+ or **GNS3** with Cisco IOS images
- Basic understanding of networking concepts
- Recommended study time: **2–3 months** (1–2 hours daily)

---

## Lab Structure

Each lab follows a consistent format:

| Section | Description |
|---|---|
| **Objective** | What you will accomplish |
| **Topology** | Text-based device layout diagram |
| **Configuration Steps** | Commands with explanations |
| **Verification** | Show commands and expected output |
| **Troubleshooting** | Common issues and solutions |

---

## Lab Index

### 🟢 Beginner Level — Foundation Labs (Labs 1–25)
*Weeks 1–2: Basic Navigation and Configuration*

| Lab | Topic |
|---|---|
| 1 | Basic Network Topology Setup |
| 2 | Connecting PCs via Switch |
| 3 | Basic Router Configuration |
| 4 | Assigning IP Addresses to PCs |
| 5 | Configuring Hostnames on Devices |
| 6 | Setting Up Console Access |
| 7 | Configuring Enable Password & Secret |
| 8 | Saving Configuration (Startup vs Running) |
| 9 | Static IP Configuration on PCs |
| 10 | Basic Ping Connectivity Test |
| 11 | Using `show` Commands Basics |
| 12 | Switch MAC Address Table Exploration |
| 13 | Configuring Duplex and Speed |
| 14 | Introduction to CDP |
| 15 | Basic Switch Security (Port Shutdown) |
| 16 | Configuring Banner MOTD |
| 17 | Telnet Access to Router |
| 18 | SSH Configuration on Router |
| 19 | Interface Status Verification |
| 20 | Basic Troubleshooting (Layer 1 & 2) |
| 21 | Password Recovery Basics |
| 22 | Using Help and Command Shortcuts |
| 23 | Configuring Clock on Router |
| 24 | Backup Configuration via TFTP |
| 25 | Restore Configuration from TFTP |

---

### 🟡 Intermediate Level — VLANs, Routing & Services (Labs 26–60)
*Weeks 3–5: Switching Technologies & Routing Protocols*

| Lab | Topic |
|---|---|
| 26–35 | VLANs (Creating, Assigning, Trunking, Native VLAN, Inter-VLAN Routing, VTP, Private VLANs, Voice VLAN) |
| 36–41 | Spanning Tree (STP, RSTP, Root Bridge, PortFast, BPDU Guard, EtherChannel LACP/PAgP) |
| 42–44 | Static Routing (Static Routes, Default Route, Floating Static Routes) |
| 45–46 | RIPv2 (Configuration & Troubleshooting) |
| 47–51 | OSPF (Single-Area, Cost Manipulation, Router ID, Passive Interfaces, Troubleshooting) |
| 52–54 | EIGRP (Basic Config, Metrics & FD, Neighbor Troubleshooting) |
| 55–60 | Network Services (DHCP, DHCP Relay, DNS, NTP, Syslog, SNMP) |

---

### 🔴 Advanced Level (Labs 61–100)
*Coming in later sections of the manual*

| Range | Topic Area |
|---|---|
| 61–69 | NAT & ACLs |
| 70–79 | WAN & Advanced Routing |
| 80–86 | Security |
| 87–91 | Advanced Services |
| 92–100 | Complex Troubleshooting |

---

## Quick Reference

### Cisco IOS Mode Hierarchy
```
> User EXEC          — Limited monitoring
# Privileged EXEC    — Full device access
(config)#            — Global Configuration
(config-if)#         — Interface Configuration
(config-router)#     — Routing Protocol Configuration
(config-line)#       — Console/VTY Line Configuration
```

### Common Port Numbers
| Port | Service |
|---|---|
| TCP 22 | SSH |
| TCP 23 | Telnet |
| UDP 53 | DNS |
| UDP 67/68 | DHCP |
| UDP 69 | TFTP |
| TCP 80 / 443 | HTTP / HTTPS |
| UDP 123 | NTP |
| UDP 161/162 | SNMP |
| UDP 514 | Syslog |

### Subnetting Cheat Sheet
| Prefix | Subnet Mask | Wildcard Mask |
|---|---|---|
| /24 | 255.255.255.0 | 0.0.0.255 |
| /25 | 255.255.255.128 | 0.0.0.127 |
| /26 | 255.255.255.192 | 0.0.0.63 |
| /27 | 255.255.255.224 | 0.0.0.31 |
| /28 | 255.255.255.240 | 0.0.0.15 |
| /29 | 255.255.255.248 | 0.0.0.7 |
| /30 | 255.255.255.252 | 0.0.0.3 |

### Administrative Distance Reference
| Protocol | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| EIGRP Internal | 90 |
| OSPF | 110 |
| RIP | 120 |
| External EIGRP | 170 |

---

## Progress Tracker

Use this checklist to track your lab completion:

- [ ] Beginner (Labs 1–25) — `__/25`
- [ ] Intermediate Switching (Labs 26–41) — `__/16`
- [ ] Intermediate Routing (Labs 42–54) — `__/13`
- [ ] Intermediate Services (Labs 55–60) — `__/6`
- [ ] Advanced NAT & ACL (Labs 61–69) — `__/9`
- [ ] Advanced WAN & Routing (Labs 70–79) — `__/10`
- [ ] Advanced Security (Labs 80–86) — `__/7`
- [ ] Advanced Services (Labs 87–91) — `__/5`
- [ ] Advanced Troubleshooting (Labs 92–100) — `__/9`

---

## Recommended Tools

| Tool | Type | Notes |
|---|---|---|
| Cisco Packet Tracer | Free (NetAcad) | Best for beginners |
| GNS3 | Free | Requires IOS images |
| EVE-NG | Free/Pro | Professional virtual lab |
| Cisco Modeling Labs (CML) | Paid | Official Cisco platform |
| Boson NetSim | Paid | Exam simulation |

---

## Conventions

- `monospaced text` — Commands and CLI output
- **Bold** — Important concepts
- *Italic* — Variables (replace with your values)

---

## Next Steps After Completing All Labs

1. Review weak areas using your lab notes
2. Take practice exams (Boson, MeasureUp)
3. Build a full enterprise lab combining all technologies
4. Schedule your **CCNA 200-301** exam

---

*Good luck on your CCNA journey!*
