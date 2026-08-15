# Lab: DNS Security

## Objective
Harden a router-based DNS service against common real-world DNS abuse patterns: prevent the server from acting as an **open resolver** (a major source of DNS amplification DDoS attacks), restrict **who is allowed to query it**, implement a simple **DNS sinkhole** to block known-malicious domains, and enable **logging** so query activity can be reviewed. This lab builds directly on the DNS lab — if you haven't done that one, review its DNS server basics first, since this lab assumes that foundation and focuses entirely on securing it.

---

## Topology

```
                         "Internet" (simulated)
                               |
                        Gi0/1  |  203.0.113.1/29
                       +---------------+
                       |      R1       |   (DNS server + firewall-style filtering)
                       +---------------+
                        Gi0/0  |  192.168.90.1/24
                               |
                             SW1
                    +----------+----------+
                    |                     |
              +-----------+         +-----------+
              |   PC1     |         |  SRV1     |
              |(internal, |         |(internal, |
              | trusted)  |         | web server)|
              +-----------+         +-----------+

              +-----------+
              |  ATTACKER |   simulates an external host on the "Internet" side
              |203.0.113.2/29|
              +-----------+
```

- **R1** is both the DNS server for the internal LAN and the device you'll harden.
- **PC1** and **SRV1** represent legitimate internal clients that should always be able to resolve names.
- **ATTACKER** simulates an external host attempting to query R1's DNS service from the outside — representing both a malicious actor and, more broadly, the general internet population that should never be able to use an internal DNS server as an open resolver.

---

## IP Addressing Table

| Device    | Interface | IP Address       | Subnet Mask       |
|-----------|-----------|-------------------|---------------------|
| R1        | Gi0/0     | 192.168.90.1       | 255.255.255.0        |
| R1        | Gi0/1     | 203.0.113.1         | 255.255.255.248      |
| ATTACKER  | NIC       | 203.0.113.2         | 255.255.255.248      |
| PC1       | NIC       | 192.168.90.10       | 255.255.255.0        |
| SRV1      | NIC       | 192.168.90.20       | 255.255.255.0        |

---

## Background: Why DNS Servers Need Hardening
An **open resolver** is a DNS server that answers recursive queries from *any* source on the internet, not just its intended clients. Attackers abuse open resolvers for **DNS amplification attacks**: they send a small, spoofed-source-IP query that generates a much larger response, which gets reflected toward a victim — the victim is flooded with traffic they never requested, and your server becomes an unwitting participant. This is why the single most important DNS security control in this lab is simply **restricting who can query the server at all**, before anything else.

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, PC1, SRV1, and ATTACKER as shown.
2. Apply the addressing above.
3. Add a route so ATTACKER can reach R1's outside interface for testing (a default route toward R1, or a static route back if you're simulating a fuller Internet segment).
4. Verify baseline reachability: PC1 and SRV1 can ping R1's inside interface; ATTACKER can ping R1's outside interface.

### Task 2 — Configure Basic DNS Service (Baseline, Intentionally Insecure)
```
ip dns server
ip host srv1.lab.local 192.168.90.20
ip host r1.lab.local 192.168.90.1
```
At this point, by default, R1 will answer DNS queries from **any** source that can reach it on UDP/53 — including ATTACKER. Confirm this insecure baseline before fixing it:
```
! From ATTACKER
nslookup srv1.lab.local 203.0.113.1
```
This should currently **succeed**, demonstrating the open-resolver problem you're about to fix.

### Task 3 — Restrict DNS Queries to Internal Sources Only
Build an ACL matching only the internal subnet, and apply it to control DNS access:
```
access-list 50 permit 192.168.90.0 0.0.0.255
ip dns server
```
> Depending on your IOS version, DNS query source restriction may be implemented via `ip dns server` in combination with a general interface ACL (Task 4) rather than a DNS-specific access-group command. This lab uses the interface ACL approach, which is both the most broadly-supported method and mirrors how you'd actually protect any UDP/53 service on a router acting as a basic DNS responder.

### Task 4 — Block Inbound DNS Queries on the Outside Interface
```
ip access-list extended OUTSIDE-IN
 deny   udp any any eq 53 log
 deny   tcp any any eq 53 log
 permit icmp any host 203.0.113.1 echo-reply
 permit ip any any
```
```
interface GigabitEthernet0/1
 ip access-group OUTSIDE-IN in
```
> This denies **any** DNS query (UDP or TCP port 53) arriving from the outside interface, regardless of source — since there should never be a legitimate reason for an external host to query this internal DNS server directly. The `log` keyword lets you observe blocked attempts, tying back into the ACL Logging lab.

### Task 5 — Retest from ATTACKER
```
! From ATTACKER
nslookup srv1.lab.local 203.0.113.1
```
This should now **fail** (timeout), confirming R1 is no longer answering external DNS queries.

### Task 6 — Retest from Internal Clients
```
! From PC1
nslookup srv1.lab.local 192.168.90.1
```
This should still **succeed** — internal clients are unaffected, since the ACL only filters the outside interface, and DNS queries from PC1 never cross it.

### Task 7 — Implement a Simple DNS Sinkhole
A DNS sinkhole redirects known-malicious domain names to a non-routable or walled-garden address instead of their real (attacker-controlled) address, preventing infected/compromised internal hosts from reaching command-and-control infrastructure by name. Simulate this with a static host entry pointing a "known-bad" test domain to an unreachable sinkhole address:
```
ip host malicious-test.example 192.0.2.1
```
> `192.0.2.0/24` is a documentation/test range (RFC 5737) that should never be reachable — using it here simulates a sinkhole without needing a real walled-garden server in the lab.

### Task 8 — Verify the Sinkhole
```
! From PC1
nslookup malicious-test.example 192.168.90.1
ping malicious-test.example
```
The name resolves (to 192.0.2.1), but the ping fails — exactly the intended sinkhole behavior: the DNS layer prevented the connection from ever reaching a real destination, without needing a firewall rule specific to that destination address.

### Task 9 — Enable Logging for DNS-Related Denies
Confirm blocked external DNS attempts are actually visible for review (building on the Syslog lab if you completed it):
```
logging buffered 16384 informational
service timestamps log datetime msec
```
Regenerate a blocked query from ATTACKER, then check:
```
show logging
```
Confirm a logged deny entry appears for the UDP/53 attempt from ATTACKER's address.

### Task 10 — Verify
```
show ip dns server
show hosts
show access-lists OUTSIDE-IN
show ip interface GigabitEthernet0/1 | include access list
show logging
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 baseline test from ATTACKER | Succeeds (demonstrates the open-resolver problem) |
| ATTACKER query to R1 after Task 4/5 | Fails/times out |
| PC1 query to R1 after Task 4 | Still succeeds — internal resolution unaffected |
| `show access-lists OUTSIDE-IN` | `deny udp any any eq 53 log` and `deny tcp any any eq 53 log` show incrementing match counts after ATTACKER's blocked attempts |
| Sinkhole domain resolution (PC1) | Resolves to 192.0.2.1 |
| Sinkhole domain ping (PC1) | Fails (192.0.2.1 is unreachable by design) |
| `show logging` | Contains a logged deny entry showing ATTACKER's source address attempting UDP/53 |
| `show ip interface GigabitEthernet0/1` | Confirms `OUTSIDE-IN` applied inbound |

---

## Challenge (Optional)
- Add several more sinkholed domains at once (simulating a small blocklist) using multiple `ip host` entries, and discuss in your lab notes why this approach doesn't scale well compared to a real DNS security product (e.g., Cisco Umbrella) that maintains and updates a large, continuously-refreshed threat-intelligence-based blocklist automatically.
- Research and document **DNS response rate limiting (RRL)** — a technique real DNS servers use to reduce their usefulness as an amplification vector even for legitimate-looking queries — and explain why blocking all external queries outright (as done in this lab) is a simpler but less flexible alternative to RRL for a server that's only ever supposed to serve internal clients.
- Combine this lab with the DHCP lab: configure DHCP to hand out R1's internal address as the DNS server, and confirm a freshly-DHCP-addressed client on the internal LAN automatically benefits from both the restricted query access and the sinkhole, without any manual DNS configuration on the client.