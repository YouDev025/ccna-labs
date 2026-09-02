# Lab: Security Logging

## Objective
This lab addresses a specific gap the Syslog Analysis lab flagged but couldn't close: standard `%SEC-6-IPACCESSLOGP`-style logging tells you traffic was denied, and a plain `%SYS-5-CONFIG_I` message tells you *that* a configuration change happened — but neither reliably tells you **who** made a change. This lab configures Cisco IOS's **configuration change auditing** (the `archive log config` feature, tied to AAA identity) so every configuration change is attributed to a specific authenticated user, and consolidates that with **authentication event logging** into a single security-focused audit trail.

---

## Topology

```
                          +-----------+
                          |    R1     |
                          +-----------+
                        Gi0/0  |  192.168.10.1/24
                                |
                              SW1
                    +----------+----------+
                    |                     |
              +-----------+         +-----------+
              |   PC1     |         |   PC2     |
              |(admin,    |         |(second    |
              | user:     |         | admin,    |
              | alice)    |         | user: bob)|
              +-----------+         +-----------+
```

- **R1** is the device being audited.
- **PC1** and **PC2** represent two different administrators, logging in with **distinct** individual accounts — this is a prerequisite for the auditing this lab configures to actually be meaningful; a single shared admin account defeats the entire purpose, since every log entry would show the same identity regardless of who actually made a change.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |
| PC2    | NIC       | 192.168.10.11       | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology and Prerequisites
```
hostname R1
ip domain-name lab.local
crypto key generate rsa modulus 2048
username alice privilege 15 secret AlicePass123
username bob privilege 15 secret BobPass456
line vty 0 4
 transport input ssh
 login local
```
> Note two **distinct** named accounts (`alice`, `bob`) rather than a single shared `admin` account — this distinction is the entire foundation this lab's auditing depends on.

### Task 2 — Enable Basic Logging Infrastructure
```
service timestamps log datetime msec
service sequence-numbers
logging buffered 32768 informational
```

### Task 3 — Confirm the Baseline Gap (Before Auditing)
```
! From PC1, logged in as alice
```
```
! On R1, make a small test change
ip access-list standard TEST-BASELINE-ACL
 permit any
```
```
show logging | include CONFIG_I
```
Confirm you see a generic `%SYS-5-CONFIG_I: Configured from console/vty by alice on ...` message — note that on many IOS versions, this **does** actually include the username already if AAA/local login was used, which is better than nothing, but it is a single terse line with no detail about **what** was actually changed, only that *something* was. Remove the test ACL:
```
no ip access-list standard TEST-BASELINE-ACL
```

### Task 4 — Enable the Archive Feature
```
archive
 log config
  logging enable
  logging size 200
  hidekeys
  notify syslog
```
> `hidekeys` prevents sensitive values (like passwords typed as part of a configuration line) from being stored in the archive log in plaintext — important, since the archive log itself becomes a sensitive artifact once enabled. `notify syslog` sends archive log events to the standard logging system as well, so they show up alongside your other syslog messages rather than only being visible via dedicated archive commands.

### Task 5 — Make a Test Configuration Change as Alice
```
! From PC1, logged in as alice
```
```
! On R1
ip access-list standard TEST-ARCHIVE-ACL
 permit host 192.168.10.99
```

### Task 6 — Make a Different Test Change as Bob
```
! From PC2, logged in as bob
```
```
! On R1
ip access-list standard TEST-ARCHIVE-ACL
 permit host 192.168.10.100
```

### Task 7 — Review the Detailed Archive Log
```
show archive log config all
```
Confirm this shows a **much more detailed** record than the basic CONFIG_I message from Task 3 — specifically, the **exact commands** entered, attributed individually to **alice** and **bob** respectively, with timestamps for each. This is the direct answer to the accountability gap identified earlier: not just "a change happened," but precisely what was typed and by whom.

### Task 8 — Compare Archive Log to Plain Running-Config
```
show run | section ip access-list standard TEST-ARCHIVE-ACL
```
Note the running config alone shows the **current end state** (both permit lines, merged together) with no indication of who added which line or in what order — confirming that the archive log provides genuinely additional forensic value beyond what `show run` alone could ever tell you.

### Task 9 — Test a Change That Should Be Especially Well-Audited: Removing a Security Control
```
! From PC1, logged in as alice — simulate removing a security-relevant ACL
no ip access-list standard TEST-ARCHIVE-ACL
```
```
show archive log config all
```
Confirm this removal is attributed specifically to alice, with the exact command logged — precisely the kind of event a security review would most want clear attribution for (e.g., "who disabled this control, and when").

### Task 10 — Consolidate with Authentication Event Logging
Combine the configuration-change auditing from this lab with login event visibility (building on the Banner and Login lab's `login block-for` feature, which already logs failures):
```
login on-success log
login on-failure log
```
```
! From PC2, attempt a login with a deliberately wrong password
```
```
show logging | include login
```
Confirm both successful and failed login attempts are now explicitly logged with source address and username attempted — giving you, alongside the archive log's "who changed what," a matching "who logged in (or tried to) and when" record.

### Task 11 — Build a Consolidated Security Timeline
Using `show archive log config all` and `show logging | include login`, reconstruct a combined timeline in your lab notes covering: who logged in, when, and what configuration changes each person made during their session — directly applying the cross-source timeline-correlation skill from the Syslog Analysis lab, but now with genuinely richer, per-user-attributed data instead of generic device-level events.

### Task 12 — Final Verification
```
show archive log config all
show archive log config record 1   (or any specific record number, to see full detail on one entry)
show run | section archive
show run | include login on-
show logging
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — before archive logging | Generic `%SYS-5-CONFIG_I` message present, minimal detail |
| Task 7 — after archive logging | Detailed, per-command, per-user log entries for both alice's and bob's changes |
| Task 8 — archive log vs. running-config | Running-config alone cannot show who added which line; archive log can |
| Task 9 — security-relevant removal | Attributed specifically to alice, with the exact removal command logged |
| Task 10 — login event logging | Both successful and failed login attempts appear in `show logging`, with username and source address |
| Task 11 — consolidated timeline | Produces a coherent, per-user-attributed record of both authentication and configuration-change events |

---

## Challenge (Optional)
- Configure `logging host <syslog-server-ip>` (from the Syslog Configuration lab) alongside `notify syslog` in the archive feature, and confirm the detailed, per-user configuration-change records arrive at a centralized syslog server, not just the local buffer — this is how a real SOC/security team would actually consume this data at scale, rather than checking each device individually.
- Research and briefly describe **AAA command accounting** (from the AAA Configuration lab: `aaa accounting commands 15`) as an alternative/complementary way to achieve similar per-user command attribution, sent directly to a RADIUS/TACACS+ server rather than the local archive log — compare the two approaches' strengths (e.g., archive log works without any AAA server at all; AAA accounting centralizes across many devices without relying on each one's local archive).
- Using this lab's alice/bob distinction as a cautionary example, write a short policy statement (a few sentences) explaining why shared administrative accounts should be avoided in any environment where accountability or auditability matters, referencing specifically how this lab's Task 7 verification would have been impossible to produce meaningfully with a single shared account instead.