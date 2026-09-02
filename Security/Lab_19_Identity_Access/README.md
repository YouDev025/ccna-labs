# Lab: Identity Access

## Objective
Configure and verify **802.1X port-based network access control** — identity-based access, distinct from the MAC-address-based approach used in the Port Security labs. A device (or its user) must **authenticate** before the switch port grants real network access at all, rather than the switch simply learning and trusting whichever MAC address shows up first. This lab also covers **MAC Authentication Bypass (MAB)** for legitimate devices that can't run an 802.1X supplicant (e.g., a network printer), and an **auth-fail VLAN** for devices that fail authentication entirely.

---

## Topology

```
              +-----------+
              | RADIUS1   |   AAA/RADIUS server providing 802.1X authentication
              |192.168.90.10/24|
              +-----------+
                    |
                  SW1
          +---------+---------+---------+
          |         |         |         |
     Fa0/1     Fa0/2     Fa0/3     Fa0/4
   (802.1X   (802.1X   (MAB —    (used to
   supplicant,fail case,printer, simulate
   success)   wrong    no        an auth
              creds)   supplicant)failure)
```

- **Fa0/1**: a PC with a properly configured 802.1X supplicant and valid credentials — should authenticate successfully.
- **Fa0/2**: used to test a **failed** authentication attempt (wrong credentials) and observe the resulting restricted access.
- **Fa0/3**: a printer with no 802.1X supplicant capability at all — relies on **MAB** instead.
- **Fa0/4**: used to demonstrate the **auth-fail VLAN** behavior for a device that never successfully authenticates.

---

## VLAN Plan

| VLAN | Purpose |
|------|---------|
| 10   | Production (authenticated devices land here) |
| 999  | Auth-fail / restricted (devices that fail 802.1X land here instead of being fully blocked) |

---

## Tasks

### Task 1 — Build the Topology
1. Place SW1, RADIUS1, and the four test endpoints on Fa0/1–Fa0/4.
2. Create VLAN 10 (production) and VLAN 999 (auth-fail/restricted) on SW1.
3. Confirm SW1 can reach RADIUS1.

### Task 2 — Enable AAA and Define the RADIUS Server
```
aaa new-model
radius server RADIUS1
 address ipv4 192.168.90.10 auth-port 1812 acct-port 1813
 key Dot1xSharedSecret789

aaa group server radius DOT1X-RADIUS
 server name RADIUS1

aaa authentication dot1x default group DOT1X-RADIUS
aaa authorization network default group DOT1X-RADIUS
```

### Task 3 — Enable 802.1X Globally
```
dot1x system-auth-control
```
> This is the master switch for 802.1X on the device — without it, none of the per-port configuration in the following tasks has any effect at all.

### Task 4 — Configure Fa0/1 for Basic 802.1X
```
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 dot1x pae authenticator
```
> `port-control auto` is what actually enforces authentication — the port starts in an unauthorized state and only moves to authorized once 802.1X (or MAB, configured later) succeeds. Compare this against `port-control force-authorized` (effectively disables 802.1X, always granting access) and `force-unauthorized` (always blocks, regardless of credentials) — `auto` is the only setting that actually performs real authentication.

### Task 5 — Verify Fa0/1's Pre-Authentication State
```
show authentication sessions interface FastEthernet0/1
```
Before the connected PC's supplicant has authenticated, confirm the port shows as **unauthorized** — no meaningful network access should be possible yet, even though the physical link is up.

### Task 6 — Authenticate the Supplicant on Fa0/1
Using a properly configured 802.1X supplicant on the connected PC (built into most modern OSes, or a dedicated client), supply valid credentials that RADIUS1 is configured to accept. Confirm:
```
show authentication sessions interface FastEthernet0/1
```
The session should now show as **Authz Success**, with the port authorized and passing traffic on VLAN 10.

### Task 7 — Test a Failed Authentication on Fa0/2
```
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 dot1x pae authenticator
```
Attempt to authenticate using **deliberately wrong** credentials from the device connected here.
```
show authentication sessions interface FastEthernet0/2
```
Confirm the session shows a failed/unauthorized state, and — with no further configuration yet — the port has **no** production network access, consistent with a straightforward failed authentication.

---

## Part 2 — MAC Authentication Bypass (MAB) for Non-802.1X Devices

### Task 8 — Understand the MAB Use Case
Not every legitimate device can run an 802.1X supplicant — printers, some IoT devices, and older equipment often have no such capability at all. Without some fallback, these devices would be permanently unable to gain network access on an 802.1X-enforced port. MAB allows the switch to instead authenticate using the device's **MAC address** as a (weaker, but pragmatic) credential, checked against the RADIUS server.

### Task 9 — Configure Fa0/3 for 802.1X-with-MAB-Fallback
```
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 authentication order dot1x mab
 authentication priority dot1x mab
 dot1x pae authenticator
 mab
```
> `authentication order dot1x mab` tells the switch to attempt 802.1X first, and only fall back to MAB if no 802.1X supplicant responds at all within the timeout — this ordering is important: MAB should be the fallback, not the primary method, since a MAC address is a much weaker credential than genuine 802.1X authentication.

### Task 10 — Pre-Register the Printer's MAC Address on RADIUS1
On your RADIUS server (or conceptually, if a real server isn't available in your platform), configure the printer's MAC address as a recognized/authorized identity — MAB typically checks the MAC address as if it were a username with no meaningful password.

### Task 11 — Verify MAB Success
```
show authentication sessions interface FastEthernet0/3
```
Confirm the printer's session shows as authorized, with the authentication **method** specifically shown as MAB (not dot1x) — this distinction matters for later auditing, since MAB-authenticated sessions represent a weaker assurance level than genuine 802.1X ones.

---

## Part 3 — Auth-Fail VLAN for Non-Compliant Devices

### Task 12 — Configure Fa0/4 with an Auth-Fail VLAN
Rather than leaving a failed device with zero access at all, some designs place it into a restricted VLAN instead — enough for remediation (e.g., reaching a captive portal or a limited set of update servers) without granting real production access:
```
interface FastEthernet0/4
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 authentication event fail action authorize vlan 999
 dot1x pae authenticator
```

### Task 13 — Trigger and Verify the Auth-Fail VLAN
Attempt authentication from a device connected to Fa0/4 using invalid credentials.
```
show authentication sessions interface FastEthernet0/4
```
Confirm the session is placed into VLAN 999 rather than being fully blocked, and that the device can reach whatever limited resources exist on that restricted VLAN (if configured), but nothing on the production VLAN 10 network.

### Task 14 — Final Verification
```
show dot1x all
show authentication sessions
show mab all   (or platform equivalent)
show run | section dot1x
show run | include authentication
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — pre-authentication | Fa0/1 shows unauthorized; no meaningful access before login |
| Task 6 — successful 802.1X | Fa0/1 shows Authz Success; VLAN 10 access granted |
| Task 7 — failed 802.1X, no fallback configured | Fa0/2 remains unauthorized; no production access |
| Task 11 — MAB success | Fa0/3's printer authorized via MAB (not dot1x), confirming the fallback correctly activated for a non-supplicant device |
| Task 13 — auth-fail VLAN | Fa0/4's failed device lands in VLAN 999, not fully blocked and not granted production access |
| `show authentication sessions` (switch-wide) | Shows all four ports with their correct method (dot1x/mab), VLAN, and authorization status |

---

## Challenge (Optional)
- Configure a **timeout-based** fallback (`authentication order dot1x mab` combined with an appropriate `dot1x timeout` value) and measure how long a non-802.1X device actually waits before MAB kicks in — discuss the tradeoff between a short timeout (faster fallback, but less time for a slow supplicant to respond) and a long one.
- Compare 802.1X+MAB against the earlier Port Security labs' MAC-based approach directly: write a short comparison covering what each control actually verifies (a MAC address existing on the wire, versus a real authenticated identity or, at minimum, a pre-registered MAC checked centrally), and explain why 802.1X is generally considered the stronger control despite MAB narrowing that gap only partially.
- Research **multi-auth** and **multi-domain authentication (MDA)** host modes (used for ports with both a phone and a PC, similar to the Advanced Port Security lab's voice+data scenario, but authenticated via 802.1X rather than simple MAC limits) and describe how they differ from the single-host mode used throughout this lab.