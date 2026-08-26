# Lab 02: Port Security

## Objective
Configure and verify **port security** on a Cisco switch: limit the number of MAC addresses allowed on an access port, learn addresses dynamically via **sticky** MAC, and observe all three **violation modes** (protect, restrict, shutdown) by deliberately triggering a violation and recovering from it.

---

## Topology

```
              +-----------+
              |    SW1    |
              +-----------+
               Fa0/1  |  Fa0/2
                |            |
          +---------+   +---------+
          |  PC1    |   |  PC2    |
          |(legit.  |   | (used to|
          | host)   |   | simulate|
          +---------+   | a second|
                          | device  |
                          | on the  |
                          | same    |
                          | port)   |
                          +---------+
```

- **Fa0/1** on SW1 is the port being secured — PC1 is its intended, single legitimate device.
- **PC2** is used at different points in the lab to simulate an unauthorized second device appearing on the secured port (by physically/logically swapping it in place of PC1, or by connecting it to the same port via a hub/unmanaged switch if your platform supports simulating that).

---

## Addressing Table

| Device | Port  | IP Address       | Subnet Mask       |
|--------|-------|-------------------|---------------------|
| PC1    | Fa0/1 | 192.168.10.10       | 255.255.255.0        |
| PC2    | Fa0/2 (initially) | 192.168.10.11 | 255.255.255.0    |

> The switch itself doesn't need an IP address for port security to function, though a management VLAN/IP is assumed to exist for your own `show` command access — configure one if your platform requires it for console/SSH management.

---

## Tasks

### Task 1 — Build the Topology
1. Connect PC1 to Fa0/1 and PC2 to Fa0/2 on SW1.
2. Apply the addressing above, and confirm both PCs can communicate with each other and with the switch's management interface (if configured).

### Task 2 — Confirm the Port Is Currently Unrestricted
```
show port-security interface FastEthernet0/1
```
Confirm port security is currently disabled — any device could be connected to this port with no restriction.

### Task 3 — Enable Port Security on Fa0/1
Port security can only be enabled on an **access** port, not a trunk or a port still in dynamic negotiation mode — set that explicitly first:
```
interface FastEthernet0/1
 switchport mode access
 switchport port-security
```

### Task 4 — Set the Maximum MAC Address Count
```
interface FastEthernet0/1
 switchport port-security maximum 1
```
> A maximum of 1 means only a single MAC address will ever be allowed on this port — appropriate for a port with one known, dedicated device like PC1.

### Task 5 — Configure Sticky MAC Learning
Rather than manually typing PC1's MAC address, let the switch **learn** it dynamically the first time PC1 sends traffic, then convert that learned address into a permanent, saved entry:
```
interface FastEthernet0/1
 switchport port-security mac-address sticky
```

### Task 6 — Generate Traffic from PC1 to Trigger Learning
```
! From PC1
ping 192.168.10.1  (switch management IP, or any other reachable device)
```
This traffic causes the switch to learn PC1's MAC address and, because of the `sticky` keyword, automatically add it as a running-config entry.

### Task 7 — Verify the Learned Address
```
show port-security address
show run interface FastEthernet0/1
```
Confirm PC1's MAC address now appears both in the port-security address table and directly in the running configuration (as a `switchport port-security mac-address sticky <MAC>` line) — this is what "sticky" means: dynamically learned, but persisted as if statically configured.

### Task 8 — Set the Violation Mode to Restrict (First Test)
```
interface FastEthernet0/1
 switchport port-security violation restrict
```
> `restrict` drops traffic from any unauthorized MAC address and increments a violation counter, but does **not** disable the port — useful for monitoring without a full outage.

### Task 9 — Trigger a Violation
Connect **PC2** to Fa0/1 in place of PC1 (or simulate a second device appearing on the same port, depending on your platform's capabilities) and generate traffic:
```
! From PC2, now connected to Fa0/1
ping 192.168.10.1
```
This should fail, since PC2's MAC address doesn't match the sticky-learned entry for PC1.

### Task 10 — Verify the Restrict Violation
```
show port-security interface FastEthernet0/1
```
Confirm the **violation count** has incremented, and the port status remains **up** (not `err-disabled`) — consistent with `restrict` mode's "drop and log, but don't shut down" behavior.

### Task 11 — Change the Violation Mode to Shutdown and Retest
```
interface FastEthernet0/1
 switchport port-security violation shutdown
```
Reconnect PC2 (or repeat whatever action simulates an unauthorized device) and generate traffic again:
```
! From PC2
ping 192.168.10.1
```

### Task 12 — Verify the Shutdown Violation and Err-Disabled State
```
show port-security interface FastEthernet0/1
show interfaces FastEthernet0/1 status
```
Confirm the port is now in an **err-disabled** state — `shutdown` mode (the default violation mode if none is specified) doesn't just drop traffic, it disables the port entirely until manually recovered.

### Task 13 — Recover the Err-Disabled Port
Reconnect the legitimate device (PC1) before recovering the port, so it comes back up with the correct device attached:
```
interface FastEthernet0/1
 shutdown
 no shutdown
```
> Manually cycling the port is one way to recover from err-disable; many production networks instead configure **errdisable recovery** to automatically re-enable the port after a timeout, which you'll configure in the Challenge section.

### Task 14 — Verify Recovery
```
show port-security interface FastEthernet0/1
show interfaces FastEthernet0/1 status
```
Confirm the port is back to a normal, forwarding state with PC1 reconnected, and the violation counter (if your platform resets it on recovery) reflects the restored state.
```
! From PC1
ping 192.168.10.1
```
Should succeed.

### Task 15 — Final Verification
```
show port-security
show port-security interface FastEthernet0/1
show port-security address
show run interface FastEthernet0/1
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 baseline | Port security disabled; any device could connect unrestricted |
| Task 7 — sticky learning | PC1's MAC appears in both `show port-security address` and the running config |
| Task 10 — restrict violation | Violation count increments; port remains `up`; PC2's traffic is dropped |
| Task 12 — shutdown violation | Port enters `err-disabled` state; PC2's traffic fails entirely |
| Task 14 — recovery | Port restored to forwarding after `shutdown`/`no shutdown`; PC1 regains connectivity |
| `show port-security` (switch-wide) | Confirms Fa0/1's configured maximum, current count, violation mode, and security-violation count |

---

## Challenge (Optional)
- Configure **errdisable recovery** so the port automatically re-enables after a timeout instead of requiring the manual `shutdown`/`no shutdown` cycle from Task 13:
  ```
  errdisable recovery cause psecure-violation
  errdisable recovery interval 300
  ```
  Retrigger a shutdown violation and confirm the port recovers automatically after the configured interval, without manual intervention.
- Test the **protect** violation mode (the least severe of the three: drops unauthorized traffic silently, with no logging and no violation counter increment) and compare its `show port-security interface` output against the `restrict` test from this lab — specifically, note what information protect mode does *not* give you compared to restrict.
- Configure a **static** (non-sticky) MAC address entry instead, using `switchport port-security mac-address <specific-MAC>`, and discuss in your lab notes a scenario where static entry would be preferable to sticky learning (e.g., a port for a device you're provisioning before it's ever connected, where you don't want to wait for the switch to learn it dynamically).