# Lab: EIGRP Authentication

## Objective
This lab isolates **one specific skill**: securing EIGRP neighbor relationships with **MD5 authentication** via key chains. Building on the EIGRP Basics lab, you will configure authentication between two legitimate routers, then introduce a simulated **rogue router** attempting to join the EIGRP domain without valid credentials — and confirm it is correctly rejected. You'll also practice diagnosing a **key mismatch**, which produces a subtly different symptom than a missing key entirely.

---

## Topology

```
  192.168.1.0/24                 10.0.12.0/30                 10.0.23.0/30
  (Branch A LAN)                (A <-> B link,               (B <-> Rogue link,
        |                        authenticated)                NOT authenticated by Rogue)
      SW-A                              |                             |
        |                        +-----------+                 +-----------+
  Gi0/0 | .1               Gi0/1 |    R-A    | Gi0/1     Gi0/0  |    R-B    | Gi0/1    Gi0/0  +-----------+
        +-----------+     10.0.12.1  +-----------+  10.0.12.2   +-----------+  10.0.23.2  10.0.23.1 | R-ROGUE |
        |    R-A    |                                                                              +-----------+
        +-----------+
      PC-A1
  192.168.1.10
```

- **R-A** and **R-B** are the legitimate, authenticated EIGRP pair — same AS and topology style as the EIGRP Basics lab, simplified to two routers plus a rogue for clarity.
- **R-ROGUE** attaches to R-B and attempts to join the same EIGRP autonomous system, simulating either an attacker or an improperly-provisioned device.

---

## IP Addressing Table

| Device   | Interface | IP Address | Subnet Mask       |
|----------|-----------|-------------|---------------------|
| R-A      | Gi0/0     | 192.168.1.1  | 255.255.255.0        |
| R-A      | Gi0/1     | 10.0.12.1     | 255.255.255.252      |
| R-B      | Gi0/0     | 10.0.12.2     | 255.255.255.252      |
| R-B      | Gi0/1     | 10.0.23.2     | 255.255.255.252      |
| R-ROGUE  | Gi0/0     | 10.0.23.1     | 255.255.255.252      |
| PC-A1    | NIC       | 192.168.1.10  | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology
1. Place R-A, R-B, R-ROGUE, and PC-A1 as shown, and apply the addressing above.
2. Confirm baseline IP reachability between adjacent devices before configuring EIGRP.

### Task 2 — Enable EIGRP on R-A and R-B (No Authentication Yet)
```
! R-A
router eigrp 100
 network 192.168.1.0 0.0.0.255
 network 10.0.12.0 0.0.0.3
 no auto-summary
 passive-interface GigabitEthernet0/0

! R-B
router eigrp 100
 network 10.0.12.0 0.0.0.3
 network 10.0.23.0 0.0.0.3
 no auto-summary
```
> Note R-B intentionally advertises `10.0.23.0 0.0.0.3` — the link toward R-ROGUE — since that's exactly the segment being tested. R-ROGUE joining or being rejected on this link is the entire point of the lab.

### Task 3 — Verify Baseline Neighbor Formation (R-A ↔ R-B Only)
```
show ip eigrp neighbors
```
Confirm R-A and R-B see each other as neighbors. R-ROGUE has not been configured with EIGRP yet, so it should not appear anywhere.

### Task 4 — Configure a Key Chain on R-A and R-B
```
! On both R-A and R-B
key chain EIGRP-KEYS
 key 1
  key-string EigrpLabSecret123
```

### Task 5 — Apply MD5 Authentication to the R-A ↔ R-B Interface
```
! On R-A
interface GigabitEthernet0/1
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP-KEYS

! On R-B
interface GigabitEthernet0/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP-KEYS
```

### Task 6 — Verify Authenticated Adjacency Still Forms
```
show ip eigrp neighbors
```
Confirm R-A and R-B are still neighbors — authentication should be transparent to legitimate, correctly-configured peers. Confirm reachability is unaffected:
```
! From PC-A1
ping 10.0.23.2
```

### Task 7 — Configure R-ROGUE Without Authentication (Attacker/Unmanaged Device Scenario)
```
! On R-ROGUE
router eigrp 100
 network 10.0.23.0 0.0.0.3
 no auto-summary
```
Deliberately do **not** configure a key chain or authentication on R-ROGUE — this represents an attacker (who wouldn't have the key) or simply an unauthorized device someone connected to the network.

### Task 8 — Apply Authentication to R-B's Interface Toward R-ROGUE
```
! On R-B
interface GigabitEthernet0/1
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP-KEYS
```

### Task 9 — Verify R-ROGUE Is Rejected
```
show ip eigrp neighbors
```
Confirm **no** neighbor relationship forms between R-B and R-ROGUE. Check R-B's logs for the specific reason:
```
show logging | include EIGRP
```
Look for an authentication-failure-related message (exact wording varies by IOS version) indicating a Hello packet was received but rejected due to authentication mismatch.

### Task 10 — Simulate a Key Mismatch (More Realistic Than a Missing Key)
Configure R-ROGUE with authentication enabled, but using the **wrong** key value — simulating a misconfigured legitimate device rather than an obvious attacker with no key at all:
```
! On R-ROGUE
key chain EIGRP-KEYS
 key 1
  key-string WrongSecretValue

interface GigabitEthernet0/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP-KEYS
```

### Task 11 — Verify the Key Mismatch Is Still Correctly Rejected
```
show ip eigrp neighbors
show logging | include EIGRP
```
Confirm the neighbor relationship still fails to form. Compare the log message (if any) between Task 9 (no authentication attempted at all) and this task (authentication attempted, but with a mismatched key) — note whether IOS's messaging distinguishes between these two cases on your platform, and record what you observe.

### Task 12 — Correct R-ROGUE's Key and Confirm It Can Now Join (Controlled Test)
To confirm your authentication configuration is working correctly in both directions — not just correctly rejecting bad actors, but correctly accepting legitimate ones — temporarily correct R-ROGUE's key to match:
```
! On R-ROGUE
key chain EIGRP-KEYS
 key 1
  key-string EigrpLabSecret123
```
```
show ip eigrp neighbors
```
Confirm R-ROGUE **now** successfully forms an adjacency with R-B, proving the rejections in Tasks 9 and 11 were genuinely due to authentication, not some unrelated misconfiguration. In your lab notes, explain why this "does it work when correct" control test matters for trusting your earlier "rejected when wrong" results.

### Task 13 — Restore the Secure, Intended End State
Since R-ROGUE represents an untrusted/simulated device, remove it from the EIGRP domain entirely to leave the lab in its intended secure final state:
```
! On R-ROGUE
no router eigrp 100
```
```
! On R-B
show ip eigrp neighbors
```
Confirm only R-A remains as R-B's neighbor.

### Task 14 — Verify
```
show ip eigrp neighbors
show key chain
show run | section eigrp
show run interface GigabitEthernet0/1
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — pre-authentication baseline | R-A ↔ R-B neighbor relationship forms normally |
| Task 6 — post-authentication | R-A ↔ R-B relationship still forms correctly; reachability unaffected |
| Task 9 — R-ROGUE with no authentication configured | No adjacency forms; log shows an authentication-related rejection |
| Task 11 — R-ROGUE with wrong key | Still rejected; compare log detail against Task 9 |
| Task 12 — R-ROGUE with corrected key | Adjacency **does** form, confirming the earlier rejections were genuinely authentication-related and not some other misconfiguration |
| Task 13 — final state | Only R-A remains as R-B's legitimate neighbor |
| `show key chain` | Confirms the configured key chain and key string details (masked or visible depending on platform) on each router |

---

## Challenge (Optional)
- Configure a **second key** (key 2) on the R-A↔R-B key chain with a future `accept-lifetime`/`send-lifetime` window, and practice rotating from key 1 to key 2 without dropping the adjacency — a real-world key-rotation exercise.
- Research and document why EIGRP (and most routing protocol) authentication uses a **shared secret with MD5 hashing** rather than encrypting the routing updates themselves — explain what authentication protects against versus what it does not (hint: it protects against a rogue device injecting fabricated routes; it does not, by itself, provide confidentiality of the routing information being exchanged).
- Combine this lab with the EIGRP Basics lab's topology (three legitimate routers plus the backup link) and apply authentication across all EIGRP-speaking interfaces, then verify the feasible successor failover behavior from that lab still works identically with authentication enabled throughout.