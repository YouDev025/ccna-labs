# Lab: Syslog Analysis

## Objective
This lab is about **reading and interpreting** syslog output, not configuring logging from scratch (see the Syslog Configuration lab for that). You will generate a realistic sequence of events across two devices, then use IOS log-filtering tools (`show logging | include`, `| begin`, `| section`) to isolate relevant messages, decode Cisco's syslog message format, and **correlate timestamps across two different devices** to reconstruct what actually happened during a simulated incident — a core skill for real troubleshooting and security investigation.

---

## Topology

```
                          +-----------+
                          |   SRV1    |   192.168.95.20/24
                          +-----------+
                                |
                              SW1        (also generates its own logs)
                                |
                        Gi0/1   |  192.168.95.1/24
           +---------------+
           |      R1       |
           +---------------+
     Gi0/0  |
             |
      +---------+
      |  PC1    |   192.168.10.10/24  (used to generate both legitimate and "suspicious" traffic)
      +---------+
```

- **R1** and **SW1** both generate log messages that you'll need to read together to fully understand what happened — a single device's logs are often only half the picture.
- **PC1** generates the traffic/events that produce the log entries you'll analyze.
- **SRV1** is the asset involved in the simulated incident.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1     | Gi0/1     | 192.168.95.1       | 255.255.255.0        |
| SRV1   | NIC       | 192.168.95.20       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology and Prepare Logging
1. Place R1, SW1, SRV1, and PC1 as shown, and apply the addressing above.
2. On R1:
   ```
   service timestamps log datetime msec
   service sequence-numbers
   logging buffered 32768 debugging
   ```
3. On SW1 (basic switch logging):
   ```
   service timestamps log datetime msec
   logging buffered 32768 debugging
   ```

### Task 2 — Generate a Realistic Sequence of Events
Perform the following actions in order, **pausing about 30–60 seconds between each** so the timestamps are meaningfully spread out and easier to correlate later:

1. On **SW1**, shut down and re-enable the port connecting to PC1 (simulating a cable reseat or a device reboot):
   ```
   interface <PC1's switchport>
    shutdown
    no shutdown
   ```
2. On **R1**, add a temporary restrictive ACL and apply it, simulating a new (possibly unauthorized) firewall rule change:
   ```
   ip access-list extended TEMP-INCIDENT-TEST
    deny tcp host 192.168.10.10 host 192.168.95.20 eq 22 log
    permit ip any any
   interface GigabitEthernet0/0
    ip access-group TEMP-INCIDENT-TEST in
   ```
3. From **PC1**, attempt SSH to SRV1 **three separate times** (each should fail due to the ACL) — this simulates a pattern that could represent either a legitimate admin being unexpectedly blocked, or a suspicious repeated access attempt; part of your analysis task is figuring out which interpretation the evidence actually supports.
4. On **R1**, remove the ACL, simulating the rule being rolled back:
   ```
   interface GigabitEthernet0/0
    no ip access-group TEMP-INCIDENT-TEST in
   ```
5. From **PC1**, attempt SSH to SRV1 one more time — this one should now succeed (or at least no longer be blocked by R1's ACL).

### Task 3 — Pull the Raw Logs
```
! On R1
show logging

! On SW1
show logging
```
Copy both outputs into your lab notes as your "raw evidence" — you'll be filtering and referencing them in the tasks below.

### Task 4 — Filter for a Specific Device or Address
Use `include` to isolate only messages mentioning PC1's address:
```
show logging | include 192.168.10.10
```
Confirm this surfaces the three failed SSH attempts (and the later successful one) without the unrelated interface-flap noise from SW1's port event.

### Task 5 — Filter for a Specific Message Type
Use `include` to isolate only the ACL-related security log messages:
```
show logging | include %SEC-6-IPACCESSLOGP
```
> This message code specifically indicates an IP access-list logged match — knowing key Cisco syslog **mnemonics** like this (facility-severity-mnemonic, e.g. `%SEC-6-IPACCESSLOGP`, `%LINK-3-UPDOWN`, `%SYS-5-CONFIG_I`) is a core analysis skill, since it lets you jump straight to the category of event you're investigating instead of reading every line.

### Task 6 — Use `begin` to Jump to a Point in Time
Find the timestamp of the interface flap event from Task 2, Step 1, then use it to view everything **from that point forward**:
```
show logging | begin <HH:MM:SS from the interface event>
```
Confirm this shows the interface event and everything chronologically after it, letting you review the sequence of events without scrolling past earlier, irrelevant history.

### Task 7 — Correlate Timestamps Across R1 and SW1
This is the core exercise of the lab. Using the timestamps from both devices' logs (Task 3), build a single combined timeline in your lab notes, e.g.:

| Time | Device | Event |
|---|---|---|
| (fill in) | SW1 | Interface to PC1 flapped down/up |
| (fill in) | R1 | ACL `TEMP-INCIDENT-TEST` applied to Gi0/0 |
| (fill in) | R1 | First denied SSH attempt from PC1 to SRV1 logged |
| (fill in) | R1 | Second denied SSH attempt logged |
| (fill in) | R1 | Third denied SSH attempt logged |
| (fill in) | R1 | ACL removed from Gi0/0 |
| (fill in) | R1 | SSH attempt no longer blocked |

> Notice that no single device's log tells this whole story — SW1 only knows about its own port, and R1 only knows about its own ACL matches. Reconstructing the full picture requires combining both sources in chronological order, which is exactly what a real investigation (or a SIEM correlating multiple log sources automatically) has to do.

### Task 8 — Interpret the Evidence
Answer the following in your lab notes, using only the evidence from your timeline:
1. Is the interface flap on SW1 related to the SSH denies on R1, or are they two unrelated events that merely happened close together in time? Justify your answer using the actual timestamps and addresses involved, not assumption.
2. Do the three failed SSH attempts, on their own, look more consistent with a legitimate user retrying a blocked connection, or with automated/scripted access attempts? What additional evidence (not present in this lab) would help you tell the difference in a real investigation (e.g., timing interval between attempts, source diversity, time of day)?
3. Was the ACL removal in Task 2 Step 4 itself logged anywhere (check for a `%SYS-5-CONFIG_I` or similar configuration-change message)? If configuration changes aren't clearly logged with **who** made them, what would you recommend adding to this setup (hint: connect this back to the SNMP/AAA or a dedicated configuration-change logging mechanism)?

### Task 9 — Verify
```
show logging | include %SEC-6-IPACCESSLOGP
show logging | include %LINK
show logging | include %SYS-5-CONFIG_I
show run | section access-list
show run | section logging
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show logging` on R1 and SW1 | Both contain timestamped, sequence-numbered entries for every event generated in Task 2 |
| `show logging \| include 192.168.10.10` | Surfaces exactly the SSH-related entries, filtering out unrelated noise |
| `show logging \| include %SEC-6-IPACCESSLOGP` | Surfaces exactly the three denied SSH attempts, and nothing else |
| `show logging \| begin <timestamp>` | Correctly starts output at or after the specified event |
| Combined timeline (Task 7) | Chronologically consistent between both devices — no event from R1 should be placed before an SW1 event that logically must have preceded it (e.g., PC1 couldn't send traffic before its port came back up) |
| Task 8 answers | Reference specific timestamps/log lines rather than general assumptions |

---

## Challenge (Optional)
- Repeat this exercise using a **remote syslog server** (from the Syslog Configuration lab) instead of local buffers, and use the server's own search/filter tools (rather than IOS's `include`/`begin`) to perform the same correlation — compare which approach was easier and why centralized logging tools generally scale better than reading local buffers device-by-device.
- Deliberately introduce a **clock skew** between R1 and SW1 (set their clocks a few minutes apart without NTP) before generating the event sequence, then attempt the same correlation exercise — document how much harder (or outright misleading) timeline reconstruction becomes without synchronized time, tying this lab directly back into why the NTP lab matters.
- Research and briefly describe how a real SIEM (Security Information and Event Management) platform automates the correlation task you did manually in Task 7, including the concept of alerting rules that would flag "repeated denied SSH attempts followed by a permitted one" as a pattern worth investigating automatically.