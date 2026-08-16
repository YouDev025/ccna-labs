# Lab RADIUS and TACACS+

## 1. Objective

This lab introduces **AAA (Authentication, Authorization, and Accounting)** using **RADIUS** and **TACACS+** in an enterprise network.

At the end of this lab, you should be able to:

* Build a basic AAA network topology.
* Configure a RADIUS server.
* Configure a TACACS+ server.
* Configure a Cisco router/switch as an AAA client.
* Authenticate network administrators using a remote AAA server.
* Understand the difference between RADIUS and TACACS+.
* Verify AAA configuration and authentication.
* Troubleshoot common AAA problems.

---

# 2. Network Topology

```text
                         ┌──────────────────┐
                         │    RADIUS Server │
                         │   192.168.20.10  │
                         └────────┬─────────┘
                                  │
                                  │
                         ┌────────┴─────────┐
                         │                  │
                         │      SW1         │
                         │   L2 Switch      │
                         │                  │
                         └────────┬─────────┘
                                  │
                                  │
                         ┌────────┴─────────┐
                         │       R1         │
                         │   AAA Client     │
                         │ 192.168.10.1     │
                         └────────┬─────────┘
                                  │
                         ┌────────┴─────────┐
                         │ TACACS+ Server   │
                         │ 192.168.20.20    │
                         └──────────────────┘
```

For a Packet Tracer implementation, the exact server capabilities may depend on the Packet Tracer version. The lab can therefore be implemented using the **AAA service available on the Server device**, while the IOS configuration demonstrates the RADIUS/TACACS+ concepts.

---

# 3. Devices

| Device         | Quantity | Role                     |
| -------------- | -------: | ------------------------ |
| Router R1      |        1 | AAA Client               |
| Switch SW1     |        1 | Network Access           |
| RADIUS Server  |        1 | Authentication Server    |
| TACACS+ Server |        1 | Authentication Server    |
| PC             |        1 | Administration / Testing |

---

# 4. Addressing Plan

| Device         | Interface | IP Address     | Mask          |
| -------------- | --------- | -------------- | ------------- |
| R1             | G0/0      | 192.168.10.1   | 255.255.255.0 |
| SW1            | VLAN 1    | 192.168.10.2   | 255.255.255.0 |
| RADIUS Server  | NIC       | 192.168.20.10  | 255.255.255.0 |
| TACACS+ Server | NIC       | 192.168.20.20  | 255.255.255.0 |
| PC             | NIC       | 192.168.10.100 | 255.255.255.0 |

Gateway:

```text
R1 → 192.168.10.1
```

---

# 5. AAA Concept

AAA means:

```text
Authentication
      +
Authorization
      +
Accounting
```

### Authentication

Answers:

> Who are you?

Example:

```text
Username: admin
Password: ********
```

### Authorization

Answers:

> What are you allowed to do?

For example:

```text
User → show commands only
Administrator → configuration commands
```

### Accounting

Answers:

> What did the user do?

For example:

```text
User login
Command executed
Session duration
Logout
```

---

# 6. RADIUS

**RADIUS** stands for:

> Remote Authentication Dial-In User Service

RADIUS is commonly used for:

* Network access authentication.
* Wi-Fi authentication.
* VPN authentication.
* 802.1X.
* Enterprise user authentication.

Common RADIUS ports:

```text
UDP 1812 → Authentication
UDP 1813 → Accounting
```

---

# 7. TACACS+

**TACACS+** is commonly used for network-device administration.

It is useful for:

* Router administration.
* Switch administration.
* Network administrator authentication.
* Command authorization.
* Accounting.

TACACS+ uses:

```text
TCP 49
```

A major advantage is that TACACS+ separates:

```text
Authentication
Authorization
Accounting
```

This provides more granular control over administrative access.

---

# 8. RADIUS vs TACACS+

| Feature                     | RADIUS         | TACACS+               |
| --------------------------- | -------------- | --------------------- |
| Transport                   | UDP            | TCP                   |
| Authentication port         | UDP 1812       | TCP 49                |
| Accounting                  | UDP 1813       | TCP 49                |
| Common use                  | Network access | Device administration |
| AAA separation              | Less granular  | More granular         |
| Command authorization       | Limited        | Strong                |
| Cisco device administration | Possible       | Very common           |

### Simple rule

```text
RADIUS  → Network Access
TACACS+ → Network Device Administration
```

---

# 9. Step 1 — Configure R1 IP Addressing

On R1:

```cisco
enable
configure terminal

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
GigabitEthernet0/0
192.168.10.1
up
up
```

---

# 10. Step 2 — Configure SW1

Create the management IP:

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

Verify:

```cisco
show ip interface brief
```

---

# 11. Step 3 — Configure the RADIUS Server

Configure the server:

```text
IP Address:       192.168.20.10
Subnet Mask:      255.255.255.0
Gateway:          192.168.20.1
```

Enable the RADIUS service if supported by the Packet Tracer version.

Create a user:

```text
Username: admin
Password: Admin@123
```

Configure the network device as an AAA client.

Example:

```text
Client Name: R1
Client IP:   192.168.10.1
Shared Key:  radiuskey
```

The **shared secret** must be identical on the router and server.

---

# 12. Step 4 — Configure RADIUS on R1

Configure the RADIUS server:

```cisco
configure terminal

radius server RAD1
 address ipv4 192.168.20.10 auth-port 1812 acct-port 1813
 key radiuskey
exit
```

Create an AAA method list:

```cisco
aaa new-model
```

Configure login authentication:

```cisco
aaa authentication login default group radius local
```

This means:

```text
1. Try RADIUS
2. If RADIUS is unavailable, use local authentication
```

Create a local backup user:

```cisco
username backup privilege 15 secret Backup@123
```

---

# 13. Step 5 — Configure VTY Access

Configure remote administration:

```cisco
line vty 0 4
 login authentication default
 transport input ssh
exit
```

This allows SSH users to authenticate through the configured AAA method.

---

# 14. Step 6 — Configure SSH

Set the hostname:

```cisco
hostname R1
```

Configure a domain name:

```cisco
ip domain-name lab.local
```

Generate RSA keys:

```cisco
crypto key generate rsa modulus 1024
```

Enable SSH:

```cisco
ip ssh version 2
```

Verify:

```cisco
show ip ssh
```

---

# 15. Step 7 — Test RADIUS Authentication

From the administration PC:

```text
PC → Command Prompt
```

Test connectivity:

```bash
ping 192.168.10.1
```

Then establish an SSH connection:

```bash
ssh -l admin 192.168.10.1
```

Enter:

```text
Username: admin
Password: Admin@123
```

Expected result:

```text
Authentication successful
```

---

# 16. Step 8 — Configure TACACS+

Configure the TACACS+ server:

```text
IP Address:       192.168.20.20
Subnet Mask:      255.255.255.0
Gateway:          192.168.20.1
```

Enable TACACS+ if supported by the server.

Create:

```text
Username: netadmin
Password: NetAdmin@123
```

Configure the network device:

```text
Client: R1
Client IP: 192.168.10.1
Shared Key: tacacskey
```

The shared secret must match the router configuration.

---

# 17. Step 9 — Configure TACACS+ on R1

On R1:

```cisco
configure terminal

tacacs server TAC1
 address ipv4 192.168.20.20
 key tacacskey
exit
```

Enable AAA:

```cisco
aaa new-model
```

Configure authentication:

```cisco
aaa authentication login TACACS group tacacs+ local
```

Configure VTY:

```cisco
line vty 0 4
 login authentication TACACS
 transport input ssh
exit
```

---

# 18. Step 10 — Configure TACACS+ Authorization

One advantage of TACACS+ is command authorization.

Example:

```cisco
aaa authorization exec default group tacacs+ local
```

You can also configure command authorization:

```cisco
aaa authorization commands 15 default group tacacs+ local
```

This allows the TACACS+ server to participate in deciding whether a user can execute privileged commands.

---

# 19. Step 11 — Configure TACACS+ Accounting

Configure accounting:

```cisco
aaa accounting exec default start-stop group tacacs+
```

For commands:

```cisco
aaa accounting commands 15 default start-stop group tacacs+
```

This allows the AAA server to record administrative activity.

---

# 20. Step 12 — Verify AAA Configuration

Display the AAA configuration:

```cisco
show running-config | section aaa
```

You should find entries similar to:

```text
aaa new-model
aaa authentication login default group radius local
aaa authentication login TACACS group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
```

---

# 21. Verify RADIUS

Check:

```cisco
show running-config | section radius
```

Expected:

```text
radius server RAD1
 address ipv4 192.168.20.10 auth-port 1812 acct-port 1813
 key radiuskey
```

---

# 22. Verify TACACS+

Use:

```cisco
show running-config | section tacacs
```

Expected:

```text
tacacs server TAC1
 address ipv4 192.168.20.20
 key tacacskey
```

---

# 23. Test Connectivity

From R1:

```cisco
ping 192.168.20.10
```

Test TACACS+ server:

```cisco
ping 192.168.20.20
```

Expected:

```text
!!!!!
```

This confirms IP connectivity.

---

# 24. Debug RADIUS

For troubleshooting:

```cisco
debug radius
```

Then attempt authentication.

After testing, disable debugging:

```cisco
undebug all
```

or:

```cisco
no debug all
```

---

# 25. Debug TACACS+

Use:

```cisco
debug tacacs
```

Then test authentication.

After testing:

```cisco
undebug all
```

**Important:** Avoid leaving debugging enabled on production devices because it can generate a large amount of output.

---

# 26. Common Problems

## Problem 1 — Authentication fails

Check:

```cisco
show running-config | section aaa
```

Verify:

* Username.
* Password.
* Server IP.
* Shared secret.
* AAA method list.

---

## Problem 2 — RADIUS server is unreachable

Test:

```cisco
ping 192.168.20.10
```

If the ping fails, check:

```cisco
show ip interface brief
```

Check the routing configuration.

---

## Problem 3 — TACACS+ authentication fails

Check:

```cisco
ping 192.168.20.20
```

Then verify:

```cisco
show running-config | section tacacs
```

The shared key must be identical:

```text
R1:
tacacskey

Server:
tacacskey
```

---

## Problem 4 — SSH does not work

Check:

```cisco
show ip ssh
```

Verify:

```cisco
show running-config | section line vty
```

You should have:

```cisco
line vty 0 4
 login authentication TACACS
 transport input ssh
```

---

# 27. Security Considerations

Do not use simple passwords in a production network.

For a real enterprise deployment:

* Use strong passwords.
* Use SSH instead of Telnet.
* Use TACACS+ for device administration when appropriate.
* Use RADIUS for network access.
* Protect AAA shared secrets.
* Use a local emergency account.
* Restrict management access with ACLs.
* Monitor AAA accounting logs.
* Synchronize device time using NTP.
* Avoid leaving debugging enabled.

---

# 28. Verification Checklist

* [ ] R1 has the correct IP address.
* [ ] SW1 has a management IP.
* [ ] RADIUS server is reachable.
* [ ] TACACS+ server is reachable.
* [ ] `aaa new-model` is enabled.
* [ ] RADIUS server is configured.
* [ ] TACACS+ server is configured.
* [ ] Shared secrets match.
* [ ] SSH is configured.
* [ ] VTY lines use AAA authentication.
* [ ] RADIUS authentication works.
* [ ] TACACS+ authentication works.
* [ ] Local authentication is available as a backup.
* [ ] TACACS+ authorization is configured.
* [ ] TACACS+ accounting is configured.
* [ ] Configuration is saved.

---

# 29. Important Commands Summary

### AAA

```cisco
show running-config | section aaa
```

### RADIUS

```cisco
show running-config | section radius
```

### TACACS+

```cisco
show running-config | section tacacs
```

### Interfaces

```cisco
show ip interface brief
```

### SSH

```cisco
show ip ssh
```

### Connectivity

```cisco
ping 192.168.20.10
ping 192.168.20.20
```

### RADIUS Debug

```cisco
debug radius
```

### TACACS+ Debug

```cisco
debug tacacs
```

### Disable Debugging

```cisco
undebug all
```

---

# 30. Lab Questions

### Question 1

What does AAA mean?

### Question 2

What is the main purpose of RADIUS?

### Question 3

What is the main purpose of TACACS+?

### Question 4

Which protocol uses UDP 1812 for authentication?

### Question 5

Which protocol uses TCP port 49?

### Question 6

What is the purpose of the shared secret?

### Question 7

Why should SSH be used instead of Telnet?

### Question 8

What is the difference between authentication and authorization?

### Question 9

Why is a local user account useful when using a remote AAA server?

### Question 10

Which command displays the AAA configuration?

### Question 11

Which command can be used to troubleshoot RADIUS authentication?

### Question 12

Which command can be used to troubleshoot TACACS+ authentication?

---

# 31. Final Architecture

The complete AAA workflow is:

```text
                 Administrator
                       │
                       │ SSH
                       ▼
                 ┌───────────┐
                 │    R1     │
                 │ AAA Client│
                 └─────┬─────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
       ┌────────────┐    ┌─────────────┐
       │   RADIUS   │    │  TACACS+    │
       │   Server   │    │   Server    │
       │ .20.10     │    │   .20.20    │
       └────────────┘    └─────────────┘
              │                 │
              ▼                 ▼
        Authentication     Authentication
        Network Access     Device Admin
```

### Key idea

```text
AAA
│
├── Authentication → Who are you?
│
├── Authorization  → What can you do?
│
└── Accounting     → What did you do?
```

And remember:

```text
RADIUS
→ Network Access
→ UDP 1812/1813

TACACS+
→ Network Device Administration
→ TCP 49
→ Strong command authorization
```
