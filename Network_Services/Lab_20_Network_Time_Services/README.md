# Lab: Network Time Services

## Objective
This lab treats accurate time as an **enterprise service with dependents**, not just a standalone protocol exercise (see the NTP and NTP Authentication labs for deep protocol-level practice). You will configure **redundant** time sources, correct **time zone and daylight saving** settings, observe **automatic failover** when a preferred source disappears, and confirm that other services which silently depend on accurate time — logging, DHCP lease timestamps, and certificate-adjacent operations — actually benefit from a properly functioning time service.

---

## Topology

```
   +-----------+          +-----------+
   |  R-NTP-A  |          |  R-NTP-B  |    Two independent, redundant time sources
   +-----------+          +-----------+
    10.5.1.1/30            10.5.2.1/30
         \                    /
          \                  /
      10.5.1.0/30      10.5.2.0/30
            \              /
             +------------+
             |  R-CLIENT  |    Uses both sources, prefers one, fails over if needed
             +------------+
              10.5.1.2/30 (to R-NTP-A)
              10.5.2.2/30 (to R-NTP-B)
              Gi0/2: 192.168.99.1/24 (LAN)
                    |
                  SW1
                    |
              +-----------+
              |   PC1     |   Also runs DHCP client — used to confirm lease timestamp accuracy
              +-----------+
```

- **R-NTP-A** and **R-NTP-B** are two independent time references — this models a realistic enterprise design where you never rely on a single NTP source.
- **R-CLIENT** synchronizes to both, with a configured **preference**, and also runs local services (logging, DHCP) whose output you'll check for time accuracy.
- **PC1** is a DHCP client used to demonstrate that lease timestamps are only meaningful if the issuing router's clock is correct.

---

## IP Addressing Table

| Device    | Interface | IP Address       | Subnet Mask       |
|-----------|-----------|-------------------|---------------------|
| R-NTP-A   | Gi0/0     | 10.5.1.1            | 255.255.255.252      |
| R-NTP-B   | Gi0/0     | 10.5.2.1            | 255.255.255.252      |
| R-CLIENT  | Gi0/0     | 10.5.1.2            | 255.255.255.252      |
| R-CLIENT  | Gi0/1     | 10.5.2.2            | 255.255.255.252      |
| R-CLIENT  | Gi0/2     | 192.168.99.1          | 255.255.255.0          |
| PC1       | NIC       | (via DHCP)           | (via DHCP)             |

---

## Tasks

### Task 1 — Build the Topology
1. Place R-NTP-A, R-NTP-B, R-CLIENT, SW1, and PC1 as shown, and apply the addressing above.
2. Confirm R-CLIENT can ping both R-NTP-A and R-NTP-B before configuring NTP.

### Task 2 — Set an Incorrect Baseline Clock and Configure Time Zone / DST
Before syncing anything, configure R-CLIENT's **time zone and daylight saving rules** — these are independent of NTP itself (NTP distributes UTC; the local zone/DST offset is applied on top of it locally) and are commonly forgotten:
```
clock timezone EST -5
clock summer-time EDT recurring
clock set 09:00:00 15 August 2026
```
> If your NTP sources are also configured with a matching or UTC-based time, you'll be able to directly observe the zone offset being applied correctly once synchronization occurs.

### Task 3 — Configure Both NTP Sources
```
! R-NTP-A
ntp master 2
clock set 14:00:00 15 August 2026

! R-NTP-B
ntp master 2
clock set 14:00:05 15 August 2026
```
> Both are set to nearly the same time (5 seconds apart) to simulate two reasonably trustworthy, independent references — small natural drift between independent clocks is normal and part of why NTP exists.

### Task 4 — Configure R-CLIENT with Both Sources and a Preference
```
ntp server 10.5.1.1 prefer
ntp server 10.5.2.1
```
> The `prefer` keyword tells R-CLIENT to favor R-NTP-A when both sources are otherwise similarly valid — without it, IOS's selection algorithm picks based on stratum, measured accuracy, and other factors that aren't always obvious from a quick glance.

### Task 5 — Verify Initial Synchronization and Source Selection
Allow a few minutes for convergence, then:
```
show ntp status
show ntp associations
```
Confirm R-NTP-A shows the `*` (or `sys.peer`) selection marker, confirming the `prefer` setting is being honored, and that R-CLIENT's clock — once you account for the EST/EDT offset from Task 2 — correctly reflects the synchronized UTC-based time.

### Task 6 — Simulate the Preferred Source Failing
```
! On R-NTP-A
interface Gi0/0
 shutdown
```
### Task 7 — Verify Automatic Failover
```
show ntp associations
show ntp status
```
Confirm R-NTP-B now becomes the selected source (its association should now show the `*`/`sys.peer` marker), and R-CLIENT remains synchronized throughout — demonstrating the value of redundant time sources rather than a single point of failure.

### Task 8 — Restore the Preferred Source and Confirm Fallback-Back
```
! On R-NTP-A
interface Gi0/0
 no shutdown
```
Wait for reconvergence, then verify R-NTP-A resumes its role as the preferred, selected source:
```
show ntp associations
```

### Task 9 — Confirm Logging Timestamps Are Now Trustworthy
```
service timestamps log datetime msec
logging buffered 8192 informational
```
Trigger a test event (e.g., flap an unused interface) and check:
```
show logging
```
Confirm the log timestamp reflects the correct local time (zone-adjusted), not the incorrect manual clock value set in Task 2 — this ties the abstract "NTP is synchronized" status back to a concrete, practical benefit: logs are only useful for correlation and incident investigation if their timestamps are accurate.

### Task 10 — Confirm DHCP Lease Timestamps Depend on Accurate Time
Configure a basic DHCP pool on R-CLIENT for the LAN segment:
```
ip dhcp excluded-address 192.168.99.1 192.168.99.10
ip dhcp pool LAN-POOL
 network 192.168.99.0 255.255.255.0
 default-router 192.168.99.1
 lease 0 4 0
```
From PC1, obtain a DHCP lease, then check:
```
show ip dhcp binding
```
Confirm the lease expiration timestamp is consistent with R-CLIENT's current (now-accurate, NTP-synchronized) clock — if R-CLIENT's clock had still been wrong (as in Task 2, before synchronization), this lease expiration value would have been silently wrong too, potentially causing a client to hold or release an address at the wrong real-world time.

### Task 11 — Verify
```
show clock detail
show ntp status
show ntp associations detail
show run | section ntp
show run | section clock
show ip dhcp binding
show logging
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — initial sync | R-NTP-A selected (`*`/`sys.peer`), R-CLIENT's local time correctly zone-adjusted |
| Task 7 — R-NTP-A down | R-NTP-B automatically becomes the selected source; R-CLIENT stays synchronized throughout, no manual intervention needed |
| Task 8 — R-NTP-A restored | R-CLIENT falls back to R-NTP-A as the preferred source once it's available again |
| Task 9 — log timestamps | Reflect accurate, zone-adjusted local time, not the incorrect manually-set clock from Task 2 |
| Task 10 — DHCP lease timestamp | Consistent with R-CLIENT's accurate synchronized clock |
| `show clock detail` | Explicitly shows the time source as NTP-synchronized (not "not authoritative") once converged |

---

## Challenge (Optional)
- Add a **third** NTP source at a different, higher stratum (e.g., stratum 4) and confirm R-CLIENT's selection algorithm still correctly prefers the two lower-stratum, `prefer`-tagged sources over it, only falling back to the third if both A and B become unavailable.
- Combine this lab with the NTP Authentication lab: require authentication on both R-NTP-A and R-NTP-B, and confirm the failover behavior from Tasks 6–8 still works correctly when authentication is in play — failover shouldn't require re-establishing trust each time.
- Write a short justification (as if for a change-management request) explaining why an enterprise network should treat NTP as a **critical infrastructure service** deserving redundancy and monitoring, referencing at least two concrete downstream consequences observed in this lab (e.g., inaccurate log correlation, incorrect DHCP lease timing) as supporting evidence.