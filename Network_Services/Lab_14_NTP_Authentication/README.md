# Lab: NTP Authentication

## Objective
This lab isolates **one specific skill**: securing NTP with **MD5 authentication** so a client only accepts time from a trusted, verified source — and, just as importantly, proving that an **unauthenticated or improperly-keyed source is correctly rejected**. If you've done the general NTP lab, this one goes deeper: it includes a simulated **rogue NTP server** so you can directly observe the difference between a client that blindly trusts any NTP source and one that only trusts authenticated peers.

---

## Topology

```
              +-----------+
              |  R-TRUE   |   Legitimate, authenticated NTP server
              |10.1.1.1/30 (to R-CLIENT)|
              +-----------+
                    |
              10.1.1.0/30
                    |
              +-----------+          +-----------+
              | R-CLIENT  |----------| R-ROGUE   |   Simulated rogue/untrusted NTP server
              +-----------+ 10.1.2.0/30 +-----------+
              10.1.1.2/30              10.1.2.2/30
              10.1.2.1/30
```

- **R-TRUE** is the legitimate, authenticated time source R-CLIENT should sync to.
- **R-ROGUE** simulates an attacker (or simply a rogue/misconfigured device) advertising itself as an NTP server on the same network, without valid authentication credentials.
- **R-CLIENT** is configured to require authentication — it should sync only with R-TRUE and correctly reject R-ROGUE.

---

## IP Addressing Table

| Device    | Interface | IP Address | Subnet Mask       |
|-----------|-----------|-------------|---------------------|
| R-TRUE    | Gi0/0     | 10.1.1.1     | 255.255.255.252      |
| R-CLIENT  | Gi0/0     | 10.1.1.2     | 255.255.255.252      |
| R-CLIENT  | Gi0/1     | 10.1.2.1     | 255.255.255.252      |
| R-ROGUE   | Gi0/0     | 10.1.2.2     | 255.255.255.252      |

---

## Tasks

### Task 1 — Build the Topology
1. Place R-TRUE, R-CLIENT, and R-ROGUE as shown, and apply the addressing above.
2. Confirm R-CLIENT can ping both R-TRUE and R-ROGUE before configuring NTP.

### Task 2 — Desynchronize the Clocks
```
! R-TRUE
clock set 10:00:00 15 August 2026

! R-CLIENT
clock set 10:20:00 15 August 2026

! R-ROGUE
clock set 08:45:00 15 August 2026
```

### Task 3 — Configure R-TRUE as an Authoritative, Authenticated Source
```
ntp master 3
ntp authenticate
ntp authentication-key 5 md5 TrueKeyABC
ntp trusted-key 5
```

### Task 4 — Configure R-ROGUE as an Unauthenticated Source
Deliberately do **not** configure authentication on R-ROGUE — this represents either a genuine attacker (who wouldn't have your key material) or an unmanaged/rogue device on the network:
```
ntp master 8
```
> Using a higher (less authoritative) stratum number here is incidental — the point of this lab isn't stratum preference, it's that R-ROGUE has no valid authentication key at all.

### Task 5 — Configure R-CLIENT to Require Authentication
```
ntp authenticate
ntp authentication-key 5 md5 TrueKeyABC
ntp trusted-key 5
ntp server 10.1.1.1 key 5
```
> Notice R-CLIENT is **not** configured with an `ntp server` statement pointing at R-ROGUE at all in this first pass — Task 7 will change that to test a more realistic and more dangerous scenario.

### Task 6 — Verify R-CLIENT Syncs Successfully with R-TRUE
Allow a few minutes for convergence, then:
```
show ntp status
show ntp associations
show ntp associations detail
```
Confirm `show ntp status` reports `Clock is synchronized`, and `show clock` on R-CLIENT now matches R-TRUE's time.

### Task 7 — Add R-ROGUE as a Second, Unauthenticated Server Reference
This is the core test of the lab: even if R-CLIENT is *told* to consider R-ROGUE as a time source, authentication should still prevent it from being trusted:
```
ntp server 10.1.2.2
```
> Note this line deliberately omits `key 5` — simulating an admin (or attacker-controlled device) that either doesn't have the key or is attempting to be accepted without one.

### Task 8 — Verify R-ROGUE Is Rejected
```
show ntp associations
show ntp associations detail
```
R-ROGUE should appear in the association list but **without** a `*`/`sys.peer` selection marker, and `show ntp associations detail` should indicate it is **not** authenticated (look for an `unsynced` or authentication-related status flag, exact wording varies by IOS version — record what you observe). Confirm R-CLIENT's actual synchronized time (`show clock`) still matches R-TRUE, **not** R-ROGUE's clock value from Task 2.

### Task 9 — Simulate a Key Mismatch (Not Just a Missing Key)
A more subtle and realistic failure: R-ROGUE claims to support authentication, but with the **wrong** key value — simulating a misconfigured legitimate device rather than an obvious attacker with no key at all:
```
! On R-ROGUE
ntp authenticate
ntp authentication-key 5 md5 WrongKeyXYZ
ntp trusted-key 5
```
```
! On R-CLIENT — update its reference to R-ROGUE to now claim key 5 too
no ntp server 10.1.2.2
ntp server 10.1.2.2 key 5
```
Since R-CLIENT's actual `ntp authentication-key 5` value is still `TrueKeyABC` (from Task 5) and R-ROGUE is now using `WrongKeyXYZ` for the same key number, the MD5 hashes will not match.

### Task 10 — Verify the Key Mismatch Is Detected
```
show ntp associations detail
```
Confirm R-ROGUE is still rejected — this time due to an authentication **failure** (hash mismatch) rather than simply lacking a key at all. Compare the detail output between Task 8 (no key offered) and this task (wrong key offered) and note any difference in how IOS reports each condition.

### Task 11 — Restore a Correct Configuration and Confirm Recovery
Remove the rogue association entirely to leave R-CLIENT in a clean, fully-trusted state:
```
no ntp server 10.1.2.2
```
```
show ntp associations
show clock
```
Confirm R-CLIENT is synchronized solely with R-TRUE.

### Task 12 — Verify
```
show ntp status
show ntp associations detail
show run | section ntp
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 6 — R-CLIENT ↔ R-TRUE | Synchronizes successfully; `show ntp status` shows synchronized |
| Task 8 — R-ROGUE with no key offered | Appears in associations but never becomes the selected sync source; R-CLIENT's clock still matches R-TRUE |
| Task 10 — R-ROGUE with wrong key | Also rejected, but the detail output should indicate an authentication failure rather than "no authentication attempted" |
| R-CLIENT's clock throughout Tasks 7–11 | Never adopts R-ROGUE's (incorrect) time at any point |
| `show run \| section ntp` on R-CLIENT | Shows `ntp authenticate`, the correct key/value, `ntp trusted-key 5`, and only the R-TRUE server statement remaining after Task 11 |

---

## Challenge (Optional)
- Configure a **second, different** authentication key (e.g., key 6) on R-TRUE and R-CLIENT, and confirm R-CLIENT can be configured to trust either key independently — useful for supporting multiple legitimate NTP sources with different credentials, such as during a key-rotation window.
- Remove `ntp trusted-key 5` from R-CLIENT (while leaving `ntp authentication-key 5` configured) and observe/document the resulting behavior — this demonstrates that simply having the correct key value configured is not sufficient; it must also be explicitly marked as trusted.
- Write a short incident-response note (as if this were a real network) describing what evidence from `show ntp associations detail` you would present to justify that a suspected rogue NTP source was successfully blocked, and what you would still recommend investigating further (e.g., why an unauthorized device was on the network sending NTP traffic at all).