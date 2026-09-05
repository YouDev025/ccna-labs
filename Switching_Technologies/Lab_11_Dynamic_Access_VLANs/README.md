# Lab: Dynamic Access VLANs

## Objective
Extend the Identity Access lab's 802.1X configuration with **RADIUS-assigned (dynamic) VLANs**: instead of every authenticated user landing in the same fixed VLAN configured on the port, the **RADIUS server itself decides which VLAN a user belongs in**, based on their identity — meaning the exact same physical switch port can place two different users into two completely different VLANs, at different times, with **no configuration change on the switch port at all** between logins.

---

## Topology

```
              +-----------+
              | RADIUS1   |   Returns a VLAN assignment per authenticated user
              |192.168.90.10/24|
              +-----------+
                    |
                  SW1
                    |
                  Fa0/1
                    |
              +-----------+
              | (shared   |   Same physical port used by both test users,
              |  port)    |   one connection at a time
              +-----------+
```

- **Fa0/1** is a single physical port with **no static VLAN assignment of its own** — its VLAN membership is determined entirely by what RADIUS1 returns at authentication time.
- Two test identities (**alice** and **bob**) will each connect to this same port at different times, and should land in **different** VLANs based purely on who authenticated, not on any port-level configuration.

---

## VLAN Plan

| VLAN | Name  | Assigned To |
|------|-------|-------------|
| 10   | USERS | alice (via RADIUS) |
| 20   | SALES | bob (via RADIUS) |

---

## Tasks

### Task 1 — Build the Topology
1. Ensure VLANs 10 and 20 already exist on SW1.
2. Connect SW1 to RADIUS1, and confirm reachability.
3. Confirm 802.1X and AAA are already configured on SW1 as in the Identity Access lab (`aaa new-model`, RADIUS server/group definitions, `dot1x system-auth-control`).

### Task 2 — Configure Fa0/1 for 802.1X Without a Fixed VLAN
```
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 dot1x pae authenticator
```
> Note `switchport access vlan 10` is still present — this is the port's **fallback/default** VLAN, used only if no dynamic VLAN is returned by RADIUS at all (or before any authentication succeeds). Once RADIUS supplies a VLAN assignment for a specific authenticated session, that assignment **overrides** this static setting for the duration of that session.

### Task 3 — Configure RADIUS1 to Return VLAN Assignments Per User
On your RADIUS server (or conceptually, if a real server isn't available in your platform), configure each user's account to return the standard RADIUS attributes used for dynamic VLAN assignment:
- **Tunnel-Type** = VLAN (13)
- **Tunnel-Medium-Type** = 802 (6)
- **Tunnel-Private-Group-ID** = the VLAN number (e.g., `10` for alice, `20` for bob)

For **alice**: `Tunnel-Private-Group-ID = 10`
For **bob**: `Tunnel-Private-Group-ID = 20`

### Task 4 — Authenticate as Alice
Connect a device with a valid 802.1X supplicant configured with alice's credentials to Fa0/1.
```
show authentication sessions interface FastEthernet0/1
show dot1x interface FastEthernet0/1 details
```
Confirm the session shows **VLAN 10 assigned dynamically** (look for an "Assigned VLAN" or similar field distinguishing this from the port's static access VLAN) rather than simply inheriting the port's configured VLAN 10 by coincidence — note that in this specific case the numbers happen to match, so also complete Task 5 to unambiguously prove dynamic assignment is actually functioning, not just coincidentally correct.

### Task 5 — Disconnect Alice and Authenticate as Bob on the SAME Port
Disconnect alice's device, then connect a device with bob's 802.1X credentials to the **same physical port**, Fa0/1.
```
show authentication sessions interface FastEthernet0/1
```
Confirm this session shows **VLAN 20** — a completely different VLAN than the port's static fallback configuration (which is still set to VLAN 10). This is the definitive proof that VLAN assignment is happening dynamically, per-identity, from RADIUS — since the port's own static configuration never changed, yet bob correctly landed in a different VLAN than alice did moments earlier on the exact same physical connection point.

### Task 6 — Verify Traffic Actually Follows the Dynamic VLAN
While bob's session is active:
```
show vlan brief
```
Confirm Fa0/1 is currently listed under VLAN 20 in the operational VLAN table, despite the interface configuration still showing `switchport access vlan 10` in `show run` — this distinction (configured/static VLAN vs. operationally active VLAN) is the core concept of this lab.

### Task 7 — Test the Fallback Behavior
Disconnect bob and attempt to connect a device with **no** 802.1X supplicant and no MAB configured on this port at all (unlike the Identity Access lab's Fa0/3, this port has no MAB fallback configured).
```
show authentication sessions interface FastEthernet0/1
```
Confirm the port remains unauthorized, with **no** traffic passing at all — since there's no successful authentication (dynamic or otherwise) and no MAB fallback to grant even limited access here, distinct from a scenario where MAB or an auth-fail VLAN would provide some alternate outcome.

### Task 8 — Final Verification
```
show authentication sessions
show dot1x all
show vlan brief
show run interface FastEthernet0/1
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 4 — alice authenticates | Session shows VLAN 10 assigned (matches port default — verify dynamism further via Task 5) |
| Task 5 — bob authenticates on the same port | Session shows VLAN 20 — definitively different from the port's static configuration, proving dynamic assignment |
| Task 6 — operational VLAN table | Fa0/1 shows under VLAN 20 in `show vlan brief` while bob is connected, despite `show run` still showing VLAN 10 configured |
| Task 7 — no authentication, no MAB | Port remains unauthorized; no traffic passes |

---

## Challenge (Optional)
- Add a **third** user account on RADIUS1 with no VLAN attributes configured at all, and confirm that user's authenticated session falls back to the port's static default VLAN (10) rather than failing or being placed in an unexpected VLAN — clarifying exactly when the static `switchport access vlan` configuration still matters even with dynamic assignment enabled.
- Combine this lab with the Identity Access lab's auth-fail VLAN and MAB configuration on the same port, and verify all three outcomes (dynamic VLAN success, MAB fallback for a non-supplicant device, and auth-fail VLAN for a failed attempt) can coexist correctly on one port depending entirely on what happens during that specific connection attempt.
- Research the legacy **VMPS (VLAN Membership Policy Server)** feature — an older, MAC-address-based dynamic VLAN assignment mechanism that predates RADIUS-assigned VLANs — and explain in your lab notes why RADIUS-assigned VLAN (tied to genuine 802.1X identity, as in this lab) is considered a stronger and more maintainable design than VMPS's simple MAC-to-VLAN table approach.