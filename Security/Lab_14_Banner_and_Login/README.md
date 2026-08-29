# Lab: Banner and Login

## Objective
Configure and verify **legal banners** (a real compliance/legal requirement in most organizations, not just cosmetic text) and **login security hardening**: automatic session timeouts, and **`login block-for`** brute-force protection, which detects repeated failed login attempts and temporarily blocks further attempts entirely — including from legitimate users, which is an important and often-surprising side effect you'll observe directly.

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
              |(legit.    |         |(simulates |
              | user)     |         | a brute-  |
              +-----------+         | force     |
                                     | attacker) |
                                     +-----------+
```

- **R1** is the device being hardened.
- **PC1** represents a legitimate user, used to confirm normal login still works and later to observe the (surprising) side effect of being blocked alongside an attacker.
- **PC2** is used to simulate repeated failed login attempts.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |
| PC2    | NIC       | 192.168.10.11       | 255.255.255.0        |

---

## Part 1 — Legal and Informational Banners

### Task 1 — Build the Topology and Confirm SSH Prerequisites
```
hostname R1
ip domain-name lab.local
crypto key generate rsa modulus 2048
username netadmin privilege 15 secret StrongAdminPass123
line vty 0 4
 transport input ssh
 login local
```

### Task 2 — Configure a Message-of-the-Day (MOTD) Banner
```
banner motd #
***************************************************************
  WARNING: This system is for authorized use only. All activity
  is logged and monitored. Unauthorized access is prohibited and
  may be subject to legal action. Disconnect immediately if you
  are not an authorized user.
***************************************************************
#
```
> The `#` characters are **delimiters**, not part of the banner text — any character not appearing anywhere in the banner text itself can be used as the delimiter; `#` is a common, safe choice. The MOTD banner displays **before** login, to anyone who connects at all, whether or not they successfully authenticate.

### Task 3 — Configure a Login Banner
```
banner login #
Enter your credentials below.
Access is restricted to authorized personnel only.
#
```
> The login banner displays immediately before the username/password prompt — functionally similar to MOTD in many setups, but some designs use both to separate a general legal notice (MOTD) from a more specific access-instruction message (login).

### Task 4 — Configure an EXEC Banner
```
banner exec #
You have successfully authenticated to R1.
Remember: all configuration changes must follow the change
management process.
#
```
> The EXEC banner displays **only after** successful authentication, making it useful for post-login reminders that shouldn't be shown to unauthenticated or failed connection attempts.

### Task 5 — Verify Banner Behavior
```
! From PC1
ssh -l netadmin 192.168.10.1
```
Confirm you see the banners in the correct order: MOTD first, then login, then (after successfully entering credentials) the EXEC banner. Attempt a connection with a **deliberately wrong password** and confirm you still see the MOTD and login banners (since those display before/during authentication, regardless of outcome), but **not** the EXEC banner (since authentication didn't succeed).

### Task 6 — Understand Why the Wording Matters (Legal Context)
In your lab notes, briefly explain why a banner stating something like "Welcome" or providing no warning at all is generally considered **legally weaker** in the context of prosecuting unauthorized access than a banner that clearly states the system is private, monitored, and that unauthorized access is prohibited — this isn't just security theater; banner wording has genuinely been referenced in real legal proceedings involving unauthorized computer access.

---

## Part 2 — Session Timeout Hardening

### Task 7 — Configure EXEC Timeout
```
line vty 0 4
 exec-timeout 10 0
line console 0
 exec-timeout 10 0
```
> `10 0` means 10 minutes, 0 seconds — an idle session will be automatically disconnected after this period. Leaving this at its default (often much longer, or in some configurations effectively unlimited) leaves an authenticated session exposed if someone walks away from an open terminal.

### Task 8 — Verify Timeout Configuration
```
show run | section line vty
show run | section line con
```
Confirm the configured `exec-timeout` values appear on both line types.

---

## Part 3 — Brute-Force Login Protection

### Task 9 — Understand the Threat Being Addressed
Without any protection, an attacker (or automated tool) can attempt unlimited password guesses against SSH with no consequence beyond the time it takes. `login block-for` addresses this by watching for a threshold of failures within a time window and then **blocking all new login attempts** (from anyone, not just the offending source, unless combined with an exception list) for a defined penalty period.

### Task 10 — Configure Login Block-For
```
login block-for 120 attempts 3 within 60
```
> This means: if **3** failed login attempts occur within any **60**-second window, block all further login attempts for **120** seconds. Tune these values based on realistic legitimate user behavior (a typo or two shouldn't trigger a lockout) versus how quickly you want to respond to an actual attack.

### Task 11 — Configure an Exception List (Trusted Management Access)
Without an exception, `login block-for` blocks **everyone**, including legitimate administrators, once triggered — which is a real and important operational risk to understand, not just a footnote. Exempt a trusted management subnet from the block:
```
ip access-list standard MGMT-EXCEPTION
 permit 192.168.10.10
login quiet-mode access-class MGMT-EXCEPTION
```
> In this lab, this exempts only PC1's specific address — in a real deployment, this would typically be a dedicated out-of-band management network or jump host address, not just "one admin's regular workstation," to keep the exception itself both meaningful and auditable.

### Task 12 — Trigger the Brute-Force Protection from PC2
```
! From PC2, attempt SSH with a wrong password 3 times within 60 seconds
ssh -l netadmin 192.168.10.1   (enter wrong password)
ssh -l netadmin 192.168.10.1   (enter wrong password)
ssh -l netadmin 192.168.10.1   (enter wrong password)
```

### Task 13 — Verify PC2 Is Now Blocked
```
! From PC2, immediately attempt again, even with the CORRECT password this time
ssh -l netadmin 192.168.10.1
```
Confirm this is refused/times out — once `login block-for`'s threshold is triggered, even correct credentials from a blocked (non-exempted) source are rejected during the penalty period.

### Task 14 — Verify PC1 (Exempted) Is NOT Blocked
```
! From PC1, during the same penalty period
ssh -l netadmin 192.168.10.1
```
Confirm this **succeeds** with the correct password, since PC1's address was explicitly exempted in Task 11 — demonstrating the exception list is working correctly and that trusted management access survives a brute-force event elsewhere on the network.

### Task 15 — Verify on R1
```
show login
show login failures
```
Confirm `show login` reports the system is currently in **quiet mode** (or shows the active block, wording varies by IOS version), with a countdown/remaining time, and `show login failures` lists the failed attempts from PC2 including source address and timestamp.

### Task 16 — Wait for the Penalty Period to Expire and Confirm Recovery
After the configured 120-second window elapses:
```
! From PC2
ssh -l netadmin 192.168.10.1
```
This should now succeed with the correct password, confirming the block was temporary and self-clearing, not a permanent lockout requiring manual intervention.

### Task 17 — Final Verification
```
show login
show login failures
show run | include login block-for
show run | include banner
show run | section line vty
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — banner order | MOTD → login banner → (only on success) EXEC banner; failed login shows MOTD/login but not EXEC |
| Task 8 — timeout | `exec-timeout 10 0` present on both VTY and console lines |
| Task 12–13 — PC2 brute-force | After 3 failed attempts within 60 seconds, even a subsequent **correct** password attempt from PC2 is refused during the block |
| Task 14 — PC1 exemption | PC1 can still log in successfully during the same block period |
| Task 15 — `show login` / `show login failures` | Shows active quiet-mode/block status and a record of PC2's failed attempts |
| Task 16 — recovery | PC2 can log in again automatically once the 120-second penalty period expires |

---

## Challenge (Optional)
- Adjust the `login block-for` thresholds to be deliberately too aggressive (e.g., `attempts 1 within 60`) and discuss in your lab notes the operational risk of a threshold this strict — specifically, how a single legitimate typo could trigger a full lockout for everyone not on the exception list.
- Combine this lab with the AAA labs: configure `login block-for` alongside AAA-based authentication rather than local-only, and confirm the brute-force protection still functions correctly regardless of which authentication backend ultimately processes the credentials.
- Research and document your organization's (or a hypothetical organization's) actual legal banner requirements — many real environments have a legally-reviewed standard banner text that should be used verbatim rather than improvised per-device, and explain why consistency across all devices matters for this specific control.