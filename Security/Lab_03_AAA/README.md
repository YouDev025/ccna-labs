# Lab 03: AAA

## Objective
Configure and verify **AAA (Authentication, Authorization, and Accounting)** on a Cisco IOS router: start with local AAA (a structural stepping stone even without a real external server), then configure **server-based authentication** (TACACS+) with **automatic fallback to local** if the server is unreachable, and understand **method lists** — the core AAA concept that everything else builds on.

---

## Topology

```
              +-----------+
              | TACACS1   |   Simulated TACACS+ server
              |192.168.90.10/24|
              +-----------+
                    |
                  SW1
                    |
              Gi0/1  |  192.168.90.1/24
           +---------------+
           |      R1       |
           +---------------+
     Gi0/0  |
             |
      +---------+
      |  PC1    |   Used to test login via SSH/console
      |192.168.10.10/24|
      +---------+
```

- **R1** is the device being configured for AAA.
- **TACACS1** represents a centralized authentication server (simulated — if your lab platform doesn't support a real TACACS+ server, treat the server-based tasks conceptually and focus verification on the fallback behavior instead, which you can still test by pointing at an address that simply doesn't respond).
- **PC1** is used to test login behavior via SSH.

---

## IP Addressing Table

| Device   | Interface | IP Address       | Subnet Mask       |
|----------|-----------|-------------------|---------------------|
| R1       | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1       | Gi0/1     | 192.168.90.1       | 255.255.255.0        |
| TACACS1  | NIC       | 192.168.90.10       | 255.255.255.0        |
| PC1      | NIC       | 192.168.10.10       | 255.255.255.0        |

---

## Background: Why AAA Instead of Just `login local`?
A single local username/password (as in the SSH lab) works for one device, but doesn't scale: every router and switch would need its own separately-managed user database, with no centralized audit trail and no easy way to revoke one person's access everywhere at once when they leave. AAA solves this by centralizing **authentication** (who are you), **authorization** (what are you allowed to do), and **accounting** (what did you do) — typically backed by a TACACS+ or RADIUS server — while still supporting **local** credentials as a fallback method in case the central server is unreachable.

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, TACACS1, and PC1 as shown, and apply the addressing above.
2. Confirm R1 can ping TACACS1, and PC1 can ping R1.

### Task 2 — Ensure SSH and a Local Fallback Account Already Exist
This lab assumes the SSH lab's basic setup is already in place — if not, configure the prerequisites now:
```
hostname R1
ip domain-name lab.local
crypto key generate rsa
username localadmin privilege 15 secret LocalFallbackPass123
```
> `localadmin` is deliberately named differently from any server-based account you'll create later, so verification output makes it obvious which authentication source actually handled a given login.

### Task 3 — Enable the AAA Subsystem
```
aaa new-model
```
> This single command fundamentally changes how R1 handles authentication from this point forward — after this, **no login method works** until you explicitly define one with a method list, including console access. Some platforms/labs recommend keeping a console session open while making AAA changes specifically so you don't lock yourself out entirely if something is misconfigured.

### Task 4 — Define a TACACS+ Server
```
tacacs server TACACS1
 address ipv4 192.168.90.10
 key TacacsSharedSecret456
```
> The `key` here is a shared secret between R1 and the TACACS+ server — it must match exactly what's configured on the server side, similar in concept to the pre-shared keys used in the VPN and NTP/EIGRP/OSPF authentication labs, though serving a different purpose here.

### Task 5 — Create a Method List with Server-First, Local-Fallback Order
```
aaa authentication login SSH-LOGIN group tacacs+ local
```
> This defines a **named method list** called `SSH-LOGIN`: try the TACACS+ server group first; if it's unreachable (not simply "wrong password" — that's a definitive answer from a reachable server, not a fallback trigger), fall back to the local username database from Task 2.

### Task 6 — Apply the Method List to the VTY Lines
```
line vty 0 4
 login authentication SSH-LOGIN
 transport input ssh
```
> Note this uses `login authentication SSH-LOGIN` (a named list) rather than `login local` from the SSH lab — `login local` was really just a simplified way of using AAA's built-in default local-only behavior all along; now you're using an explicit, named list with a defined fallback order instead.

### Task 7 — Test Login When the TACACS+ Server Is Reachable but Doesn't Recognize the User
If your lab environment has a working TACACS+ server, configure a test account there and confirm SSH login succeeds using the server-side credentials, authenticated entirely by TACACS1, not R1's local database:
```
! From PC1
ssh -l <server-configured-username> 192.168.10.1
```
Confirm this succeeds using only server-side credentials. If your platform doesn't support a real TACACS+ server, skip to Task 8, treating this step as conceptual (the mechanism is the same either way — the difference is just whether a real server processes the request).

### Task 8 — Test Fallback When the Server Is Unreachable
Simulate the TACACS+ server being down or unreachable by pointing R1 at an address that doesn't respond, or by shutting down R1's Gi0/1 interface temporarily:
```
! Option A: temporarily break reachability to the real server
interface GigabitEthernet0/1
 shutdown
```
```
! From PC1
ssh -l localadmin 192.168.10.1
```
Enter `localadmin`'s password from Task 2. Confirm this succeeds — R1 attempted the TACACS+ server first, found it unreachable, and correctly fell back to the local database. Restore the interface afterward:
```
interface GigabitEthernet0/1
 no shutdown
```

### Task 9 — Test That a Wrong Password Does NOT Trigger Fallback (Important Distinction)
With the TACACS+ server reachable again, attempt to log in with a valid server-recognized username but a **deliberately wrong password**:
```
! From PC1
ssh -l <server-configured-username> 192.168.10.1
```
Enter an incorrect password. Confirm this is **rejected outright** — it does **not** fall back to trying the local database, since the server was reachable and gave a definitive "authentication failed" answer, which is a completely different situation from the server being unreachable at all. This distinction (unreachable vs. reachable-but-wrong) is one of the most commonly misunderstood parts of AAA fallback behavior.

### Task 10 — Apply AAA to the Console Line Too (Carefully)
```
line console 0
 login authentication SSH-LOGIN
```
> Modifying console authentication carries real risk of lockout if misconfigured — always keep an active session open (as in Task 3's note) before testing a change like this, and confirm you can still log in via console using the local fallback account before ending your session.

### Task 11 — Enable Basic AAA Accounting (Conceptual/Verification-Focused)
```
aaa accounting exec default start-stop group tacacs+
```
> This tells R1 to send accounting records (session start/stop) to the TACACS+ server for every EXEC session — if your server platform can display these, confirm a record appears for your Task 7/8 test logins; if not, treat this as a conceptual configuration step and verify only that the command is accepted and appears in the running config.

### Task 12 — Verify
```
show run | section aaa
show aaa method-lists authentication
show tacacs
debug aaa authentication   (optional, generate a login attempt then 'undebug all')
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — `aaa new-model` | Command accepted; console access still available via an existing open session |
| Task 5/6 — method list applied | `show run | section aaa` shows `SSH-LOGIN` defined and applied to VTY lines |
| Task 7 — server-side login (if available) | Succeeds using only TACACS1-side credentials |
| Task 8 — server unreachable | Falls back to `localadmin`; login succeeds using local credentials |
| Task 9 — wrong password, server reachable | Login rejected outright; does **not** fall back to local — confirms fallback is for unreachability, not failed credentials |
| Task 10 — console authentication | Console access still works via the local fallback account after applying the method list |
| `show tacacs` | Shows the configured server and connection statistics |

---

## Challenge (Optional)
- Configure a **second** named method list (e.g., `CONSOLE-LOGIN`) that's local-only (no TACACS+ at all), and apply it specifically to the console line while keeping `SSH-LOGIN` (server-first) on the VTY lines — discuss in your lab notes why a network team might deliberately want different authentication behavior for console vs. remote access.
- Research and configure (if your platform supports it) **AAA authorization** (`aaa authorization exec default group tacacs+ local`) in addition to authentication, and explain the practical difference between a user being *authenticated* (proven who they are) versus *authorized* (allowed to do a specific thing, like reach privileged EXEC mode) — these are frequently conflated but are genuinely separate AAA functions.
- Write a short incident-response note: if a network engineer reports they "can't log into any router anymore" right after an AAA change was deployed, list the specific things you'd check first, in order, referencing this lab's own Task 3 note about console session risk as your starting point.