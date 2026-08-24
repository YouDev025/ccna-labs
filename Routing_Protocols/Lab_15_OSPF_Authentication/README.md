# Lab: OSPF Authentication

## Objective
This lab isolates **one specific skill**: securing OSPF neighbor relationships with authentication, progressing from **plaintext** (to directly observe why it's inadequate) to **MD5**, then to **area-wide authentication** so every interface in an area inherits the requirement automatically. As with the NTP Authentication and EIGRP Authentication labs, you'll finish by introducing a simulated **rogue router** and confirming it's correctly rejected — including the more realistic case of a wrong key rather than no key at all.

---

## Topology

```
  192.168.1.0/24            10.0.12.0/30            10.0.23.0/30
  (Branch A LAN)          (A <-> B link,           (B <-> Rogue link)
        |                  authenticated)                 |
      SW-A                       |                  +-----------+
        |                 +-----------+       Gi0/1 |    R-B    | Gi0/0
  Gi0/0 | .1         Gi0/1|    R-A    |Gi0/1  10.0.23.2 +-----------+  10.0.23.1
        +-----------+     +-----------+                        |
        |    R-A    |      10.0.12.1                     +-----------+
        +-----------+                                     |  R-ROGUE  |
      PC-A1                                                +-----------+
  192.168.1.10
```

- **R-A** and **R-B** are the legitimate, authenticated OSPF pair — Area 0 throughout, kept simple (single area) so the focus stays entirely on authentication.
- **R-ROGUE** attaches to R-B and attempts to join Area 0 without valid credentials.

---

## IP Addressing Table

| Device   | Interface | IP Address | Subnet Mask       |
|----------|-----------|-------------|---------------------|
| R-A      | Gi0/0     | 192.168.1.1  | 255.255.255.0        |
| R-A      | Gi0/1     | 10.0.12.1     | 255.255.255.252      |
| R-B      | Gi0/1     | 10.0.12.2     | 255.255.255.252      |
| R-B      | Gi0/0     | 10.0.23.2     | 255.255.255.252      |
| R-ROGUE  | Gi0/0     | 10.0.23.1     | 255.255.255.252      |
| PC-A1    | NIC       | 192.168.1.10  | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology
1. Place R-A, R-B, R-ROGUE, and PC-A1 as shown, and apply the addressing above.
2. Confirm baseline IP reachability between adjacent devices before configuring OSPF.

### Task 2 — Enable OSPF on R-A and R-B (No Authentication Yet)
```
! R-A
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0

! R-B
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
```
> Note R-B intentionally advertises the link toward R-ROGUE — that link being open and untested is the entire point of this lab.

### Task 3 — Verify Baseline Adjacency
```
show ip ospf neighbor
```
Confirm R-A and R-B form a `FULL` adjacency. R-ROGUE has no OSPF configuration yet, so it shouldn't appear anywhere.

---

## Part 1 — Plaintext Authentication (Understand Why It's Weak)

### Task 4 — Configure Plaintext Authentication Between R-A and R-B
```
! On R-A
interface GigabitEthernet0/1
 ip ospf authentication
 ip ospf authentication-key PlainTextPass1

! On R-B
interface GigabitEthernet0/1
 ip ospf authentication
 ip ospf authentication-key PlainTextPass1
```

### Task 5 — Verify the Adjacency Still Forms
```
show ip ospf neighbor
```
Confirm R-A and R-B remain `FULL` — legitimate peers with the matching plaintext password authenticate successfully.

### Task 6 — Observe the Weakness of Plaintext Authentication
If your lab platform supports packet capture on the R-A↔R-B link, capture a few OSPF Hello packets and inspect them — the authentication key is sent **completely unencrypted**, readable by anyone who can see the traffic. If packet capture isn't available in your environment, note this as a conceptual observation: plaintext OSPF authentication protects against a device that doesn't know the key at all, but provides **no protection** against anyone able to passively observe the link, since the key itself is visible in every Hello packet. This is exactly why MD5 (Part 2) should be used instead in any real deployment.

---

## Part 2 — MD5 Authentication

### Task 7 — Replace Plaintext with MD5 Authentication
```
! On R-A
interface GigabitEthernet0/1
 no ip ospf authentication
 no ip ospf authentication-key PlainTextPass1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OspfLabSecret456

! On R-B
interface GigabitEthernet0/1
 no ip ospf authentication
 no ip ospf authentication-key PlainTextPass1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 OspfLabSecret456
```

### Task 8 — Verify MD5 Authentication Works
```
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/1
```
Confirm the adjacency remains `FULL`, and the interface output confirms message-digest authentication is active. Unlike plaintext, the actual key value is now hashed before transmission, not sent in the clear.

---

## Part 3 — Area-Wide Authentication (Simplify Multi-Interface Configuration)

### Task 9 — Convert to Area-Level Authentication
Rather than configuring authentication on every individual interface, Area 0 can require message-digest authentication **by default** for all its interfaces at once:
```
! On R-A
interface GigabitEthernet0/1
 no ip ospf authentication message-digest
router ospf 1
 area 0 authentication message-digest

! On R-B
interface GigabitEthernet0/1
 no ip ospf authentication message-digest
router ospf 1
 area 0 authentication message-digest
```
> Note the `ip ospf message-digest-key` lines from Task 7 are **not** removed — the key itself stays on the interface; only the requirement to *use* authentication moves from the interface level to the area level. This means adding a new interface to Area 0 later automatically requires authentication without needing to remember to configure it individually each time.

### Task 10 — Verify Area-Level Authentication
```
show ip ospf neighbor
show ip ospf
```
Confirm the adjacency remains `FULL`, and `show ip ospf` reports Area 0 as requiring message-digest authentication.

---

## Part 4 — Rogue Router Rejection

### Task 11 — Configure R-ROGUE Without Authentication
```
! On R-ROGUE
router ospf 1
 router-id 99.99.99.99
 network 10.0.23.0 0.0.0.3 area 0
```
Deliberately no authentication configuration — representing an attacker or an unmanaged device.

### Task 12 — Apply Area Authentication to R-B's Interface Toward R-ROGUE
Since R-B's Area 0 authentication requirement is area-wide (Task 9), simply bringing up a new Area 0 interface on R-B is enough — but it still needs its own message-digest key configured, since (as noted in Task 9) the key itself remains a per-interface setting even under area-wide enforcement:
```
! On R-B
interface GigabitEthernet0/0
 ip ospf message-digest-key 1 md5 OspfLabSecret456
```

### Task 13 — Verify R-ROGUE Is Rejected
```
show ip ospf neighbor
show logging | include OSPF
```
Confirm no adjacency forms between R-B and R-ROGUE, and check for an authentication-related log message.

### Task 14 — Simulate a Key Mismatch
```
! On R-ROGUE
router ospf 1
 area 0 authentication message-digest
interface GigabitEthernet0/0
 ip ospf message-digest-key 1 md5 WrongKeyValue
```
```
show ip ospf neighbor
show logging | include OSPF
```
Confirm the adjacency still fails to form, and compare the log detail (if any) against Task 13's no-authentication-at-all case.

### Task 15 — Positive Control Test: Correct the Key and Confirm Acceptance
```
! On R-ROGUE
interface GigabitEthernet0/0
 no ip ospf message-digest-key 1 md5 WrongKeyValue
 ip ospf message-digest-key 1 md5 OspfLabSecret456
```
```
show ip ospf neighbor
```
Confirm R-ROGUE **now** successfully forms an adjacency — proving the earlier rejections in Tasks 13 and 14 were genuinely caused by authentication, not some unrelated issue.

### Task 16 — Restore the Secure, Intended Final State
```
! On R-ROGUE
no router ospf 1
```
```
! On R-B
show ip ospf neighbor
```
Confirm only R-A remains as R-B's legitimate neighbor.

### Task 17 — Final Verification
```
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/1
show ip ospf
show run | section router ospf
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — plaintext auth | R-A↔R-B adjacency forms; key travels unencrypted (conceptually or observably, depending on platform) |
| Task 8 — MD5 auth | Adjacency remains `FULL`; key is now hashed, not sent in the clear |
| Task 10 — area-level auth | Adjacency remains `FULL`; `show ip ospf` confirms Area 0 requires message-digest authentication |
| Task 13 — R-ROGUE, no key | No adjacency forms; log shows an authentication-related rejection |
| Task 14 — R-ROGUE, wrong key | Still rejected |
| Task 15 — R-ROGUE, corrected key | Adjacency **does** form, confirming earlier rejections were genuinely authentication-related |
| Task 16 — final state | Only R-A remains as R-B's neighbor |

---

## Challenge (Optional)
- Research and configure (if your IOS version supports it) **SHA-based OSPF authentication via key chains** (`ip ospf authentication key-chain`), which is newer and stronger than MD5, and compare its configuration syntax against the MD5 approach used in this lab.
- Configure area-level authentication on a **third** area in a more complex topology and confirm each area's authentication requirement is independent — a key mismatch or missing key in one area should not affect adjacencies in a different area.
- Write a short comparison in your lab notes of plaintext, MD5, and area-wide MD5 authentication, specifically addressing: what each protects against, what each does not, and why area-wide configuration reduces (but does not eliminate) the risk of a forgotten per-interface authentication setting on a newly added link.