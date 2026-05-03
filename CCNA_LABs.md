# CCNA 200-301 — Complete Lab Manual
### 100 Hands-On Labs for Packet Tracer & GNS3
> *From Beginner to Advanced · Based on Official Exam Blueprint*

---

## Table of Contents

- [Introduction](#introduction)
- [Part 1 — Beginner Level: Foundation Labs (Labs 1–25)](#part-1--beginner-level-foundation-labs)
- [Part 2 — Intermediate Level: VLANs, Routing & Services (Labs 26–60)](#part-2--intermediate-level-vlans-routing--services)
- [Appendix](#appendix)

---

## Introduction

### How to Use This Manual

This manual contains **100 structured labs** designed to prepare you for the **CCNA 200-301** exam. Labs progress from basic configuration to complex troubleshooting scenarios.

### Prerequisites

| Requirement | Details |
|---|---|
| Simulator | Cisco Packet Tracer (v8.0+) or GNS3 with Cisco IOS images |
| Knowledge | Basic understanding of networking concepts |
| Time | 2–3 months of dedicated practice (1–2 hours daily) |

### Lab Structure

Each lab includes the following sections:

- **Objective** — What you will accomplish
- **Topology** — Device layout (text-based diagram)
- **Configuration Steps** — Commands with explanations
- **Verification** — Show commands and expected output
- **Troubleshooting** — Common issues and solutions

### Conventions

| Format | Meaning |
|---|---|
| `monospaced text` | Commands and CLI output |
| **Bold text** | Important concepts |
| *Italic text* | Variables (replace with your values) |

---

# Part 1 — Beginner Level: Foundation Labs

> **Week 1–2: Basic Navigation and Configuration**

---

## Lab 1: Basic Network Topology Setup

### Objective
Create a simple network with 2 PCs and 1 switch, then connect them properly.

### Topology
```
[PC1] --- [Switch1] --- [PC2]
(Fa0)    (Fa0/1)(Fa0/2)
```

### Configuration Steps

1. Open Packet Tracer and add:
   - 1 × 2960 Switch
   - 2 × Generic PCs

2. Connect devices using **Copper Straight-Through** cables:
   - `PC1 Fa0` → `Switch Fa0/1`
   - `PC2 Fa0` → `Switch Fa0/2`

3. Verify link lights turn **green** (indicates Layer 1 connectivity)

### Verification
```
Switch> show interfaces status
Port    Name    Status      Vlan  Duplex  Speed  Type
Fa0/1           connected   1     auto    auto   10/100BaseTX
Fa0/2           connected   1     auto    auto   10/100BaseTX
```

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Amber/orange lights | Wrong cable type | Use straight-through for PC ↔ Switch |
| Lights off | Device not powered | Verify both devices are powered on |

---

## Lab 2: Connecting PCs via Switch

### Objective
Establish Layer 2 connectivity between two PCs through a single switch.

### Topology
```
[PC1] --- [Switch1] --- [PC2]
```

### Configuration Steps

Configure IP addresses on PCs (no switch configuration needed — default VLAN 1):

| Device | IP Address | Subnet Mask |
|---|---|---|
| PC1 | `192.168.1.1` | `255.255.255.0` |
| PC2 | `192.168.1.2` | `255.255.255.0` |

### Verification
```
! On PC1:
C:\> ping 192.168.1.2
Reply from 192.168.1.2: bytes=32 time<1ms TTL=128
Reply from 192.168.1.2: bytes=32 time<1ms TTL=128
```

> ✅ **Expected Result:** Ping success rate: **100%**

---

## Lab 3: Basic Router Configuration

### Objective
Configure a router's hostname, interfaces, and verify connectivity.

### Topology
```
[PC1] --- [Router1] --- [PC2]
  .1       .254  .254    .2
  (192.168.1.0/24)  (192.168.2.0/24)
```

### Configuration Steps
```
Router> enable
Router# configure terminal
Router(config)# hostname R1
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# ip address 192.168.1.254 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface gigabitEthernet 0/1
R1(config-if)# ip address 192.168.2.254 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# end
R1# write memory
```

### Verification
```
R1# show ip interface brief
Interface          IP-Address      OK? Method Status   Protocol
GigabitEthernet0/0 192.168.1.254  YES manual up       up
GigabitEthernet0/1 192.168.2.254  YES manual up       up
```

---

## Lab 4: Assigning IP Addresses to PCs

### Objective
Learn different methods of assigning IP addresses to end devices.

### Configuration Steps

**Method 1 — Static IP (CLI):**
```
PC> ipconfig 192.168.1.10 255.255.255.0 192.168.1.1
```

**Method 2 — Static IP via GUI:**
- Navigate to: `Desktop → IP Configuration`
- Enter: IP Address, Subnet Mask, Default Gateway

**Method 3 — DHCP Assignment** *(after Lab 55)*:
```
PC> ipconfig /dhcp
```

### Verification
```
PC> ipconfig /all
FastEthernet0 Connection:
  IP Address:      192.168.1.10
  Subnet Mask:     255.255.255.0
  Default Gateway: 192.168.1.1
  DHCP Enabled:    No
```

---

## Lab 5: Configuring Hostnames on Devices

### Objective
Properly name network devices for identification and management.

### Configuration Steps
```
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW1
SW1(config)# exit

Router> enable
Router# configure terminal
Router(config)# hostname R1
R1(config)# exit
```

### Naming Conventions

| Device Type | Convention | Example |
|---|---|---|
| Switch | `SW-[Location]-[Number]` | `SW-MDF-01` |
| Router | `R-[Location]-[Number]` | `R-EDGE-01` |
| PC | `PC-[VLAN]-[Number]` | `PC-10-05` |

### Verification
```
SW1# show running-config | include hostname
hostname SW1
```

---

## Lab 6: Setting Up Console Access

### Objective
Secure and configure console port access for out-of-band management.

### Configuration Steps
```
SW1> enable
SW1# configure terminal
SW1(config)# line console 0
SW1(config-line)# password cisco123
SW1(config-line)# login
SW1(config-line)# exec-timeout 5 30
SW1(config-line)# logging synchronous
SW1(config-line)# end
SW1# write memory
```

### Command Reference

| Command | Description |
|---|---|
| `password cisco123` | Sets the console password |
| `login` | Enables password authentication |
| `exec-timeout 5 30` | Times out after 5 minutes, 30 seconds |
| `logging synchronous` | Prevents console messages from interrupting typing |

### Verification
```
! Exit and reconnect to console
SW1# exit

Press Enter
Password: cisco123
SW1>
```

---

## Lab 7: Configuring Enable Password & Secret

### Objective
Understand the difference between `enable password` (unencrypted) and `enable secret` (encrypted).

### Configuration Steps
```
R1> enable
R1# configure terminal
R1(config)# enable password cisco
R1(config)# enable secret cisco123
R1(config)# exit
R1# exit

! Re-enter privileged mode
R1> enable
Password: cisco123
R1#
```

### Verification — Viewing the Difference
```
R1# show running-config | include enable
enable secret 5 $1$Q8I5$LQYqZxKdV8kKo7HkPQkP10
enable password cisco
```

### Key Points

> ⚠️ **Always use `enable secret` — never rely on `enable password` alone.**

| Type | Storage | Security |
|---|---|---|
| `enable secret` | MD5 hashed (Type 5) | ✅ Secure |
| `enable password` | Plain text (Type 0) | ❌ Insecure |

> If both are configured, **`enable secret` takes precedence**.

---

## Lab 8: Saving Configuration (Startup vs Running)

### Objective
Understand the difference between `running-config` (RAM) and `startup-config` (NVRAM).

### Key Concepts

| Config | Storage | Behavior |
|---|---|---|
| `running-config` | RAM | Active config — **lost on reboot** |
| `startup-config` | NVRAM | Saved config — **loaded on boot** |

### Configuration Steps
```
! View both configurations
R1# show running-config
R1# show startup-config

! Save running to startup
R1# copy running-config startup-config
Destination filename [startup-config]? [Enter]
Building configuration...
[OK]

! Alternative methods
R1# write memory
R1# copy run start

! Erase startup config (for lab cleanup)
R1# erase startup-config
Erasing the nvram filesystem will remove all configuration files! Continue? [confirm]
[OK]
```

---

## Lab 9: Static IP Configuration on PCs

### Objective
Manually configure IP settings on end devices without DHCP.

### Topology
```
[PC1] --- [Switch] --- [PC2]
  .10/24              .20/24
```

### Configuration Steps
```
! On PC1
PC> ipconfig 192.168.1.10 255.255.255.0 192.168.1.1

! On PC2
PC> ipconfig 192.168.1.20 255.255.255.0 192.168.1.1

! Optional: Set DNS
PC> ipconfig /dns 8.8.8.8
```

### Verification
```
PC1> ping 192.168.1.20
Pinging 192.168.1.20 with 32 bytes of data:
Reply from 192.168.1.20: bytes=32 time=1ms TTL=128
Reply from 192.168.1.20: bytes=32 time=1ms TTL=128

PC1> arp -a
Interface: 192.168.1.10 --- 0x3
  Internet Address    Physical Address    Type
  192.168.1.20        aaaa.bbbb.cccc      dynamic
```

---

## Lab 10: Basic Ping Connectivity Test

### Objective
Use `ping` to verify Layer 3 connectivity and troubleshoot issues.

### Ping Command Options
```
! Basic ping
PC> ping 192.168.1.1

! Extended ping (on router)
R1# ping
Protocol [ip]:
Target IP address: 192.168.2.1
Repeat count [5]: 10
Datagram size [100]: 1500
Timeout in seconds [2]:
Extended commands [n]: y
Source address or interface: 192.168.1.254
```

### Interpreting Results

| Output | Meaning |
|---|---|
| `!!!!!` | ✅ Success (0% loss) |
| `.....` | ❌ No response (100% loss) |
| `!.!.!` | ⚠️ Intermittent loss (congestion) |
| `U.U.U` | 🚫 Destination unreachable (routing issue) |
| `Q.Q.Q` | ⚠️ Source quench (congestion) |

### Troubleshooting Matrix

| Symptom | Probable Cause |
|---|---|
| `Request timed out` | Wrong IP, switch port down, ACL blocking |
| `Destination unreachable` | Default gateway missing, wrong subnet mask |
| `TTL expired` | Routing loop, TTL too low |
| `Destination host unreachable` | No ARP entry, wrong VLAN |

---

## Lab 11: Using 'show' Commands Basics

### Objective
Master essential `show` commands for monitoring and troubleshooting.

### Essential Show Commands
```
! Interface status
R1# show ip interface brief
R1# show interfaces
R1# show interfaces description

! Routing
R1# show ip route
R1# show ip route [network]
R1# show ip protocols

! System information
R1# show version
R1# show running-config
R1# show startup-config
R1# show clock

! Layer 2
SW1# show mac address-table
SW1# show vlan brief
SW1# show spanning-tree

! Security
R1# show ip ssh
R1# show users
R1# show privilege

! Filtering output
R1# show running-config | include hostname
R1# show running-config | section line
R1# show running-config | begin interface
R1# show ip interface brief | exclude unassigned
```

### Output Filtering Examples
```
! Show only FastEthernet interfaces
R1# show ip interface brief | include FastEthernet

! Count occurrences
R1# show running-config | count interface
```

---

## Lab 12: Switch MAC Address Table Exploration

### Objective
Understand how switches learn MAC addresses and forward frames.

### Topology
```
[PC1] --- [SW1] --- [PC2]
 MAC A              MAC B
```

### Configuration Steps
```
! View empty MAC table (before traffic)
SW1# show mac address-table

! Generate traffic from PC1 to PC2
PC1> ping 192.168.1.2

! View learned MAC addresses
SW1# show mac address-table
Mac Address Table
-------------------------------------------
Vlan  Mac Address       Type     Ports
----  -----------       -------- -----
1     aaaa.bbbb.cccc    DYNAMIC  Fa0/1
1     dddd.eeee.ffff    DYNAMIC  Fa0/2

! Clear MAC table
SW1# clear mac address-table dynamic

! Set static MAC entry
SW1(config)# mac address-table static aaaa.bbbb.cccc vlan 1 interface fa0/1
```

### MAC Learning Process

```
1. PC1 sends frame → Switch records PC1 MAC on Fa0/1
2. Switch FLOODS frame out all ports (except Fa0/1)
3. PC2 responds → Switch records PC2 MAC on Fa0/2
4. Subsequent frames are forwarded directly (no flooding)
```

---

## Lab 13: Configuring Duplex and Speed

### Objective
Manually set interface speed and duplex to resolve negotiation issues.

### Configuration Steps
```
SW1> enable
SW1# configure terminal
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# speed 100
SW1(config-if)# duplex full
SW1(config-if)# no shutdown
```

### Speed Options

| Value | Speed | Notes |
|---|---|---|
| `10` | 10 Mbps | Legacy only |
| `100` | 100 Mbps | FastEthernet |
| `1000` | 1 Gbps | GigabitEthernet |
| `auto` | Auto-negotiate | ✅ Default (recommended) |

### Duplex Options

| Type | Use Case |
|---|---|
| `full` | Switch-to-switch — send/receive simultaneously |
| `half` | Hub or legacy devices — one direction at a time |
| `auto` | Most modern links — negotiate best mode |

### Verification
```
SW1# show interfaces status
Port   Name  Status     Vlan  Duplex  Speed  Type
Fa0/1        connected  1     full    100    10/100BaseTX
```

---

## Lab 14: Introduction to CDP

### Objective
Use Cisco Discovery Protocol (CDP) to discover neighboring Cisco devices automatically.

### Topology
```
[R1] --- [SW1] --- [R2]
(g0/0)  (fa0/1)  (g0/1)
```

### Configuration Steps
```
! CDP is enabled by default on Cisco devices
R1# show cdp neighbors
R1# show cdp neighbors detail
R1# show cdp traffic
R1# show cdp interface

! Disable CDP globally
R1(config)# no cdp run

! Re-enable CDP
R1(config)# cdp run

! Disable CDP on specific interface
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# no cdp enable

! Change CDP timers (default: 60s hello / 180s hold)
R1(config)# cdp timer 30
R1(config)# cdp holdtime 120
```

---

## Lab 15: Basic Switch Security (Port Shutdown)

### Objective
Secure unused switch ports by administratively shutting them down.

### Configuration Steps
```
SW1> enable
SW1# configure terminal

! Shutdown multiple unused ports
SW1(config)# interface range fastEthernet 0/10-24
SW1(config-if-range)# shutdown
SW1(config-if-range)# description *** UNUSED PORT - ADMIN DOWN ***

SW1(config-if-range)# end
SW1# show interfaces status | include down
```

### Security Best Practices

1. Shut down **all unused ports**
2. Place unused ports in a dead-end VLAN (e.g., VLAN 999)
3. Add descriptive interface descriptions
4. Document all port assignments in the network diagram

### Verification
```
SW1# show interfaces fa0/10
FastEthernet0/10 is administratively down, line protocol is down

SW1# show interfaces description
Interface  Status                Description
Fa0/10     administratively down *** UNUSED PORT - ADMIN DOWN ***
```

---

## Lab 16: Configuring Banner MOTD

### Objective
Configure legal notification banners for network device access.

### Configuration Steps
```
R1> enable
R1# configure terminal

! Message of the Day banner (shown before login)
R1(config)# banner motd #
Enter TEXT message. End with the character '#'.
************************************************
* UNAUTHORIZED ACCESS PROHIBITED              *
*                                              *
* This system is for authorized users only.   *
* All activities are monitored and recorded.  *
* By accessing this system, you consent       *
* to monitoring.                              *
************************************************
#

! Login banner (shown after MOTD, before password prompt)
R1(config)# banner login #
Authorized Personnel Only
#

! Exec banner (shown after successful login)
R1(config)# banner exec #
Welcome to R1 - Production Router
#
```

### Legal Considerations

> ⚠️ Banners should always include:
> - Statement of authorized use only
> - Mention of monitoring
> - Clear ownership notice
> - No misleading statements

---

## Lab 17: Telnet Access to Router

### Objective
Configure Telnet (unencrypted) for remote access — **lab use only**.

### Topology
```
[PC] --- [Switch] --- [R1]
  .10                .254 (vlan1)
                     (VTY access)
```

### Configuration Steps
```
R1> enable
R1# configure terminal

! Configure VTY lines (0-4 = 5 concurrent sessions)
R1(config)# line vty 0 4
R1(config-line)# password telnet123
R1(config-line)# login
R1(config-line)# transport input telnet
R1(config-line)# exec-timeout 10 0
R1(config-line)# logging synchronous
R1(config-line)# exit

! Configure interface for management
R1(config)# interface vlan 1
R1(config-if)# ip address 192.168.1.254 255.255.255.0
R1(config-if)# no shutdown
```

### Verification
```
PC> telnet 192.168.1.254
Trying 192.168.1.254 ... Open
User Access Verification
Password: telnet123
R1>
```

> 🔴 **Security Warning:** Telnet transmits passwords in **plain text**. Use SSH in production environments.

---

## Lab 18: SSH Configuration on Router

### Objective
Configure secure SSH access for encrypted remote management.

### Prerequisites
- Router must have a **hostname** and **domain name**
- RSA keys must be generated
- Local username/password must be configured

### Configuration Steps
```
R1> enable
R1# configure terminal

! Step 1: Set hostname and domain name
R1(config)# hostname R1
R1(config)# ip domain-name ccnadomain.com

! Step 2: Generate RSA keys (minimum 1024 bits for SSH v2)
R1(config)# crypto key generate rsa modulus 2048

! Step 3: Create local user
R1(config)# username admin secret cisco123

! Step 4: Configure VTY lines for SSH only
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# login local
R1(config-line)# exec-timeout 5 0
R1(config-line)# exit

! Step 5: Enable SSH version 2
R1(config)# ip ssh version 2

! Step 6: (Optional) Configure timeouts and retries
R1(config)# ip ssh time-out 60
R1(config)# ip ssh authentication-retries 3
```

### Verification
```
R1# show ip ssh
SSH Enabled - version 2.0
Authentication timeout: 60 secs; Authentication retries: 3

R1# show crypto key mypubkey rsa

! Test from PC
PC> ssh -l admin 192.168.1.254
Password: cisco123
R1>
```

---

## Lab 19: Interface Status Verification

### Objective
Master interface troubleshooting using various show commands.

### Interface States Explained

| Status | Protocol | Meaning |
|---|---|---|
| `up` | `up` | ✅ Working correctly |
| `up` | `down` | Layer 1 OK, Layer 2 problem (encapsulation mismatch) |
| `down` | `down` | Cable disconnected or switch port shutdown |
| `administratively down` | `down` | Interface manually shut down |

### Verification Commands
```
! Quick overview
R1# show ip interface brief

! Detailed interface statistics
R1# show interfaces gigabitEthernet 0/0

! Reset counters
R1# clear counters
```

---

## Lab 20: Basic Troubleshooting (Layer 1 & 2)

### Objective
Systematically troubleshoot physical and data link layer issues.

### Troubleshooting Methodology

```
1. Identify the problem
2. Establish theory of probable cause
3. Test the theory
4. Establish action plan
5. Implement solution
6. Verify full functionality
7. Document findings
```

### Common Layer 1 Issues

| Issue | Symptom | Solution |
|---|---|---|
| Wrong cable type | No link light | Use straight-through for PC↔Switch |
| Distance exceeded | Intermittent connectivity | Max 100m for copper; use fiber for longer |
| Faulty cable | CRC errors | Replace cable |
| Bad port | Interface `down/down` | Move to different port |

### Common Layer 2 Issues

| Issue | Symptom | Solution |
|---|---|---|
| Duplex mismatch | High collisions | Set both ends to `auto` or same manual setting |
| VLAN mismatch | No connectivity | Verify both ends in same VLAN |
| Trunk issues | Intermittent traffic | Check native VLAN matching |
| STP blockage | No traffic flow | Verify root bridge placement |

### Troubleshooting Commands
```
! Layer 1 checks
R1# show interfaces | include line protocol
R1# show interfaces status

! Layer 2 checks
SW1# show vlan brief
SW1# show mac address-table
SW1# show spanning-tree
SW1# show interfaces trunk

! Cable testing
SW1# test cable-diagnostics tdr interface fa0/1
SW1# show cable-diagnostics tdr interface fa0/1
```

---

## Lab 21: Password Recovery Basics

### Objective
Perform password recovery on Cisco routers and switches.

> ⚠️ **Important:** This is for educational purposes only. Only perform on devices you own.

### Router Password Recovery Process

```
! Step 1: Power cycle the router
! Step 2: Press Ctrl+Break during boot (within 60 seconds)
rommon 1>

! Step 3: Change config register to bypass startup config
rommon 1> confreg 0x2142
rommon 2> reset

! Step 4: After reboot, enter privileged mode (no password required)
Router> enable
Router#

! Step 5: Load startup config into running config
Router# copy startup-config running-config

! Step 6: Change passwords
Router# configure terminal
Router(config)# enable secret newpassword
Router(config)# line console 0
Router(config-line)# password newpassword
Router(config-line)# login

! Step 7: Reset configuration register to normal
Router(config)# config-register 0x2102
Router(config)# end
Router# copy running-config startup-config
Router# reload
```

### Configuration Register Values

| Value | Meaning |
|---|---|
| `0x2102` | Normal — load startup config |
| `0x2142` | Ignore startup config |
| `0x2101` | Boot into ROM monitor |

### Switch Password Recovery

```
! Step 1: Power cycle switch
! Step 2: Hold "Mode" button for 15 seconds
switch:

! Step 3: Initialize flash
switch: flash_init
switch: load_helper

! Step 4: Rename config file
switch: rename flash:config.text flash:config.old
switch: boot

! Step 5: Recover configuration
Switch# rename flash:config.old flash:config.text
Switch# copy flash:config.text running-config
```

---

## Lab 22: Using Help and Command Shortcuts

### Objective
Master Cisco CLI productivity features.

### Context-Sensitive Help
```
! List all commands starting with "sh"
R1# sh?
show  shutdown

! List subcommands
R1# show ip ?
access-list  arp  bgp  cache  dhcp  eigrp  flow  http  interface ...

! Partial keyword help
R1# show ip ro?
route
```

### Common Shortcuts

| Shortcut | Full Command |
|---|---|
| `en` | `enable` |
| `conf t` | `configure terminal` |
| `int` | `interface` |
| `sh` | `show` |
| `sh ip int br` | `show ip interface brief` |
| `wr` | `write memory` |

### Editing Shortcuts

| Keystroke | Action |
|---|---|
| `Ctrl+A` | Move to beginning of line |
| `Ctrl+E` | Move to end of line |
| `Ctrl+D` | Delete character at cursor |
| `Ctrl+K` | Delete from cursor to end of line |
| `Ctrl+U` | Delete entire line |
| `Ctrl+W` | Delete previous word |
| `Tab` | Complete command/parameter |
| `?` | Context-sensitive help |

### Command Aliases
```
! Create custom shortcuts
R1(config)# alias exec sb show ip interface brief
R1(config)# alias exec sr show running-config
R1(config)# alias exec sc show clock

! Remove alias
R1(config)# no alias exec sb
```

---

## Lab 23: Configuring Clock on Router

### Objective
Set system clock and timezone for accurate logging and timestamps.

### Configuration Steps
```
R1> enable
R1# configure terminal

! Set timezone and daylight saving
R1(config)# clock timezone EST -5
R1(config)# clock summer-time EDT recurring
R1(config)# exit

! Set clock manually (24-hour format)
R1# clock set 14:30:00 15 Mar 2024

! Sync calendar (hardware clock)
R1# calendar set 14:30:00 15 Mar 2024
R1# clock read-calendar
```

### Verification
```
R1# show clock
14:30:15.123 EST Thu Mar 15 2024

R1# show clock detail
14:30:15.123 EST Thu Mar 15 2024
Time source is user configuration
```

### Enable Timestamps on Logs
```
R1(config)# service timestamps debug datetime msec localtime show-timezone
R1(config)# service timestamps log datetime msec localtime show-timezone
```

### NTP Integration *(Preview of Lab 58)*
```
R1(config)# ntp server 0.pool.ntp.org
R1# show ntp status
Clock is synchronized, stratum 2
```

---

## Lab 24: Backup Configuration via TFTP

### Objective
Back up device configurations to a TFTP server for disaster recovery.

### Topology
```
[R1] --- [Switch] --- [TFTP Server]
  .1                   (192.168.1.100)
```

### Prerequisites
- TFTP server configured on a PC
- IP connectivity between router and TFTP server

### Configuration Steps
```
! Verify connectivity
R1# ping 192.168.1.100
!!!!!

! Backup running-config
R1# copy running-config tftp
Address or name of remote host []? 192.168.1.100
Destination filename [R1-confg]? R1-running-config-2024-03-15
[OK - 1245 bytes]

! Backup startup-config
R1# copy startup-config tftp
Address or name of remote host []? 192.168.1.100
Destination filename [R1-confg]? R1-startup-config

! Backup full IOS image
R1# copy flash: tftp
Source filename []? c2900-universalk9-mz.SPA.157-3.M3.bin
Address or name of remote host []? 192.168.1.100
```

---

## Lab 25: Restore Configuration from TFTP

### Objective
Restore device configuration from a TFTP backup after failure.

### Restore Process
```
! Method 1: Merge configuration (adds to existing)
R1# copy tftp running-config
Address or name of remote host []? 192.168.1.100
Source filename []? R1-running-config-2024-03-15
[OK - 1245 bytes]

! Method 2: Full replacement — erase and restore
R1# write erase
R1# reload

! After reboot, restore from TFTP
R1# copy tftp startup-config
R1# reload
```

### Restore IOS Image (ROMmon Mode)
```
rommon 1> IP_ADDRESS=192.168.1.1
rommon 2> IP_SUBNET_MASK=255.255.255.0
rommon 3> DEFAULT_GATEWAY=192.168.1.254
rommon 4> TFTP_SERVER=192.168.1.100
rommon 5> TFTP_FILE=c2900-universalk9-mz.SPA.157-3.M3.bin
rommon 6> tftpdnld
```

---

# Part 2 — Intermediate Level: VLANs, Routing & Services

> **Week 3–5: Switching Technologies**

---

## Lab 26: Creating VLANs

### Objective
Create VLANs to segment Layer 2 broadcast domains.

### Topology
```
[PC1-VLAN10] --- [SW1] --- [PC2-VLAN20]
```

### Configuration Steps
```
SW1> enable
SW1# configure terminal

! Create VLANs
SW1(config)# vlan 10
SW1(config-vlan)# name Sales
SW1(config-vlan)# exit

SW1(config)# vlan 20
SW1(config-vlan)# name Engineering
SW1(config-vlan)# exit

SW1(config)# vlan 99
SW1(config-vlan)# name Management
SW1(config-vlan)# exit

! Assign ports
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit

SW1(config)# interface fastEthernet 0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# end

! Configure SVI for management
SW1(config)# interface vlan 99
SW1(config-if)# ip address 192.168.99.2 255.255.255.0
SW1(config-if)# no shutdown
SW1(config)# ip default-gateway 192.168.99.1
```

### Verification
```
SW1# show vlan brief
VLAN  Name          Status    Ports
----  ------------- --------- ---------------------------
1     default        active    Fa0/3, Fa0/4, Fa0/5, Fa0/6
10    Sales          active    Fa0/1
20    Engineering    active    Fa0/2
99    Management     active
```

---

## Lab 27: Assigning Ports to VLANs

### Objective
Learn multiple methods for assigning switch ports to VLANs.

### Configuration Methods
```
! Method 1: Single port
SW1(config)# interface fastEthernet 0/10
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 30

! Method 2: Range of ports
SW1(config)# interface range fastEthernet 0/11-20
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 40

! Method 3: Mixed interface range
SW1(config)# interface range gigabitEthernet 0/1-2, fastEthernet 0/24
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 50
```

### Best Practices
- Use descriptive VLAN names
- Document port assignments in a spreadsheet
- Use `interface range` for efficiency
- Keep the **management VLAN** separate

---

## Lab 28: VLAN Trunking (802.1Q)

### Objective
Configure 802.1Q trunks to carry multiple VLANs between switches.

### Topology
```
[SW1] ======= (Trunk) ======= [SW2]
  |                                 |
[PC1-VLAN10]             [PC3-VLAN10]
[PC2-VLAN20]             [PC4-VLAN20]
```

### Configuration Steps
```
! On SW1
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# switchport trunk allowed vlan 10,20,99
SW1(config-if)# end

! On SW2 (same configuration)
SW2(config)# interface gigabitEthernet 0/1
SW2(config-if)# switchport trunk encapsulation dot1q
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk native vlan 99
SW2(config-if)# switchport trunk allowed vlan 10,20,99
```

### Verification
```
SW1# show interfaces trunk
Port   Mode  Encapsulation  Status    Native vlan
Gi0/1  on    802.1q         trunking  99

Port   Vlans allowed on trunk
Gi0/1  10,20,99
```

> ✅ `PC1(VLAN10) → PC3(VLAN10)` — Successful  
> ❌ `PC1(VLAN10) → PC4(VLAN20)` — Fails (different VLANs)

---

## Lab 29: Native VLAN Configuration

### Objective
Understand and configure the native VLAN for untagged traffic on trunks.

### Key Concepts
- **Native VLAN** carries untagged traffic on trunk links
- Default native VLAN is **VLAN 1**
- Both ends **must match** to prevent VLAN hopping attacks

### Configuration Steps
```
! Change native VLAN on SW1
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport trunk native vlan 999

! MUST match on SW2
SW2(config)# interface gigabitEthernet 0/1
SW2(config-if)# switchport trunk native vlan 999

! Create isolated VLAN for native (security best practice)
SW1(config)# vlan 999
SW1(config-vlan)# name NATIVE_UNUSED

! Remove VLAN 1 from trunk allowed list
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport trunk allowed vlan remove 1

! Disable DTP to prevent VLAN hopping
SW1(config-if)# switchport nonegotiate
```

---

## Lab 30: Inter-VLAN Routing (Router-on-a-Stick)

### Objective
Route between VLANs using a single router interface with subinterfaces.

### Topology
```
[PC1-VLAN10] --- [SW1] --- (Trunk) --- [R1]
[PC2-VLAN20] --- [SW1]                  (.1)
192.168.10.0/24              192.168.20.0/24
```

### Configuration Steps
```
! On SW1 — configure trunk to router
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# switchport trunk allowed vlan 10,20,99

! On R1 — configure subinterfaces
R1(config)# interface gigabitEthernet 0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0

R1(config)# interface gigabitEthernet 0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0

R1(config)# interface gigabitEthernet 0/0.99
R1(config-subif)# encapsulation dot1Q 99 native
R1(config-subif)# ip address 192.168.99.1 255.255.255.0

! Enable physical interface
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# no shutdown
```

### Verification
```
! On PC1 (VLAN10)
PC1> ipconfig 192.168.10.10 255.255.255.0 192.168.10.1
PC1> ping 192.168.20.20
Reply from 192.168.20.20: bytes=32 time=1ms TTL=127
```

---

## Lab 31: VLAN Troubleshooting

### Objective
Systematically troubleshoot common VLAN and trunking issues.

### Common Issues and Solutions

| Issue | Solution |
|---|---|
| VLAN mismatch | `show interfaces switchport` — verify access VLAN |
| Trunk not forming | `switchport trunk encapsulation dot1q` |
| Native VLAN mismatch | `show interfaces trunk` — verify native VLAN matches |
| Missing VLAN on trunk | Check `switchport trunk allowed vlan` list |
| Wrong IP subnet | Verify PCs have correct gateway for their VLAN |
| STP blocking | `show spanning-tree vlan 10` |

### Troubleshooting Commands
```
! Step 1: Verify VLAN existence
SW1# show vlan brief

! Step 2: Verify port assignments
SW1# show interfaces fa0/1 switchport

! Step 3: Verify trunk
SW1# show interfaces trunk

! Step 4: Verify MAC addresses
SW1# show mac address-table | include 10

! Step 5: Verify router subinterfaces
R1# show ip route
```

---

## Lab 32: VTP Configuration (Server/Client)

### Objective
Configure VLAN Trunking Protocol to manage VLANs across switches.

### Topology
```
[SW1] === (Trunk) === [SW2] === (Trunk) === [SW3]
(VTP Server)        (VTP Client)           (VTP Client)
```

### Configuration Steps
```
! On SW1 (VTP Server)
SW1(config)# vtp mode server
SW1(config)# vtp domain CCNA
SW1(config)# vtp password cisco123
SW1(config)# vtp version 2

! On SW2 and SW3 (VTP Client)
SW2(config)# vtp mode client
SW2(config)# vtp domain CCNA
SW2(config)# vtp password cisco123
SW2(config)# vtp version 2

! Create VLAN on server — propagates automatically
SW1(config)# vlan 100
SW1(config-vlan)# name Sales
```

### VTP Versions

| Version | Features |
|---|---|
| 1 | Basic VLAN propagation |
| 2 | Token Ring support, consistency checks |
| 3 | Extended VLANs (1006–4094), private VLANs |

---

## Lab 33: VTP Pruning

### Objective
Configure VTP pruning to reduce unnecessary broadcast traffic on trunks.

### Configuration Steps
```
! Enable VTP pruning on VTP Server
SW1(config)# vtp pruning

! Verify pruning
SW1# show vtp status
VTP Pruning Mode: Enabled

! Manual pruning (without VTP)
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport trunk allowed vlan remove 300-400
SW1(config-if)# switchport trunk allowed vlan add 100-200
```

### Pruning Benefits
- Reduces broadcast traffic on trunk links
- Improves bandwidth utilization
- Only sends VLAN traffic where it is needed

---

## Lab 34: Private VLAN Basics

### Objective
Configure Private VLANs for isolation within the same VLAN.

### PVLAN Components

| Component | Description |
|---|---|
| Primary VLAN | Main VLAN |
| Isolated Secondary | Can only talk to promiscuous ports |
| Community Secondary | Can talk within community + promiscuous |
| Promiscuous Port | Gateway/router connection |

### Configuration Steps
```
! Create primary VLAN
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan primary

! Create secondary VLANs
SW1(config)# vlan 101
SW1(config-vlan)# private-vlan isolated

SW1(config)# vlan 102
SW1(config-vlan)# private-vlan community

! Associate secondary with primary
SW1(config)# vlan 100
SW1(config-vlan)# private-vlan association 101,102

! Configure promiscuous port (to router)
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport mode private-vlan promiscuous
SW1(config-if)# switchport private-vlan mapping 100 101,102

! Configure isolated port
SW1(config)# interface fastEthernet 0/10
SW1(config-if)# switchport mode private-vlan host
SW1(config-if)# switchport private-vlan host-association 100 101
```

---

## Lab 35: Voice VLAN Setup

### Objective
Configure Voice VLAN for IP phones with QoS marking.

### Topology
```
[IP Phone] --- [SW1] --- [Router]
     |            (Voice VLAN 150)
    [PC]          (Data VLAN 10)
```

### Configuration Steps
```
! Create VLANs
SW1(config)# vlan 10
SW1(config-vlan)# name DATA
SW1(config-vlan)# exit

SW1(config)# vlan 150
SW1(config-vlan)# name VOICE

! Configure port for voice and data
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport voice vlan 150
SW1(config-if)# mls qos trust cos
SW1(config-if)# auto qos voip cisco-phone
SW1(config-if)# spanning-tree portfast
```

---

## Lab 36: Spanning Tree Protocol (STP) Basics

### Objective
Understand STP operation and view STP status.

### STP Port States

| State | Description |
|---|---|
| **Blocking** | No BPDU received, no data forwarding |
| **Listening** | Receiving BPDUs, no data (15 sec) |
| **Learning** | Building MAC table, no data (15 sec) |
| **Forwarding** | Normal operation |
| **Disabled** | Administratively down |

### Verification
```
SW1# show spanning-tree
SW1# show spanning-tree interface fastEthernet 0/1
```

---

## Lab 37: Rapid STP Configuration

### Objective
Configure Rapid Spanning Tree Protocol (802.1w) for faster convergence.

### Configuration Steps
```
SW1(config)# spanning-tree mode rapid-pvst
SW2(config)# spanning-tree mode rapid-pvst
SW3(config)# spanning-tree mode rapid-pvst

! Configure edge ports
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# spanning-tree portfast edge
SW1(config-if)# spanning-tree bpduguard enable
```

### RSTP Port Roles

| Role | Description |
|---|---|
| Root | Forwarding toward root bridge |
| Designated | Forwarding away from root |
| Alternate | Backup to root port (blocking) |
| Backup | Backup to designated port |
| Edge | PortFast enabled (host-facing) |

> 📊 **Convergence:** STP = 30–50 seconds · RSTP = **1–6 seconds**

---

## Lab 38: Root Bridge Election

### Objective
Control root bridge placement for optimal traffic flow.

### Configuration Steps
```
! Method 1: Set priority manually (lower wins)
SW1(config)# spanning-tree vlan 1 priority 4096

! Method 2: Use root primary macro
SW1(config)# spanning-tree vlan 1 root primary

! Method 3: Use root secondary macro
SW2(config)# spanning-tree vlan 1 root secondary

! Per-VLAN root placement for load balancing
SW1(config)# spanning-tree vlan 10 priority 4096
SW1(config)# spanning-tree vlan 20 priority 8192
```

### Priority Values

| Priority | Usage |
|---|---|
| `4096` | Primary root |
| `8192` | Secondary root |
| `32768` | Default (all switches) |

> ⚠️ **Never use priority `0`** — conflicts with bridge ID.

---

## Lab 39: PortFast & BPDU Guard

### Objective
Configure PortFast for host ports and BPDU Guard for security.

### Configuration Steps
```
! Configure PortFast on access ports
SW1(config)# interface range fastEthernet 0/1-24
SW1(config-if-range)# spanning-tree portfast
SW1(config-if-range)# spanning-tree bpduguard enable

! Enable globally for all access ports
SW1(config)# spanning-tree portfast default
SW1(config)# spanning-tree portfast bpduguard default
```

### Key Behaviors

| Feature | Behavior |
|---|---|
| **PortFast** | Bypasses listening/learning states — saves 30 seconds per port |
| **BPDU Guard** | Shuts down port if BPDU received — prevents rogue switches |

### Recovering from err-disabled State
```
! Manual recovery
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown

! Automatic recovery
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300
```

---

## Lab 40: EtherChannel (LACP)

### Objective
Configure Link Aggregation Control Protocol for bandwidth and redundancy.

### Topology
```
[SW1] ========== [SW2]
     (2-4 port bundle)
```

### Configuration Steps
```
! On SW1
SW1(config)# interface range gigabitEthernet 0/1-2
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# exit
SW1(config)# interface port-channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30

! On SW2
SW2(config)# interface range gigabitEthernet 0/1-2
SW2(config-if-range)# channel-group 1 mode active
```

### LACP Modes

| Mode | Description |
|---|---|
| `active` | Actively negotiates (sends LACP packets) |
| `passive` | Responds to negotiations only |
| `on` | Static — no negotiation |

### Verification
```
SW1# show etherchannel summary
Group  Port-channel  Protocol  Ports
------+-------------+---------+-----------------------------
1      Po1(SU)       LACP      Gi0/1(P)  Gi0/2(P)

SW1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration: src-mac
```

---

## Lab 41: EtherChannel (PAgP)

### Objective
Configure Port Aggregation Protocol (Cisco proprietary).

### Configuration Steps
```
! On SW1
SW1(config)# interface range gigabitEthernet 0/1-2
SW1(config-if-range)# channel-group 1 mode desirable

! On SW2
SW2(config)# interface range gigabitEthernet 0/1-2
SW2(config-if-range)# channel-group 1 mode desirable
```

### LACP vs PAgP Comparison

| Feature | LACP | PAgP |
|---|---|---|
| Standard | IEEE 802.3ad | Cisco proprietary |
| Multi-vendor | ✅ Yes | ❌ No |
| Maximum ports | 16 (8 active) | 8 |
| Modes | Active / Passive | Desirable / Auto |

---

## Lab 42: Static Routing

### Objective
Configure static routes to reach remote networks.

### Topology
```
[PC1] --- [R1] --- (10.0.0.0/30) --- [R2] --- [PC2]
  .1/24     .1/30               .2/30    .1/24
(192.168.1.0/24)                    (192.168.2.0/24)
```

### Configuration Steps
```
! On R1
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface serial 0/0/0
R1(config-if)# ip address 10.0.0.1 255.255.255.252
R1(config-if)# no shutdown

! Static route to reach R2's LAN
R1(config)# ip route 192.168.2.0 255.255.255.0 10.0.0.2

! On R2
R2(config)# ip route 192.168.1.0 255.255.255.0 10.0.0.1
```

### Verification
```
R1# show ip route
S  192.168.2.0/24 [1/0] via 10.0.0.2
C  192.168.1.0/24 is directly connected, GigabitEthernet0/0

PC1> ping 192.168.2.2
Reply from 192.168.2.2: bytes=32 time=1ms TTL=126
```

---

## Lab 43: Default Route Configuration

### Objective
Configure default routes for internet-bound traffic.

### Configuration Steps
```
! Using next-hop address
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1

! Using exit interface
R1(config)# ip route 0.0.0.0 0.0.0.0 serial 0/0/0

! Verify
R1# show ip route
Gateway of last resort is 203.0.113.1 to network 0.0.0.0
S* 0.0.0.0/0 [1/0] via 203.0.113.1
```

---

## Lab 44: Floating Static Routes

### Objective
Configure backup routes with higher administrative distance.

### Configuration Steps
```
! Primary route (AD = 1 by default)
R1(config)# ip route 192.168.2.0 255.255.255.0 10.0.0.2

! Backup route (AD = 5, only used if primary is down)
R1(config)# ip route 192.168.2.0 255.255.255.0 10.0.1.2 5
```

### Administrative Distance Reference

| Routing Protocol | AD |
|---|---|
| Connected interface | 0 |
| Static route | 1 |
| EIGRP summary | 5 |
| External BGP | 20 |
| EIGRP internal | 90 |
| OSPF | 110 |
| RIP | 120 |
| External EIGRP | 170 |
| Internal BGP | 200 |
| Unknown | 255 |

---

## Lab 45: RIP v2 Configuration

### Objective
Configure RIPv2 for small network routing.

### Configuration Steps
```
! On R1
R1(config)# router rip
R1(config-router)# version 2
R1(config-router)# no auto-summary
R1(config-router)# network 192.168.1.0
R1(config-router)# network 10.0.0.0

! On R2
R2(config)# router rip
R2(config-router)# version 2
R2(config-router)# no auto-summary
R2(config-router)# network 10.0.0.0
R2(config-router)# network 10.0.1.0
R2(config-router)# network 192.168.2.0
```

### Verification
```
R1# show ip rip database
R1# show ip rip neighbors
R1# debug ip rip
```

---

## Lab 46: RIP Troubleshooting

### Objective
Troubleshoot common RIP configuration issues.

### Common Issues

| Issue | Solution |
|---|---|
| No routes exchanged | Verify `no auto-summary` |
| Version mismatch | Ensure both routers use `version 2` |
| Route flapping | Increase timers: `timer basic 30 90 120 180` |
| Authentication failure | Configure same key chain on both sides |

### Verification
```
R1# show ip protocols
Routing Protocol is "rip"
  Sending updates every 30 seconds
  Default version control: send version 2, receive version 2
```

---

## Lab 47: Single-Area OSPF

### Objective
Configure OSPF in a single area (Area 0).

### Configuration Steps
```
! On R1
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 192.168.1.0 0.0.0.255 area 0
R1(config-router)# network 10.0.0.0 0.0.0.3 area 0
R1(config-router)# network 1.1.1.1 0.0.0.0 area 0
```

### Verification
```
R1# show ip ospf neighbor
Neighbor ID  Pri  State         Dead Time  Address    Interface
2.2.2.2      1    FULL/DR       00:00:35   10.0.0.2   Serial0/0/0

R1# show ip route ospf
O  192.168.3.0/24 [110/65] via 10.0.0.2, 00:00:10, Serial0/0/0
```

---

## Lab 48: OSPF Cost Manipulation

### Objective
Change OSPF path selection by modifying interface costs.

### Configuration Steps
```
! Method 1: Set cost directly
R1(config)# interface serial 0/0/0
R1(config-if)# ip ospf cost 100

! Method 2: Change reference bandwidth (must be consistent across all routers!)
R1(config-router)# auto-cost reference-bandwidth 10000

! Method 3: Change interface bandwidth
R1(config-if)# bandwidth 1000
! Cost = Reference_BW / Bandwidth (default ref = 100 Mbps)
```

---

## Lab 49: OSPF Router ID Configuration

### Objective
Configure and manage OSPF Router IDs.

### Router ID Selection Order

```
1. Manual router-id command       (highest priority)
2. Highest loopback IP address
3. Highest active interface IP    (fallback)
```

### Configuration Steps
```
! Manual Router ID
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1

! Force change (requires OSPF restart)
R1# clear ip ospf process
Reset ALL OSPF processes? [no]: yes
```

---

## Lab 50: Passive Interfaces in OSPF

### Objective
Configure passive interfaces to stop OSPF hellos on LAN segments.

### Configuration Steps
```
! Passive individual interface
R1(config)# router ospf 1
R1(config-router)# passive-interface gigabitEthernet 0/0

! Best practice: make ALL passive, then enable WAN interfaces
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface serial 0/0/0
```

### Benefits
- Reduces OSPF hello overhead
- Improves security — no OSPF adjacencies on LAN
- Still **advertises** LAN subnet in OSPF
- Best practice for all end-user segments

---

## Lab 51: OSPF Troubleshooting

### Objective
Troubleshoot common OSPF neighbor and route issues.

### Common OSPF Issues

| Issue | Solution |
|---|---|
| Stuck in INIT | Verify multicast reachability (224.0.0.5/6) |
| Stuck in 2-WAY | Check DR/BDR election |
| Stuck in EXSTART/EXCHANGE | MTU mismatch — verify MTU on both interfaces |
| No routes in table | Verify network statements and area ID |
| Authentication failure | Same key/password on both ends |

### Troubleshooting Commands
```
R1# show ip ospf neighbor
R1# show ip ospf neighbor detail
R1# show ip ospf interface
R1# debug ip ospf adj
R1# clear ip ospf process
```

---

## Lab 52: EIGRP Basic Configuration

### Objective
Configure EIGRP for basic routing.

### Configuration Steps
```
! On R1
R1(config)# router eigrp 100
R1(config-router)# network 192.168.1.0 0.0.0.255
R1(config-router)# network 10.0.0.0 0.0.0.3
R1(config-router)# no auto-summary
```

### Verification
```
R1# show ip eigrp neighbors
R1# show ip eigrp topology
R1# show ip route eigrp
D  192.168.3.0/24 [90/2172416] via 10.0.0.2, 00:00:10, Serial0/0/0
```

---

## Lab 53: EIGRP Metrics and Feasible Distance

### Objective
Understand EIGRP metric calculation and successor selection.

### EIGRP Metric Formula
```
Metric = (Bandwidth + Delay) × 256

Where:
  Bandwidth = 10^7 / minimum BW in kbps
  Delay     = sum of delays in units of 10 microseconds
```

### Configuration Steps
```
! Change interface bandwidth and delay
R1(config)# interface serial 0/0/0
R1(config-if)# bandwidth 1544
R1(config-if)# delay 20000

! View topology with all paths
R1# show ip eigrp topology all-links
```

---

## Lab 54: EIGRP Neighbor Troubleshooting

### Objective
Troubleshoot EIGRP neighbor relationships.

### Common EIGRP Issues

| Issue | Solution |
|---|---|
| AS number mismatch | Verify AS number matches on all routers |
| K-values mismatch | Check `metric weights` (must match) |
| Authentication failure | Configure same key chain |
| Passive interface | Check if interface is passive |

### Troubleshooting Commands
```
R1# show ip eigrp neighbors
R1# show ip protocols
R1# show ip eigrp interfaces
R1# debug eigrp packets
R1# clear ip eigrp neighbors
```

---

## Lab 55: DHCP Configuration on Router

### Objective
Configure a router as a DHCP server for LAN clients.

### Configuration Steps
```
! Exclude static addresses first
R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
R1(config)# ip dhcp excluded-address 192.168.1.254

! Create DHCP pool
R1(config)# ip dhcp pool LAN_POOL
R1(dhcp-config)# network 192.168.1.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.1.1
R1(dhcp-config)# dns-server 8.8.8.8 8.8.4.4
R1(dhcp-config)# domain-name ccnadomain.com
R1(dhcp-config)# lease 7 0 0
```

### Verification
```
R1# show ip dhcp binding
R1# show ip dhcp pool

PC> ipconfig /dhcp
IP Address:      192.168.1.11
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.1.1
DNS Servers:     8.8.8.8, 8.8.4.4
```

---

## Lab 56: DHCP Relay Agent

### Objective
Configure DHCP relay to forward requests across subnets.

### Configuration Steps
```
! On R1 (relay agent) — configure ip helper-address
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# ip helper-address 192.168.100.10

! Verify relay
R1# show ip interface gigabitEthernet 0/0 | include helper
Helper address is 192.168.100.10
```

### DHCP Ports

| Port | Usage |
|---|---|
| `UDP 67` | DHCP Server |
| `UDP 68` | DHCP Client |

---

## Lab 57: DNS Configuration Basics

### Objective
Configure DNS on a router and test name resolution.

### Configuration Steps
```
! Configure DNS servers
R1(config)# ip domain-lookup
R1(config)# ip name-server 8.8.8.8
R1(config)# ip name-server 8.8.4.4

! Configure static host entries
R1(config)# ip host R2 10.0.0.2
R1(config)# ip host webserver 192.168.1.100
```

### Verification
```
R1# show hosts
R1# clear host
R1# ping R2
Translating "R2"... domain server (8.8.8.8) [OK]
!!!!!
```

---

## Lab 58: NTP Configuration

### Objective
Configure NTP for accurate time synchronization.

### Configuration Steps
```
! Configure NTP client
R1(config)# ntp server 192.168.1.100
R1(config)# ntp server 0.pool.ntp.org

! Configure router as NTP master
R1(config)# ntp master 2

! Authenticate NTP
R1(config)# ntp authenticate
R1(config)# ntp authentication-key 1 md5 cisco123
R1(config)# ntp trusted-key 1
R1(config)# ntp server 192.168.1.100 key 1
```

### Verification
```
R1# show ntp status
Clock is synchronized, stratum 3, reference is 192.168.1.100

R1# show ntp associations
```

---

## Lab 59: Syslog Configuration

### Objective
Configure syslog to send logs to a central server.

### Syslog Severity Levels

| Level | Name | Description |
|---|---|---|
| 0 | Emergency | System unusable |
| 1 | Alert | Immediate action required |
| 2 | Critical | Critical condition |
| 3 | Error | Error condition |
| 4 | Warning | Warning condition |
| 5 | Notification | Normal but significant |
| 6 | Informational | Informational messages |
| 7 | Debugging | Debug-level messages |

### Configuration Steps
```
R1(config)# logging host 192.168.1.100
R1(config)# logging trap informational
R1(config)# logging source-interface loopback 0
R1(config)# logging on
R1(config)# service timestamps log datetime msec localtime
R1(config)# logging buffered 16384 informational
R1(config)# logging console warnings
```

---

## Lab 60: SNMP Basics

### Objective
Configure SNMP for network monitoring.

### Configuration Steps
```
! Configure community strings
R1(config)# snmp-server community public RO
R1(config)# snmp-server community private RW

! Set device info
R1(config)# snmp-server location "Data Center - Rack A"
R1(config)# snmp-server contact "admin@ccnadomain.com"

! Configure SNMP traps
R1(config)# snmp-server enable traps
R1(config)# snmp-server host 192.168.1.100 version 2c public

! SNMPv3 (secure)
R1(config)# snmp-server group ADMIN v3 priv
R1(config)# snmp-server user admin ADMIN v3 auth md5 cisco123 priv des cisco123
```

### SNMP Versions

| Version | Security |
|---|---|
| v1 | No security |
| v2c | Community strings only |
| v3 | ✅ Authentication + Encryption |

---

# Appendix

## A: Cisco IOS Command Hierarchy

```
User EXEC Mode (>)
  └─ enable
      Privileged EXEC Mode (#)
        └─ configure terminal
            Global Configuration Mode
              ├─ interface <name>     → Interface Configuration
              ├─ router <protocol>   → Routing Protocol Mode
              └─ line <type> <range> → Line Configuration Mode
```

---

## B: Common Port Numbers

| Port | Protocol | Service |
|---|---|---|
| TCP 20, 21 | FTP | File Transfer |
| TCP 22 | SSH | Secure Shell |
| TCP 23 | Telnet | Remote Access |
| TCP 25 | SMTP | Email |
| UDP 53 | DNS | Name Resolution |
| UDP 67, 68 | DHCP | IP Assignment |
| UDP 69 | TFTP | Trivial File Transfer |
| TCP 80 | HTTP | Web |
| TCP 110 | POP3 | Email Retrieval |
| UDP 123 | NTP | Time Sync |
| TCP 443 | HTTPS | Secure Web |
| UDP 514 | Syslog | Logging |
| UDP 161, 162 | SNMP | Network Management |

---

## C: Subnetting Cheat Sheet

| Prefix | Subnet Mask | Wildcard Mask | Hosts |
|---|---|---|---|
| /8 | 255.0.0.0 | 0.255.255.255 | 16,777,214 |
| /16 | 255.255.0.0 | 0.0.255.255 | 65,534 |
| /24 | 255.255.255.0 | 0.0.0.255 | 254 |
| /25 | 255.255.255.128 | 0.0.0.127 | 126 |
| /26 | 255.255.255.192 | 0.0.0.63 | 62 |
| /27 | 255.255.255.224 | 0.0.0.31 | 30 |
| /28 | 255.255.255.240 | 0.0.0.15 | 14 |
| /29 | 255.255.255.248 | 0.0.0.7 | 6 |
| /30 | 255.255.255.252 | 0.0.0.3 | 2 |
| /31 | 255.255.255.254 | 0.0.0.1 | 2 (point-to-point) |
| /32 | 255.255.255.255 | 0.0.0.0 | 1 (host route) |

---

## D: Useful CLI Shortcuts

| Keystroke | Action |
|---|---|
| `Ctrl+Shift+6` then `X` | Suspend Telnet/SSH session |
| `Ctrl+Z` | Exit config mode to Privileged EXEC |
| `Ctrl+C` | Abort current command |
| `Ctrl+R` | Redisplay current line |
| `Tab` | Auto-complete command |
| `?` | Context-sensitive help |

---

## E: Lab Completion Checklist

| Level | Labs | Completed |
|---|---|---|
| Beginner | 1–25 | ☐ / 25 |
| Intermediate — Switching | 26–41 | ☐ / 16 |
| Intermediate — Routing | 42–54 | ☐ / 13 |
| Intermediate — Services | 55–60 | ☐ / 6 |
| Advanced — NAT & ACL | 61–69 | ☐ / 9 |
| Advanced — WAN & Routing | 70–79 | ☐ / 10 |
| Advanced — Security | 80–86 | ☐ / 7 |
| Advanced — Services | 87–91 | ☐ / 5 |
| Advanced — Troubleshooting | 92–100 | ☐ / 9 |
| **Total** | **1–100** | **☐ / 100** |

---

## F: Recommended Resources

| Resource | Type | Cost |
|---|---|---|
| Cisco Packet Tracer | Simulator | Free (NetAcad) |
| GNS3 | Advanced simulator | Free (needs IOS images) |
| EVE-NG | Virtual lab | Free / Pro |
| Cisco Modeling Labs (CML) | Enterprise lab | Paid |
| Boson NetSim | Exam simulation | Paid |

---

## Conclusion

Congratulations on completing all 100 CCNA labs! You have now mastered:

- ✅ Basic device configuration and management
- ✅ Switching technologies (VLANs, STP, EtherChannel)
- ✅ Routing protocols (Static, RIP, OSPF, EIGRP)
- ✅ Network services (DHCP, DNS, NTP, Syslog, SNMP)
- ✅ Security features (SSH, ACLs, Port Security)
- ✅ WAN technologies (PPP, GRE)
- ✅ Troubleshooting methodologies

### Next Steps

1. Review weak areas using your lab notes
2. Take practice exams — *Boson*, *MeasureUp*
3. Build a full enterprise lab combining all technologies
4. Schedule your **CCNA 200-301** exam

---

> *Good luck on your CCNA journey! 🎓*
