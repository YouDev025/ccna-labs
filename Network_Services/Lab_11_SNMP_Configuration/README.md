# Lab: SNMP Configuration

## Objective
Configure and verify **SNMP** on a Cisco IOS router for both monitoring (polling) and alerting (traps): start with SNMPv2c community strings to understand the basics, then upgrade to **SNMPv3** with authentication and encryption, and confirm everything with `show snmp` commands plus real polling/trap tools from a network management station (NMS).

---

## Topology

```
                          +-----------+
                          |    NMS    |   Network management station
                          |192.168.80.10/24 |  (runs snmpget/snmpwalk, receives traps)
                          +-----------+
                                |
                              SW1
                                |
                        Gi0/1   |  192.168.80.1/24
           +---------------+
           |      R1       |   (SNMP agent)
           +---------------+
     Gi0/0  |
             |
      +---------+
      |  PC1    |   Used to generate a monitored event (interface flap)
      |192.168.10.10/24|
      +---------+
```

- **R1** is the SNMP **agent** — the device being monitored/managed.
- **NMS** is the SNMP **manager** — it polls R1 for data and receives trap notifications. In a lab environment this can be a Linux host with `net-snmp` tools (`snmpget`, `snmpwalk`), a dedicated NMS product, or a simulated syslog/trap receiver if your platform doesn't support real SNMP polling.
- **PC1** is used to generate a monitored event so a trap has something real to report.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1     | Gi0/1     | 192.168.80.1       | 255.255.255.0        |
| NMS    | NIC       | 192.168.80.10       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |

---

## Background: SNMP Versions
- **SNMPv1/v2c** — uses plaintext **community strings** as a shared "password" (commonly `public` for read-only, `private` for read-write in default/example configs — **never use these defaults in production**). No encryption, no per-user authentication; anyone who can send a packet with the right community string and reach the device can query or, with a read-write string, modify it.
- **SNMPv3** — adds real **per-user authentication** (MD5/SHA) and optional **encryption** (DES/AES) of the SNMP payload itself. This is the version that should be used in any real deployment; v2c is included in this lab mainly so you understand what SNMPv3 is improving on.

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, NMS, and PC1 as shown.
2. Apply the addressing above and confirm R1 can ping NMS, and PC1 can ping R1.

### Task 2 — Configure SNMPv2c (Read-Only) First
```
snmp-server community LabPublicRO RO
```
> Use a non-default string like `LabPublicRO` rather than `public`, even in a lab — building the right habit matters more than the specific lab value.

### Task 3 — Poll R1 from the NMS (SNMPv2c)
From the NMS (or any host with SNMP tools installed):
```
snmpget -v2c -c LabPublicRO 192.168.80.1 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c LabPublicRO 192.168.80.1 1.3.6.1.2.1.1
```
The first command queries `sysName` (the device's configured hostname via the standard MIB-II OID); the second walks the broader `system` subtree, returning multiple system-identification values.

### Task 4 — Add a Read-Write Community and Understand the Risk
```
snmp-server community LabPrivateRW RW
```
> A read-write community string allows remote configuration changes via SNMP `SET` operations. In this lab, only add it to observe the difference in access level — in production, prefer SNMPv3 with proper access control over any RW community string, and restrict even that with an ACL (Task 5).

### Task 5 — Restrict SNMP Access with an ACL
Limit which hosts are even allowed to query R1's SNMP agent at all, regardless of community string correctness:
```
access-list 40 permit host 192.168.80.10
snmp-server community LabPublicRO RO 40
snmp-server community LabPrivateRW RW 40
```
Test from a different, non-permitted source address (if available in your topology) and confirm SNMP queries are rejected/timeout, while NMS (192.168.80.10) continues to work normally.

### Task 6 — Configure SNMP Traps
Enable R1 to proactively notify the NMS of significant events, rather than relying solely on polling:
```
snmp-server enable traps
snmp-server host 192.168.80.10 version 2c LabPublicRO
```
> `snmp-server enable traps` with no further qualifiers enables a broad default set of trap categories; in production you would typically scope this to only the trap types you actually want (e.g., `snmp-server enable traps snmp linkdown linkup`) to avoid unnecessary noise.

### Task 7 — Trigger and Observe a Trap
Generate a link-status change to produce a trap:
```
interface GigabitEthernet0/0
 shutdown
 no shutdown
```
On the NMS, confirm a trap was received (using a trap receiver tool such as `snmptrapd`, or your platform's equivalent) showing the link-down/link-up event from R1.

### Task 8 — Upgrade to SNMPv3
Configure a proper SNMPv3 group and user with authentication and encryption:
```
snmp-server group LAB-V3-GROUP v3 priv
snmp-server user labadmin LAB-V3-GROUP v3 auth sha AuthPass123 priv aes 128 PrivPass456
```
> `auth sha AuthPass123` sets the authentication protocol/password; `priv aes 128 PrivPass456` sets the encryption protocol/password. Both should be strong, unique values in any real deployment — these are lab placeholders only.

### Task 9 — Poll R1 with SNMPv3
From the NMS:
```
snmpget -v3 -u labadmin -l authPriv -a SHA -A AuthPass123 -x AES -X PrivPass456 192.168.80.1 1.3.6.1.2.1.1.5.0
```
This should succeed and return the same `sysName` value as the SNMPv2c test in Task 3, but now the exchange is authenticated and encrypted.

### Task 10 — Verify
On R1:
```
show snmp
show snmp community
show snmp user
show snmp group
show snmp host
show access-lists 40
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| SNMPv2c `snmpget`/`snmpwalk` from NMS | Succeeds using `LabPublicRO`; fails or is rejected with a wrong community string |
| SNMP ACL (Task 5) | Queries from a non-permitted source fail even with the correct community string |
| Interface flap trap | Received at the NMS trap receiver, correctly identifying R1 and the affected interface |
| SNMPv3 `snmpget` with auth+priv | Succeeds and returns the same value as the v2c query, confirming SNMPv3 works correctly |
| SNMPv3 with wrong auth/priv password | Fails — confirms authentication is actually being enforced, not just configured |
| `show snmp` on R1 | Shows packet counters (in/out) incrementing as each polling test is run |
| `show snmp community` | Lists both community strings and confirms ACL 40 is attached to each |
| `show snmp user` / `show snmp group` | Confirms `labadmin` and `LAB-V3-GROUP` exist with the expected auth/priv settings |

---

## Challenge (Optional)
- Remove the SNMPv2c community strings entirely once SNMPv3 is confirmed working, and re-test to confirm v2c polling now fails while v3 continues to succeed — reflecting the recommended production practice of retiring v2c once v3 is fully deployed.
- Configure a second SNMPv3 user with **read-only**, **auth-only** (no encryption) access, and compare its `show snmp user` output and behavior against the full `authPriv` user from Task 8.
- Research and document (implementing if your platform supports it) how SNMP traps differ from **SNMP informs** — specifically, why informs (which require acknowledgment from the receiver) can be more reliable over unreliable network paths than plain traps.