# Lab 03: NTP

## Objective
Configure and verify **Network Time Protocol (NTP)** synchronization across a small router topology: set up an authoritative NTP server, configure clients to synchronize to it, secure the synchronization with authentication, and confirm accurate, consistent time using `show ntp associations`, `show ntp status`, and `show clock`.

---

## Topology

```
              +-----------+
              |   R-NTP   |   NTP server (stratum source)
              |10.0.0.1/30 (to R2)|
              +-----------+
                    |
              10.0.0.0/30
                    |
              +-----------+
              |    R2     |   NTP client of R-NTP, and NTP server for R3
              +-----------+
              10.0.0.2/30 (to R-NTP)
              10.0.1.1/30 (to R3)
                    |
              10.0.1.0/30
                    |
              +-----------+
              |    R3     |   NTP client of R2
              +-----------+
              10.0.1.2/30
```

- **R-NTP** is configured as the authoritative time source for this lab (simulating a stratum 1 appliance or an upstream NTP server such as `pool.ntp.org` — in a real deployment R-NTP itself would sync to an external source, but for this lab it will act as its own reference).
- **R2** synchronizes to R-NTP, and in turn **also acts as an NTP server** for R3 — demonstrating a common hierarchical (stratum-layered) NTP design.
- **R3** synchronizes only to R2, never directly to R-NTP.

---

## IP Addressing Table

| Device | Interface | IP Address | Subnet Mask       |
|--------|-----------|-------------|---------------------|
| R-NTP  | Gi0/0     | 10.0.0.1     | 255.255.255.252      |
| R2     | Gi0/0     | 10.0.0.2     | 255.255.255.252      |
| R2     | Gi0/1     | 10.0.1.1     | 255.255.255.252      |
| R3     | Gi0/0     | 10.0.1.2     | 255.255.255.252      |

> Static routes or a routing protocol should already connect these three links so each router can reach the others' addresses — this lab assumes basic IP reachability is already working.

---

## Tasks

### Task 1 — Build the Topology
1. Place R-NTP, R2, and R3 as shown, and apply the addressing above.
2. Confirm each router can ping the adjacent router's address (R-NTP↔R2, R2↔R3) before configuring NTP.

### Task 2 — Deliberately Desynchronize the Clocks
So the lab clearly demonstrates synchronization working, manually set each router's clock to a different, incorrect time before configuring NTP:
```
! On R-NTP — set to the "correct" reference time
clock set 14:00:00 15 August 2026

! On R2 — set several minutes off
clock set 14:07:32 15 August 2026

! On R3 — set even further off
clock set 13:52:10 15 August 2026
```
Confirm with `show clock` on each device that all three currently disagree.

### Task 3 — Configure R-NTP as the Master Time Source
```
ntp master 3
```
> The number (3 here) sets the stratum level R-NTP will advertise itself as. Stratum 1 is reserved for devices directly attached to a precision reference clock (e.g., GPS/atomic); using a higher stratum number like 3 here is more realistic for a router acting as an internal reference without dedicated hardware.

### Task 4 — Configure R2 as an NTP Client of R-NTP
```
ntp server 10.0.0.1
```

### Task 5 — Configure R2 to Also Serve NTP to R3
A router can be an NTP client and an NTP server for a different link at the same time — this is exactly what makes hierarchical NTP designs possible:
```
ntp master 4
```
> Alternatively, in more advanced configurations, R2 can be configured to relay/pass through the time it learns from R-NTP rather than declaring itself a new master — for this lab, using `ntp master` on R2 at a higher (less authoritative) stratum number is sufficient to demonstrate the hierarchy concept simply.

### Task 6 — Configure R3 as an NTP Client of R2
```
ntp server 10.0.1.1
```

### Task 7 — Wait and Observe Convergence
NTP synchronization is not instantaneous — allow a few minutes for the association to form and the clock to step/slew into alignment. Periodically check:
```
show ntp status
show ntp associations
show clock
```
on R2 and R3 until each shows a synchronized state.

### Task 8 — Secure NTP with Authentication (Recommended Practice)
Unauthenticated NTP allows any device that can reach the server to (at least attempt to) influence a client's clock, which has real security implications (certificate validity windows, logging timestamps, etc. all depend on accurate time). Add authentication between R-NTP and R2:
```
! On R-NTP
ntp authenticate
ntp authentication-key 1 md5 NtpLabKey123
ntp trusted-key 1

! On R2 (client side)
ntp authenticate
ntp authentication-key 1 md5 NtpLabKey123
ntp trusted-key 1
no ntp server 10.0.0.1
ntp server 10.0.0.1 key 1
```
Repeat the same pattern between R2 and R3 with a (optionally different) key if you want the full chain authenticated.

### Task 9 — Verify
On each router:
```
show clock
show clock detail
show ntp status
show ntp associations
show ntp associations detail
show run | section ntp
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show clock` on all three routers, before NTP (Task 2) | All three show different, deliberately incorrect times |
| `show ntp status` on R2 | Eventually shows `Clock is synchronized`, reference clock = R-NTP's IP, correct stratum |
| `show ntp associations` on R2 | Shows R-NTP as a peer with a `*` (or `sys.peer`) marker indicating it's the selected synchronization source |
| `show clock` on R2 (after convergence) | Matches R-NTP's time (accounting for the small propagation/processing delay) |
| `show ntp status` on R3 | Eventually shows synchronized, with R2 (not R-NTP directly) as its reference |
| `show clock` on R3 (after convergence) | Matches R2's (and therefore R-NTP's) time |
| After Task 8 (authentication) | `show ntp associations detail` shows the association as authenticated; removing/mismatching the key on one side should cause the association to fail to form or become unsynchronized — test this by temporarily changing the key on only one side |
| `show run \| section ntp` | Confirms `ntp authenticate`, `ntp authentication-key`, and `ntp trusted-key` are present wherever authentication was configured |

---

## Challenge (Optional)
- Break authentication deliberately (mismatch the key on R2 vs. R-NTP) and use `show ntp associations detail` to identify exactly what output indicates an authentication failure versus a normal unsynchronized/reachability problem — document the difference in your lab notes.
- Add a fourth router as a second, independent NTP client of R-NTP, and use `show ntp associations` on R-NTP itself to confirm it can serve multiple clients simultaneously.
- Research (and note without necessarily configuring, if your lab platform doesn't support it) how `ntp source <interface>` and `ip access-list` based NTP filtering (`ntp access-group`) can be used to control which interface NTP packets originate from and which peers are permitted to synchronize against a server — both common hardening steps in production NTP deployments.