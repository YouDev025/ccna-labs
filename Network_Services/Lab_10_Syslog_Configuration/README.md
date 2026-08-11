# Lab: Syslog Configuration

## Objective
Configure and verify **centralized syslog logging** on Cisco IOS routers: set logging severity levels correctly, send logs to a remote syslog server, add timestamps and sequence numbers for accurate correlation, and confirm log messages actually arrive and are readable.

---

## Topology

```
                          +-----------+
                          |  SYSLOG1  |   Syslog server
                          |192.168.70.10/24|
                          +-----------+
                                |
                              SW1
                                |
                        Gi0/1   |  192.168.70.1/24
           +---------------+
           |      R1       |
           +---------------+
     Gi0/0  |
             |
           SW-LAN
             |
      +---------+
      |  PC1    |   Used to generate test traffic/events
      |192.168.10.10/24|
      +---------+
```

- **R1** is the device being monitored — it will send its log messages to **SYSLOG1**.
- **SYSLOG1** represents a syslog collector (in real deployments this would be a dedicated tool like a SIEM, rsyslog server, or Kiwi Syslog; in this lab it can be simulated with any device capable of receiving UDP 514 syslog messages, or by using R1's own local buffer if a real collector isn't available in your lab environment).
- **PC1** is used to generate events on R1 (e.g., triggering ACL denies, interface changes) so there's something meaningful to log.

---

## IP Addressing Table

| Device  | Interface | IP Address       | Subnet Mask       |
|---------|-----------|-------------------|---------------------|
| R1      | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1      | Gi0/1     | 192.168.70.1       | 255.255.255.0        |
| SYSLOG1 | NIC       | 192.168.70.10       | 255.255.255.0        |
| PC1     | NIC       | 192.168.10.10       | 255.255.255.0        |

---

## Background: Syslog Severity Levels
Cisco IOS uses eight standard severity levels, 0 (most severe) through 7 (least severe). Understanding these is essential before configuring logging — setting the wrong level either floods you with noise or hides the events you actually need:

| Level | Name | Example |
|---|---|---|
| 0 | Emergencies | System unusable |
| 1 | Alerts | Immediate action needed |
| 2 | Critical | Critical conditions |
| 3 | Errors | Error conditions |
| 4 | Warnings | Warning conditions |
| 5 | Notifications | Normal but significant (e.g., interface up/down) |
| 6 | Informational | Informational messages (e.g., ACL logging, config changes) |
| 7 | Debugging | Debug-level messages |

> Setting a logging level to, say, `informational` (6) includes everything from 0–6. It does **not** mean "only level 6" — each level includes all levels more severe than itself.

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, SW-LAN, SYSLOG1, and PC1 as shown.
2. Apply the addressing above and confirm basic connectivity: R1 can ping SYSLOG1, and PC1 can ping R1.

### Task 2 — Configure Local Logging Basics First
Before sending anything remote, make sure local logging is sane and timestamped — this also helps you compare local buffer output against what arrives at SYSLOG1 later:
```
service timestamps log datetime msec localtime show-timezone
service sequence-numbers
logging buffered 16384 informational
```
> `service timestamps` adds accurate date/time to every message (without it, messages only show relative uptime, which is nearly useless for correlating events later). `service sequence-numbers` adds a counter to each message, which helps detect if any messages were lost/dropped, especially over UDP-based remote logging.

### Task 3 — Configure Remote Syslog Logging
```
logging host 192.168.70.10
logging trap informational
logging source-interface GigabitEthernet0/1
logging on
```
> `logging trap` sets the severity threshold for messages sent to the **remote** server — this can be set independently from the local buffer level configured in Task 2. `logging source-interface` ensures the syslog packets are always sourced from a predictable, consistent address (important if R1 has multiple interfaces/paths to the server) rather than whichever interface the routing table happens to pick.

### Task 4 — Generate Log Events
Trigger a few different types of log messages so you have varied severity levels to observe:
1. Shut down and re-enable an unused interface to generate a **link-status** (severity 5, notification) event:
   ```
   interface GigabitEthernet0/2
    shutdown
    no shutdown
   ```
2. Make a small, harmless configuration change (severity 5) to generate a **config change** log entry, e.g.:
   ```
   ip access-list standard TEST-LOGGING-ACL
    permit any
   no ip access-list standard TEST-LOGGING-ACL
   ```
3. If you completed the ACL Logging lab, trigger a `deny ... log` match from PC1 for a severity 6 event, or add one now:
   ```
   ip access-list extended TEMP-DENY-TEST
    deny icmp host 192.168.10.10 host 192.168.70.1 log
    permit ip any any
   interface GigabitEthernet0/0
    ip access-group TEMP-DENY-TEST in
   ```
   From PC1, ping R1's Gi0/1 (192.168.70.1) to trigger the denied/logged match, then remove the ACL afterward.

### Task 5 — Verify Locally on R1
```
show logging
```
Confirm you see timestamped, sequence-numbered entries for each event generated in Task 4.

### Task 6 — Verify Remotely on SYSLOG1
Check that the same (or equivalent) messages arrived at the syslog server. Depending on your syslog server software, this might be a live message viewer, a log file, or a web dashboard — confirm:
- Messages appear with the correct source IP (R1's Gi0/1 address, per Task 3's `logging source-interface`).
- Timestamps are present and reasonably close to R1's local clock (small delay is normal).
- Severity levels are correctly represented.

### Task 7 — Adjust Severity Threshold and Observe the Difference
Lower the remote trap level to only send warnings and more severe events, filtering out the routine notifications from Task 4:
```
logging trap warnings
```
Trigger the interface flap from Task 4 again — this time it should **not** appear at SYSLOG1 (severity 5 is now below the new threshold of 4/warnings), even though it still appears in the local buffer (assuming the local buffered level from Task 2 remains at `informational`). This demonstrates that local and remote logging levels are independent settings.

### Task 8 — Verify
```
show logging
show running-config | section logging
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show logging` on R1 | Shows buffered messages with accurate date/time and sequence numbers |
| Interface flap event | Appears in local buffer; appears at SYSLOG1 while trap level ≥ notifications (5) |
| Config change event | Appears in local buffer and at SYSLOG1 (severity 5) |
| ACL deny-and-log event | Appears in local buffer and at SYSLOG1 (severity 6, informational) |
| Messages at SYSLOG1 | Source IP matches R1's Gi0/1 address specifically, confirming `logging source-interface` is working |
| After lowering trap level to `warnings` (Task 7) | New interface flap events stop appearing at SYSLOG1, but still appear in R1's local buffer — confirming local and remote severity thresholds operate independently |
| `show running-config \| section logging` | Confirms `logging host`, `logging trap`, `logging source-interface`, `service timestamps`, and `service sequence-numbers` are all present |

---

## Challenge (Optional)
- Configure a second syslog server address as a backup destination (`logging host <second-ip>`) and confirm R1 sends the same messages to both simultaneously.
- Research and document (implementing if your platform/syslog server supports it) how **syslog over TLS** or a **reliable delivery transport** differs from the default UDP-based syslog used in this lab, and why that distinction matters for compliance-sensitive environments.
- Using the ACL Logging lab's rate-limiting concept, generate a sustained burst of denied traffic while remote logging is active, and compare the message volume/pattern that arrives at SYSLOG1 against what you'd expect if every single packet were logged individually — tying the two labs' concepts together.