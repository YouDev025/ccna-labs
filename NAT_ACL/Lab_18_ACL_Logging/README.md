# Lab: ACL Logging

## Objective
Practice configuring and verifying **ACL logging** on a Cisco IOS router — using the `log` and `log-input` keywords on access-list entries to record matches to the console/buffer/syslog, understanding the performance and rate-limiting implications, and distinguishing `log` from `log-input`.

---

## Topology

```
                          +-----------+
                          |   SRV1    |   Web/SSH server
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
      192.168.10.1/24  |               |  192.168.10.1... (see table)
                        |
                      SW-A
                +-------+-------+
                |               |
           +---------+     +---------+
           |  PC1    |     |  PC2    |
           | .10     |     | .11     |
           |(attacker|     |(normal  |
           | sim)    |     | user)   |
           +---------+     +---------+
```

- **R1** filters and logs traffic between the Users subnet and the DMZ server.
- **PC1** simulates a host that generates traffic that should be **denied and logged** (e.g., a port scan or SSH attempt against the server).
- **PC2** simulates a normal user whose permitted traffic is also logged, to compare log volume/behavior between `permit` and `deny` logging.
- **SRV1** is the DMZ server being protected/monitored.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       | Default Gateway |
|--------|-----------|-------------------|--------------------|-----------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0      | —               |
| R1     | Gi0/2     | 192.168.40.1       | 255.255.255.0      | —               |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0      | 192.168.10.1    |
| PC2    | NIC       | 192.168.10.11       | 255.255.255.0      | 192.168.10.1    |
| SRV1   | NIC       | 192.168.40.20       | 255.255.255.0      | 192.168.40.1    |

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW-A, SW-DMZ, PC1, PC2, and SRV1 as shown above.
2. Cable R1 Gi0/0 to SW-A (PC1, PC2), and R1 Gi0/2 to SW-DMZ (SRV1).
3. Apply the addressing from the table above.
4. Verify baseline connectivity with **no ACL applied**: PC1 and PC2 can both reach SRV1 on all ports.

### Task 2 — Enable Logging Infrastructure
Before testing ACL logging, make sure logging is actually visible somewhere useful:
```
logging buffered 16384 debugging
logging console informational
service timestamps log datetime msec
```
> `logging buffered` keeps recent log messages in RAM (`show logging` to view them) — very useful in a lab without a syslog server. In production you would typically also configure `logging host <syslog-server-ip>`.

### Task 3 — Build an ACL with `deny ... log`
Create an ACL that denies SSH from PC1 to SRV1 and logs each match, while allowing everything else:
```
ip access-list extended DMZ-LOGGING
 deny   tcp host 192.168.10.10 host 192.168.40.20 eq 22 log
 permit ip any any
```
Apply it inbound on the Users-facing interface:
```
interface GigabitEthernet0/0
 ip access-group DMZ-LOGGING in
```

### Task 4 — Generate Denied Traffic and Observe Logging
From **PC1**, attempt SSH to SRV1 (192.168.40.20) several times.

On R1:
```
show logging
```
You should see messages similar to:
```
%SEC-6-IPACCESSLOGP: list DMZ-LOGGING denied tcp 192.168.10.10(random port) -> 192.168.40.20(22), 1 packet
```

### Task 5 — Understand Log Rate Limiting
Generate a burst of denied traffic (e.g., a quick loop of SSH attempts or a simple port scan tool from PC1) and observe that IOS does **not** log every single packet individually once a `deny` entry is matched repeatedly in a short window — it aggregates matches and logs a summary count periodically (default behavior, roughly every 5 minutes per unique flow, or governed by the platform's logging rate limiter). Note this behavior in your lab notes; it's an important operational detail — heavy log volume from ACL matches can itself become a DoS vector against the router's CPU if unmanaged, which is exactly why this rate limiting exists.

Optionally inspect/adjust the global ACL logging rate limiter (platform-dependent command, verify availability on your IOS version):
```
ip access-list logging interval 10
```

### Task 6 — Compare `log` vs `log-input`
Modify the ACL to use `log-input` instead of `log` on the same line, which additionally records the **ingress interface and source MAC address** (Layer 2 detail) of the matching packet — useful for tracing exactly where on the LAN an offending host is connected:
```
ip access-list extended DMZ-LOGGING
 no deny tcp host 192.168.10.10 host 192.168.40.20 eq 22 log
 deny   tcp host 192.168.10.10 host 192.168.40.20 eq 22 log-input
 permit ip any any
```
Regenerate the denied SSH traffic from PC1 and compare the new log entries — they should now include the ingress interface (e.g., `GigabitEthernet0/0`) and PC1's MAC address, which plain `log` does not provide.

### Task 7 — Add Logging to a `permit` Entry for Comparison
Add a `permit ... log` entry for PC2's HTTP traffic to SRV1, so you can compare permit-logging volume/behavior against deny-logging:
```
ip access-list extended DMZ-LOGGING
 15 permit tcp host 192.168.10.11 host 192.168.40.20 eq 80 log
```
> Logging `permit` entries is far noisier in production (every successful connection is logged) and is generally reserved for specific audit/compliance needs rather than blanket use — this task is here so you can directly observe and compare that volume difference, not because it's a best practice to log all permits.

Generate HTTP traffic from PC2 to SRV1 and observe the corresponding `%SEC-6-IPACCESSLOGP` (or platform-equivalent) messages logged for the **permitted** connection as well.

### Task 8 — Verify
```
show logging
show access-lists DMZ-LOGGING
show ip interface GigabitEthernet0/0 | include access list
show run | section access-list
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| PC1 → SRV1 SSH attempts | Denied; `show logging` shows `%SEC-6-IPACCESSLOGP` deny entries |
| `show access-lists DMZ-LOGGING` | Match/hit counters increment on the `deny ... log` line each time PC1 attempts SSH |
| Rapid repeated PC1 SSH attempts | Log entries show periodic **aggregated counts**, not one line per packet, confirming rate limiting |
| ACL updated to `log-input` | New log entries include ingress interface and PC1's source MAC address |
| PC2 → SRV1 HTTP with `permit ... log` | Succeeds, and is still logged — confirming `log` works on `permit` lines too, not just `deny` |
| PC2 → SRV1 SSH (not explicitly logged) | Still permitted by the trailing `permit ip any any`, but does **not** generate a logged match, since only the specific `eq 22`/`eq 80` lines have `log`/`log-input` attached |

---

## Challenge (Optional)
- Configure `logging host <syslog-server-ip>` (using a simulated syslog server if available in your lab environment) and confirm the same ACL log messages arrive at the external syslog server instead of only the local buffer.
- Compare CPU load (`show processes cpu | include IP Input` or similar) during a sustained burst of logged denies versus the same burst against an identical ACL **without** `log`, to observe the real performance cost of ACL logging under load.
- Design and implement a `deny ... log` rule that specifically flags a simulated port-scan pattern (e.g., a range of destination ports) and document how the aggregated logging behavior changes when many different ports/flows are being denied simultaneously versus one repeated single flow.