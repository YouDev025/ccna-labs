# Lab VoIP Service Basics

## 1. Objective

This lab introduces the basic configuration of a **VoIP (Voice over IP)** service in an enterprise network.

At the end of this lab, you should be able to:

* Build a small enterprise VoIP topology.
* Configure IP addressing.
* Configure DHCP for IP Phones.
* Create a dedicated Voice VLAN.
* Configure a Cisco router as a basic VoIP call manager using CME.
* Create telephone extensions.
* Register IP Phones with the router.
* Test calls between IP Phones.
* Verify VoIP configuration using `show` commands.
* Troubleshoot common VoIP problems.

---

# 2. Network Topology

The lab uses the following topology:

```text
                    ┌─────────────────┐
                    │      R1         │
                    │ Cisco CME       │
                    │ 192.168.10.1    │
                    └────────┬────────┘
                             │
                         Trunk Link
                             │
                    ┌────────┴────────┐
                    │       SW1       │
                    │    L2 Switch    │
                    └─────┬─────┬─────┘
                          │     │
                       Access Access
                          │     │
                    ┌─────┘     └─────┐
                    │                 │
              ┌─────┴─────┐     ┌─────┴─────┐
              │   IP      │     │    IP     │
              │  Phone 1  │     │  Phone 2  │
              │ Ext. 1001 │     │ Ext. 1002 │
              └───────────┘     └───────────┘
```

### Devices

| Device   | Quantity | Role                                  |
| -------- | -------: | ------------------------------------- |
| Router   |        1 | DHCP + CME / Call Manager             |
| Switch   |        1 | Voice VLAN                            |
| IP Phone |        2 | VoIP endpoints                        |
| PC       |        2 | Optional, connected through IP Phones |

---

# 3. Addressing Plan

| Device     | Interface | IP Address      | Network    |
| ---------- | --------- | --------------- | ---------- |
| R1         | G0/0.10   | 192.168.10.1/24 | Voice VLAN |
| IP Phone 1 | DHCP      | 192.168.10.0/24 | Voice VLAN |
| IP Phone 2 | DHCP      | 192.168.10.0/24 | Voice VLAN |
| SW1        | VLAN 10   | 192.168.10.2/24 | Management |

### Voice VLAN

```text
VLAN ID:       10
VLAN Name:     VOICE
Network:       192.168.10.0/24
Gateway:       192.168.10.1
```

### Telephone Extensions

```text
Phone 1 → Extension 1001
Phone 2 → Extension 1002
```

---

# 4. Step 1 — Create the Voice VLAN

On SW1:

```cisco
enable
configure terminal

vlan 10
 name VOICE

exit
```

Verify:

```cisco
show vlan brief
```

You should see:

```text
10    VOICE
```

---

# 5. Step 2 — Configure the Switch Trunk

The link between SW1 and R1 will transport VLAN 10.

Example:

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10
 no shutdown
exit
```

Verify:

```cisco
show interfaces trunk
```

Expected result:

```text
Port        Mode         Encapsulation
Gi0/1       on           802.1q
```

---

# 6. Step 3 — Configure the IP Phone Ports

Connect Phone 1 to:

```text
SW1 Fa0/1
```

Connect Phone 2 to:

```text
SW1 Fa0/2
```

Configure the ports:

```cisco
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit
```

Verify:

```cisco
show vlan brief
```

The phone ports should belong to VLAN 10.

---

# 7. Step 4 — Configure R1

Enter configuration mode:

```cisco
enable
configure terminal
```

Configure the physical interface:

```cisco
interface GigabitEthernet0/0
 no shutdown
exit
```

Create the Voice VLAN subinterface:

```cisco
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit
```

The router is now the default gateway for the Voice VLAN.

---

# 8. Step 5 — Configure DHCP for IP Phones

The router can provide IP addresses to the phones.

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.10
```

Create the DHCP pool:

```cisco
ip dhcp pool VOICE
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 option 150 ip 192.168.10.1
exit
```

### Important

Option 150 tells Cisco IP Phones where to find the **TFTP server** used to obtain their configuration.

In this basic lab, R1 also provides the required CME/TFTP functionality.

Verify DHCP:

```cisco
show ip dhcp pool
```

And:

```cisco
show ip dhcp binding
```

You should eventually see IP addresses assigned to the IP Phones.

---

# 9. Step 6 — Configure Cisco CME

Cisco CME provides basic call-manager functionality.

Enter:

```cisco
telephony-service
```

Configure the number of phones:

```cisco
max-ephones 2
max-dn 2
```

Configure the source address:

```cisco
ip source-address 192.168.10.1 port 2000
```

Configure automatic phone assignment:

```cisco
auto assign 1 to 2
```

Configure the TFTP files:

```cisco
create cnf-files
```

Exit:

```cisco
exit
```

---

# 10. Step 7 — Create Telephone Extensions

Create the first directory number:

```cisco
ephone-dn 1
 number 1001
 name Phone1
exit
```

Create the second:

```cisco
ephone-dn 2
 number 1002
 name Phone2
exit
```

The extensions are now:

```text
Phone 1 → 1001
Phone 2 → 1002
```

---

# 11. Step 8 — Configure IP Phones

You can explicitly configure the phones.

### Phone 1

```cisco
ephone 1
 mac-address XXXX.XXXX.XXXX
 type 7960
 button 1:1
exit
```

Replace:

```text
XXXX.XXXX.XXXX
```

with the MAC address of Phone 1.

### Phone 2

```cisco
ephone 2
 mac-address YYYY.YYYY.YYYY
 type 7960
 button 1:2
exit
```

Replace:

```text
YYYY.YYYY.YYYY
```

with the MAC address of Phone 2.

---

# 12. Step 9 — Save the Configuration

```cisco
end
write memory
```

or:

```cisco
copy running-config startup-config
```

---

# 13. Step 10 — Verify IP Addressing

On R1:

```cisco
show ip interface brief
```

Expected:

```text
Interface              IP-Address      Status
GigabitEthernet0/0     unassigned      up
GigabitEthernet0/0.10  192.168.10.1    up
```

Test connectivity:

```cisco
ping 192.168.10.2
```

If a PC is connected, test:

```cisco
ping 192.168.10.X
```

---

# 14. Step 11 — Verify DHCP

Use:

```cisco
show ip dhcp binding
```

Example:

```text
Bindings from all pools:

IP address       Client-ID
192.168.10.11    ...
192.168.10.12    ...
```

The phones should receive addresses from:

```text
192.168.10.11
192.168.10.12
...
```

---

# 15. Step 12 — Verify CME

Use:

```cisco
show telephony-service
```

Check:

* Maximum phones.
* Maximum directory numbers.
* Source address.
* TFTP configuration.

Example:

```text
max-ephones 2
max-dn 2
ip source-address 192.168.10.1 port 2000
```

---

# 16. Step 13 — Verify Registered Phones

Use:

```cisco
show ephone registered
```

You should see the registered phones.

Example:

```text
Mac Address      IP Address      Device Name
----             ----------      -----------
AAAA.BBBB.CCCC   192.168.10.11   SEP...
DDDD.EEEE.FFFF   192.168.10.12   SEP...
```

Another useful command:

```cisco
show ephone
```

---

# 17. Step 14 — Verify Directory Numbers

Use:

```cisco
show ephone-dn
```

Expected:

```text
Number 1001
Number 1002
```

You should see:

```text
1001 → Phone 1
1002 → Phone 2
```

---

# 18. Step 15 — Test the VoIP Call

From Phone 1:

```text
Dial 1002
```

Phone 2 should ring.

Then test the reverse direction:

```text
Phone 2 → 1001
```

Phone 1 should ring.

### Expected result

```text
Phone 1
1001
   │
   │  VoIP Call
   ▼
Phone 2
1002
```

The call should establish successfully.

---

# 19. Step 16 — Verify Active Calls

During an active call, use:

```cisco
show ephone
```

You can also use:

```cisco
show voice call status
```

depending on the IOS/Packet Tracer version.

---

# 20. Useful Verification Commands

### Interfaces

```cisco
show ip interface brief
```

### VLANs

```cisco
show vlan brief
```

### Trunks

```cisco
show interfaces trunk
```

### DHCP

```cisco
show ip dhcp binding
```

### DHCP pool

```cisco
show ip dhcp pool
```

### CME

```cisco
show telephony-service
```

### Registered phones

```cisco
show ephone registered
```

### Phones

```cisco
show ephone
```

### Extensions

```cisco
show ephone-dn
```

### Running configuration

```cisco
show running-config
```

---

# 21. Troubleshooting

## Problem 1 — Phone does not receive an IP address

Check:

```cisco
show ip dhcp binding
```

Check the interface:

```cisco
show ip interface brief
```

Check VLAN:

```cisco
show vlan brief
```

Check trunk:

```cisco
show interfaces trunk
```

Verify that the Voice VLAN is correctly configured.

---

## Problem 2 — Phone receives an IP but does not register

Check:

```cisco
show ephone registered
```

Verify CME:

```cisco
show telephony-service
```

Verify:

```cisco
ip source-address 192.168.10.1 port 2000
```

Verify DHCP Option 150:

```cisco
show running-config | section dhcp
```

You should have:

```cisco
option 150 ip 192.168.10.1
```

---

## Problem 3 — Phone 1 cannot call Phone 2

Check the directory numbers:

```cisco
show ephone-dn
```

Verify:

```text
Phone 1 → 1001
Phone 2 → 1002
```

Check registered phones:

```cisco
show ephone registered
```

Check CME:

```cisco
show telephony-service
```

---

## Problem 4 — VLAN communication does not work

Check:

```cisco
show vlan brief
```

and:

```cisco
show interfaces trunk
```

Make sure VLAN 10 exists:

```cisco
vlan 10
 name VOICE
```

And the trunk allows it:

```cisco
switchport trunk allowed vlan 10
```

---

# 22. Complete Verification Checklist

Before finishing the lab, verify the following:

* [ ] VLAN 10 exists.
* [ ] IP Phone ports belong to VLAN 10.
* [ ] Router trunk is operational.
* [ ] R1 has IP `192.168.10.1/24`.
* [ ] DHCP pool `VOICE` exists.
* [ ] DHCP Option 150 is configured.
* [ ] IP Phones receive IP addresses.
* [ ] CME is configured.
* [ ] Extensions `1001` and `1002` exist.
* [ ] Phone 1 registers successfully.
* [ ] Phone 2 registers successfully.
* [ ] Phone 1 can call Phone 2.
* [ ] Phone 2 can call Phone 1.
* [ ] Configuration is saved.

---

# 23. Final Expected Configuration

The important configuration on R1 should contain elements similar to:

```cisco
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

ip dhcp excluded-address 192.168.10.1 192.168.10.10

ip dhcp pool VOICE
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 option 150 ip 192.168.10.1

telephony-service
 max-ephones 2
 max-dn 2
 ip source-address 192.168.10.1 port 2000
 auto assign 1 to 2
 create cnf-files

ephone-dn 1
 number 1001
 name Phone1

ephone-dn 2
 number 1002
 name Phone2
```

---

# 24. Lab Questions

### Question 1

What is the purpose of a Voice VLAN?

### Question 2

Why is DHCP Option 150 important for Cisco IP Phones?

### Question 3

What is the role of Cisco CME?

### Question 4

What command shows registered IP Phones?

### Question 5

What command displays the configured telephone extensions?

### Question 6

What happens if the IP Phone receives an IP address but cannot register?

### Question 7

Why do we use a trunk between the switch and router?

### Question 8

What is the difference between an IP address and a telephone extension?

### Question 9

How can you verify that DHCP is assigning addresses correctly?

### Question 10

Which command can be used to verify the Voice VLAN?

---

# 25. Learning Outcome

After completing this lab, you should understand the basic workflow of an enterprise VoIP deployment:

```text
Switch
  ↓
Voice VLAN
  ↓
DHCP
  ↓
IP Phone receives IP
  ↓
Option 150
  ↓
CME / TFTP
  ↓
Phone Registration
  ↓
Extension
  ↓
VoIP Call
```

The complete service can therefore be summarized as:

```text
IP Addressing
      ↓
Voice VLAN
      ↓
DHCP
      ↓
CME
      ↓
IP Phone Registration
      ↓
Extensions
      ↓
Call
```
