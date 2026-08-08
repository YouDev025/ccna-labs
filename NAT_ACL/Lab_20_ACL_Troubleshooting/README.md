# Lab: ACL Troubleshooting

## Objective
Practice **systematically diagnosing and fixing broken ACL configurations** on a Cisco IOS router. Unlike the other labs in this series, you will start from a topology that is **already configured but broken** — five common real-world ACL mistakes have been deliberately introduced. Your job is to find and fix each one using `show` commands and structured troubleshooting, not by simply rewriting everything from scratch.

---

## Topology

```
                          +-----------+
                          |   SRV1    |   Web (80/443) + SSH (22) server
                          |192.168.40.20/24|
                          +-----------+
                                |
                              SW-DMZ
                                |
                        Gi0/2   |  192.168.40.1/24
                       +---------------+
                       |      R1       |
                       +---------------+
                Gi0/0  |               |  Gi0/1
      192.168.10.1/24  |               |  192.168.20.1/24
                        |               |
                      SW-A            SW-B
                        |               |
                +-------+-------+   +---------+
                |               |   |  PC3    |
           +---------+     +---------+ (Admin)|
           |  PC1    |     |  PC2    | +---------+
           | .10     |     | .11     |
           |(Users)  |     |(Users)  |
           +---------+     +---------+
```

- **R1** separates **Users** (VLAN 10), **Admin** (VLAN 20), and **DMZ** (VLAN 40).
- **Intended policy** (same as the Extended ACLs lab, so you have a known-good target to troubleshoot toward):
  1. Users may reach SRV1 on HTTP (80) and HTTPS (443) only.
  2. Admin may reach SRV1 on HTTP, HTTPS, SSH (22), and may ping it.
  3. Users must not reach the Admin subnet at all.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       | Default Gateway |
|--------|-----------|-------------------|--------------------|-----------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0      | —               |
| R1     | Gi0/1     | 192.168.20.1       | 255.255.255.0      | —               |
| R1     | Gi0/2     | 192.168.40.1       | 255.255.255.0      | —               |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0      | 192.168.10.1    |
| PC2    | NIC       | 192.168.10.11       | 255.255.255.0      | 192.168.10.1    |
| PC3    | NIC       | 192.168.20.10       | 255.255.255.0      | 192.168.20.1    |
| SRV1   | NIC       | 192.168.40.20       | 255.255.255.0      | 192.168.40.1    |

---

## Task 1 — Build the Topology and Load the Broken Configuration
1. Build the topology and addressing exactly as shown above.
2. Apply the following **starting configuration** to R1 exactly as written — this is the "broken" state you must diagnose. Do not fix anything yet; just apply it as-is:

```
ip access-list extended USERS-TO-DMZ
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 80
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.20 eq 443
 deny   ip 192.168.10.0 0.0.0.255 host 192.168.40.20
 permit ip any any

ip access-list extended ADMIN-TO-DMZ
 permit tcp 192.168.20.0 0.0.255.255 host 192.168.40.20 eq 80
 permit tcp 192.168.20.0 0.0.255.255 host 192.168.40.20 eq 443
 permit tcp 192.168.20.0 0.0.255.255 host 192.168.40.20 eq 22
 permit icmp 192.168.20.0 0.0.255.255 host 192.168.40.20

interface GigabitEthernet0/0
 ip access-group USERS-TO-DMZ out

interface GigabitEthernet0/1
 ip access-group ADMIN-TO-DMZ in
```

3. Verify baseline reachability of each device to its own default gateway before testing the policy itself (this confirms the topology/cabling/addressing is fine, so any failures you find next are genuinely ACL-related).

### Symptoms You Should Observe
- PC1/PC2 → SRV1 on HTTP/HTTPS: **fails** (should succeed).
- PC3 → SRV1 ping: **fails** (should succeed).
- PC3 → SRV1 SSH: **works inconsistently or fails** depending on platform behavior with the wildcard mask used.
- PC1 → PC3 (Admin subnet): currently **still succeeds** (should be blocked — this policy requirement was never even implemented).

---

## Task 2 — Investigate Systematically
Do **not** guess-and-check. Use this sequence on R1 for each broken behavior:
```
show ip interface GigabitEthernet0/0 | include access list
show ip interface GigabitEthernet0/1 | include access list
show access-lists USERS-TO-DMZ
show access-lists ADMIN-TO-DMZ
show run | section access-list
```
For each symptom, ask:
1. Is the ACL applied to the **right interface**?
2. Is it applied in the **right direction** (`in` vs `out`)?
3. Are the **wildcard masks** correct for the intended subnet?
4. Is a **required rule missing** entirely?
5. Are the match counters incrementing on the lines you expect, or on a different line than you expect (indicating a processing-order problem)?

---

## Task 3 — Find and Fix Bug #1 (Wrong Direction on Users ACL)
**Symptom:** PC1/PC2 cannot reach SRV1 on HTTP/HTTPS at all, even though the ACL clearly has `permit` lines for exactly that traffic.
**Diagnosis:** `USERS-TO-DMZ` is applied `out` on Gi0/0 — but Gi0/0 is the interface where Users traffic **enters** R1, so it needs to be filtered `in`, not `out`. Applied `out`, it's evaluating traffic in the wrong direction relative to where Users' packets actually travel through this interface.
**Fix:**
```
interface GigabitEthernet0/0
 no ip access-group USERS-TO-DMZ out
 ip access-group USERS-TO-DMZ in
```
**Retest:** PC1/PC2 → SRV1 HTTP/HTTPS should now succeed.

---

## Task 4 — Find and Fix Bug #2 (Incorrect Wildcard Mask on Admin ACL)
**Symptom:** `ADMIN-TO-DMZ` was clearly intended to match the 192.168.20.0/24 subnet, but the wildcard mask used is `0.0.255.255`.
**Diagnosis:** `0.0.255.255` matches far more than intended — it matches any address in `192.168.0.0` through `192.168.255.255` with the third and fourth octet being anything, which is much broader than the single /24 this policy is supposed to scope to. For a /24, the correct wildcard is `0.0.0.255`.
**Fix (rebuild the ACL with the correct mask on every line):**
```
no ip access-list extended ADMIN-TO-DMZ
ip access-list extended ADMIN-TO-DMZ
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 80
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 443
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.20 eq 22
 permit icmp 192.168.20.0 0.0.0.255 host 192.168.40.20
 permit ip any any
```
> Note the added trailing `permit ip any any` — the original also lacked this, meaning the implicit deny was silently blocking everything not explicitly matched. This is Bug #3, fixed here at the same time since rebuilding the ACL either way.

**Retest:** PC3 → SRV1 SSH and other permitted traffic should now work correctly and predictably.

---

## Task 5 — Find and Fix Bug #3 (Missing Trailing Permit — Implicit Deny)
Already addressed in Task 4's rebuild above — call this out explicitly in your lab notes:
**Symptom:** Before the fix, any traffic from Admin to SRV1 that wasn't HTTP/HTTPS/SSH/ICMP (or any Admin traffic to any *other* destination through this same ACL) would silently fail, because there was no trailing `permit ip any any` and the implicit deny caught it.
**Verify this was the cause** by checking `show access-lists ADMIN-TO-DMZ` before your fix — no explicit deny line existed, meaning any denied match had to be attributed to the implicit deny, confirmed by a non-zero total deny count with no visible deny statement.

---

## Task 6 — Find and Fix Bug #4 (ADMIN-TO-DMZ Applied Inbound in the Wrong Place)
**Symptom:** Even after Tasks 4–5, double-check direction/placement logic here too. `ADMIN-TO-DMZ` is applied `in` on Gi0/1 — Gi0/1 **is** the Admin-facing interface, and Admin traffic **does** enter R1 there, so this one is actually correct. Confirm this explicitly rather than assuming a bug exists everywhere — **not every symptom is a placement bug**, and part of troubleshooting is correctly ruling things out.
**Verification-only step:**
```
show ip interface GigabitEthernet0/1 | include access list
```
Confirm it reads `ADMIN-TO-DMZ` applied `in`, and leave it unchanged.

---

## Task 7 — Find and Fix Bug #5 (Missing Policy Entirely — Users to Admin)
**Symptom:** PC1 → PC3 (Admin subnet) currently succeeds, but the policy requires this to be blocked.
**Diagnosis:** No rule anywhere denies Users-to-Admin traffic — this isn't a misconfiguration so much as a **missing requirement** that was never implemented.
**Fix:**
```
ip access-list extended USERS-TO-DMZ
 5 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
```
> Using a low sequence number (`5`) inserts this line **before** the existing `permit tcp ... eq 80/443` lines' sequence numbers if needed — check with `show access-lists USERS-TO-DMZ` first to confirm your new line lands before the trailing `permit ip any any`, and adjust the sequence number if it doesn't.

**Retest:** PC1/PC2 → PC3 should now fail, while PC1/PC2 → SRV1 HTTP/HTTPS continues to succeed.

---

## Task 8 — Full Regression Test
Re-run every test from the intended policy, now that all five issues are fixed:
- PC1/PC2 → SRV1 HTTP/HTTPS: succeed.
- PC1/PC2 → SRV1 ping/SSH: fail.
- PC1/PC2 → PC3 (Admin): fail.
- PC3 → SRV1 HTTP/HTTPS/SSH/ping: all succeed.

```
show access-lists
show run | section access-list
show ip interface GigabitEthernet0/0 | include access list
show ip interface GigabitEthernet0/1 | include access list
```

---

## Verification Checklist

| Bug | Symptom | Root Cause | Fix |
|---|---|---|---|
| #1 | Users can't reach SRV1 at all on 80/443 | `USERS-TO-DMZ` applied `out` instead of `in` on Gi0/0 | Reapply `in` |
| #2 | Admin ACL matches far more than the /24 subnet | Wildcard mask `0.0.255.255` instead of `0.0.0.255` | Rebuild with correct mask |
| #3 | Unrelated Admin traffic silently dropped | Missing trailing `permit ip any any` | Add explicit trailing permit |
| #4 | (False lead) | `ADMIN-TO-DMZ` direction was actually already correct | No change — confirmed via `show ip interface` |
| #5 | Users can still reach Admin subnet | Policy requirement never implemented | Add `deny` line, correctly sequenced |
| Final regression test | All four policy requirements | — | All pass |

---

## Challenge (Optional)
- Intentionally reintroduce Bug #2's wildcard mask error on a **different** ACL you build yourself for a new subnet, hand your device configuration to a lab partner without telling them what's wrong, and have them apply the same systematic troubleshooting method from this lab to find it independently.
- Write a personal "ACL troubleshooting checklist" based on the five bug categories in this lab (direction, wildcard mask, missing trailing permit, false-lead verification, missing requirement) that you can reuse on future labs or real equipment.
- Using `show access-lists` sequence numbers, practice inserting a new `deny` rule at a specific position in an existing ACL without deleting and rebuilding the whole thing, and confirm processing order is exactly what you intended.