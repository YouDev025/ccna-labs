# Lab Application Services

## Objective

This lab introduces the deployment and configuration of common network application services within an enterprise environment. The objective is to configure and verify services such as **DHCP, DNS, HTTP, FTP, Email, and NTP**, ensuring clients can automatically obtain network settings and access shared resources.

---

## Network Topology

### Devices

- 2 Cisco 2911 Routers
- 2 Cisco 2960 Switches
- 1 Server (Application Server)
- 4 PCs
- Copper Straight-Through Cables

---

### Topology Overview

```
                 Headquarters

           PC0             PC1
            |               |
        +-----------------------+
        |      Switch0          |
        +-----------------------+
                 |
              Router0
                 |
        =====================
             WAN Network
        =====================
                 |
              Router1
                 |
        +-----------------------+
        |      Switch1          |
        +-----------------------+
           |               |
         PC2             PC3
           |
     Application Server

             Branch Office
```

---

## IP Addressing

### Headquarters LAN

| Device | Interface | IP Address |
|---------|-----------|------------|
| Router0 | G0/0 | 192.168.10.1/24 |
| PC0 | DHCP | Automatic |
| PC1 | DHCP | Automatic |

---

### Branch LAN

| Device | Interface | IP Address |
|---------|-----------|------------|
| Router1 | G0/0 | 192.168.20.1/24 |
| PC2 | DHCP | Automatic |
| PC3 | DHCP | Automatic |
| Server | Fa0 | 192.168.20.100/24 |

---

### WAN

| Link | Network |
|------|---------|
| Router0 ↔ Router1 | 10.0.0.0/30 |

---

## Services Configured

### DHCP

Automatically assigns:

- IP Address
- Subnet Mask
- Default Gateway
- DNS Server

Example DHCP Pools

| Pool | Network |
|------|----------|
| HQ | 192.168.10.0/24 |
| Branch | 192.168.20.0/24 |

---

### DNS

Provides hostname resolution.

Example Records

| Hostname | IP Address |
|-----------|------------|
| www.company.local | 192.168.20.100 |
| ftp.company.local | 192.168.20.100 |
| mail.company.local | 192.168.20.100 |

---

### HTTP

Hosts a company web page.

Example URL

```
http://www.company.local
```

---

### FTP

Provides centralized file sharing.

Example credentials

```
Username: admin
Password: cisco
```

---

### Email

Configure:

- SMTP
- POP3

Example accounts

| User | Email |
|------|--------|
| user1 | user1@company.local |
| user2 | user2@company.local |

---

### NTP

Synchronizes network device clocks.

Example Server

```
192.168.20.100
```

---

## Tasks

- Build the network topology.
- Configure IP addressing.
- Configure static routing between sites.
- Configure DHCP services.
- Configure DNS services.
- Configure HTTP services.
- Configure FTP services.
- Configure Email services.
- Configure NTP synchronization.
- Verify connectivity from all clients.

---

## Routing

Static routing is used between Headquarters and Branch Office.

Example

```bash
ip route 192.168.20.0 255.255.255.0 10.0.0.2
```

```bash
ip route 192.168.10.0 255.255.255.0 10.0.0.1
```

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

### DHCP Bindings

```bash
show ip dhcp binding
```

### DHCP Pool

```bash
show ip dhcp pool
```

### DNS Test

```bash
ping www.company.local
```

### HTTP Test

Open the Web Browser on a PC and browse to:

```
http://www.company.local
```

### FTP Test

```bash
ftp 192.168.20.100
```

### Email Test

Use the Email application in Packet Tracer to:

- Send an email.
- Receive an email.

### NTP Status

```bash
show clock
```

```bash
show ntp associations
```

---

## Expected Results

- All interfaces are operational.
- PCs obtain IP addresses automatically using DHCP.
- DNS successfully resolves hostnames.
- The HTTP web page is reachable from all clients.
- FTP users can authenticate and transfer files.
- Email messages are successfully exchanged.
- NTP synchronizes the device clocks.
- End-to-end connectivity is verified across the WAN.

---

## Skills Practiced

- IP Addressing
- Static Routing
- DHCP Configuration
- DNS Configuration
- HTTP Server Configuration
- FTP Server Configuration
- Email Services
- NTP Configuration
- Network Verification
- Cisco IOS Troubleshooting

---

## Key Concepts

- DHCP
- DNS
- HTTP
- HTTPS
- FTP
- SMTP
- POP3
- IMAP
- NTP
- Client-Server Architecture
- Application Layer Services
- Enterprise Network Services

---

## Conclusion

This lab demonstrates the deployment of common enterprise application services used in modern networks. By configuring DHCP, DNS, HTTP, FTP, Email, and NTP, network administrators can provide automated addressing, name resolution, file sharing, web access, messaging, and time synchronization. These services form the foundation of reliable and scalable enterprise network infrastructures.