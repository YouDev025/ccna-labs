# Lab: Firewall Concepts

## Objective
Understand the practical difference between **stateless** filtering (plain ACLs) and **stateful** filtering, by building the same basic security goal three different ways with increasing sophistication: a plain extended ACL (and its limitations), a **reflexive ACL** (session-aware, but still fundamentally ACL-based), and Cisco IOS **Zone-Based Policy Firewall (ZBFW)** — genuine stateful inspection, the modern approach. Same topology and same goal throughout, so the differences are directly comparable.

---

## Topology

```
                          "Outside" / Internet (simulated)
                                       |
                                Gi0/1  |  203.0.113.1/29
                       +---------------+
                       |      R1       |
                       +---------------+
                        Gi0/0  |  192.168.10.1/24
                                |
                              SW1
                                |
                          +---------+
                          |  PC1    |   Inside host initiating outbound connections
                          |192.168.10.10/24|
                          +---------+

                          +-----------+
                          |  SRV1     |   Outside web server
                          |203.0.113.10/29|
                          +-----------+
```

- **R1** separates a trusted **inside** network (PC1's LAN) from an untrusted **outside** network (SRV1, representing the internet).
- **Goal throughout this lab**: PC1 should be able to initiate connections to SRV1 and receive replies; SRV1 (or anything else on the outside) should **never** be able to initiate a connection inward to PC1.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1     | Gi0/1     | 203.0.113.1          | 255.255.255.248      |
| SRV1   | NIC       | 203.0.113.10          | 255.255.255.248      |
| PC1    | NIC       | 192.168.10.10        | 255.255.255.0        |

---

## Part 1 — Plain Extended ACL (Stateless) and Its Limitation

### Task 1 — Build the Topology
Place R1, SW1, PC1, and SRV1 as shown, and apply the addressing above. Confirm basic reachability before any ACL is applied.

### Task 2 — Attempt the Goal with a Simple Outbound-Only ACL
```
ip access-list extended OUTSIDE-IN
 deny ip any any
```
```
interface GigabitEthernet0/1
 ip access-group OUTSIDE-IN in
```

### Task 3 — Observe the Problem
```
! From PC1
curl http://203.0.113.10
```
This **fails** — even though nothing was written to block PC1's outbound request specifically, the reply traffic coming **back** from SRV1 is itself inbound on Gi0/1, and is caught by the blanket `deny ip any any`. A stateless ACL has no concept of "this inbound packet is a reply to a connection we ourselves initiated" — it only sees each packet in isolation, matched against static rules with no memory of what came before.

### Task 4 — The Naive (and Flawed) Fix
A common but risky workaround is to just permit all inbound TCP traffic with the ACK bit set, assuming (incorrectly) that this safely represents "only replies":
```
ip access-list extended OUTSIDE-IN
 no deny ip any any
 permit tcp any any established
 deny ip any any
```
```
! From PC1
curl http://203.0.113.10
```
This now succeeds — but explain in your lab notes why this is **not** genuinely stateful and why it's a weaker control than it appears: `established` merely checks whether the ACK (or RST) flag is set on the packet — it has **no actual knowledge of whether a real corresponding outbound connection exists at all**. An attacker can simply craft a packet with the ACK bit set from the outside and it will be let straight through by this rule, since the router isn't tracking any real session state, just inspecting a single flag on each individual packet.

---

## Part 2 — Reflexive ACLs (Session-Aware, Still ACL-Based)

### Task 5 — Remove the Flawed Established-Based Rule
```
no ip access-list extended OUTSIDE-IN
```

### Task 6 — Configure a Reflexive ACL
Reflexive ACLs dynamically create a temporary, session-specific permit entry for return traffic, triggered by outbound traffic actually being seen — a genuine improvement over guessing based on a single flag:
```
ip access-list extended OUTBOUND-TRACK
 permit tcp any any reflect REFLECT-SESSIONS
 permit udp any any reflect REFLECT-SESSIONS
 permit icmp any any reflect REFLECT-SESSIONS

ip access-list extended OUTSIDE-IN
 evaluate REFLECT-SESSIONS
 deny ip any any
```
```
interface GigabitEthernet0/0
 ip access-group OUTBOUND-TRACK out
interface GigabitEthernet0/1
 ip access-group OUTSIDE-IN in
```

### Task 7 — Verify Reflexive ACL Behavior
```
! From PC1
curl http://203.0.113.10
```
Should succeed. While the connection is active (or immediately after):
```
show ip access-lists REFLECT-SESSIONS
```
Confirm a **temporary** reflexive entry now exists, specific to this session (matching PC1's actual source port and SRV1's address) — unlike the `established` workaround, this entry only exists because a genuine matching outbound packet was actually observed, and it will expire automatically once the session ends or times out.

### Task 8 — Confirm Unsolicited Inbound Traffic Is Still Blocked
```
! From SRV1, attempt to initiate a connection toward PC1
curl http://192.168.10.10
```
This should fail — reflexive ACLs only create return-path entries for traffic that was actually initiated from the inside; they never permit new inbound-initiated sessions.

---

## Part 3 — Zone-Based Policy Firewall (True Stateful Inspection)

### Task 9 — Remove the Reflexive ACL Configuration
```
interface GigabitEthernet0/0
 no ip access-group OUTBOUND-TRACK out
interface GigabitEthernet0/1
 no ip access-group OUTSIDE-IN in
```

### Task 10 — Define Security Zones
```
zone security INSIDE-ZONE
zone security OUTSIDE-ZONE
```

### Task 11 — Assign Interfaces to Zones
```
interface GigabitEthernet0/0
 zone-member security INSIDE-ZONE
interface GigabitEthernet0/1
 zone-member security OUTSIDE-ZONE
```
> Once an interface is a zone member, **all** traffic through it is denied by default unless a policy explicitly permits it — including traffic between two interfaces in the *same* zone, unless that specific same-zone traffic is also addressed. This is a significant behavioral shift from plain ACLs, where nothing is filtered unless you explicitly apply a `deny`.

### Task 12 — Define Traffic of Interest (Class Map)
```
class-map type inspect match-any INSIDE-TO-OUTSIDE-TRAFFIC
 match protocol tcp
 match protocol udp
 match protocol icmp
```

### Task 13 — Define the Inspection Policy
```
policy-map type inspect INSIDE-TO-OUTSIDE-POLICY
 class type inspect INSIDE-TO-OUTSIDE-TRAFFIC
  inspect
 class class-default
  drop
```
> The `inspect` action is what makes this genuinely stateful — R1 will track real connection state for matched traffic and automatically permit legitimate return traffic, without needing any separate reflexive or established-based rule at all.

### Task 14 — Create the Zone Pair and Apply the Policy
```
zone-pair security INSIDE-TO-OUTSIDE source INSIDE-ZONE destination OUTSIDE-ZONE
 service-policy type inspect INSIDE-TO-OUTSIDE-POLICY
```
> Note the zone pair is **directional** — this specific pair governs traffic originating from INSIDE-ZONE toward OUTSIDE-ZONE. There is deliberately **no** corresponding OUTSIDE-ZONE → INSIDE-ZONE zone pair configured in this lab, meaning any connection attempt originating from the outside has no policy permitting it at all, and is dropped by the zone's default-deny behavior from Task 11.

### Task 15 — Verify Outbound-Initiated Traffic Works
```
! From PC1
curl http://203.0.113.10
```
Should succeed.

### Task 16 — Verify Inbound-Initiated Traffic Is Still Blocked
```
! From SRV1
curl http://192.168.10.10
```
Should fail — with no OUTSIDE-ZONE → INSIDE-ZONE zone pair defined, the zone's default-deny posture blocks this entirely, exactly as intended.

### Task 17 — Inspect Real Session State
```
show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE sessions
show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE
```
Confirm this shows **actual tracked session information** (source/destination, protocol, state) for PC1's connection — genuinely richer and more accurate than either the `established`-flag workaround or even the reflexive ACL's simpler temporary-entry approach.

### Task 18 — Final Verification
```
show zone security
show zone-pair security
show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE
show run | section zone
show run | section class-map
show run | section policy-map
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — plain deny-all ACL | Outbound connection attempt fails entirely, since reply traffic is also blocked |
| Task 4 — `established` workaround | Connection succeeds, but is flagged in lab notes as a weak, spoofable control |
| Task 7 — reflexive ACL | Connection succeeds; temporary session-specific entry visible in `show ip access-lists REFLECT-SESSIONS` |
| Task 8 — reflexive ACL, unsolicited inbound | Blocked, confirming reflexive ACLs don't permit new inbound-initiated sessions |
| Task 15 — ZBFW, outbound-initiated | Succeeds |
| Task 16 — ZBFW, inbound-initiated | Blocked, with no explicit rule needed beyond the zone's default-deny |
| Task 17 — session inspection | Shows real, detailed tracked session state |

---

## Challenge (Optional)
- Add a **second inside host** and a class-map/policy-map refinement that only allows HTTP/HTTPS (not all TCP/UDP/ICMP) from the inside zone outward, demonstrating that ZBFW policies can be as granular as an extended ACL while still being genuinely stateful.
- Configure a **DMZ zone** (a third zone) hosting a server that should be reachable from the outside on specific ports only, while still being restricted from initiating connections back into the inside zone — a three-zone design much closer to a real enterprise firewall deployment than this lab's simple two-zone setup.
- Write a short comparison table in your lab notes covering all three approaches from this lab (plain ACL/`established`, reflexive ACL, ZBFW) across: whether it's genuinely stateful, configuration complexity, visibility into active sessions, and default-permit vs. default-deny behavior once applied.