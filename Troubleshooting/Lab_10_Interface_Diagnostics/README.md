# Lab: Interface Diagnostics

## Objective
Practice systematic interface-level troubleshooting: identifying physical layer errors, duplex/speed mismatches, and interface status problems, then verifying that fixes restore normal operation.

## Scenario
Two routers/switches, **R1** and **R2**, are connected via a point-to-point link. Users report slow or intermittent connectivity across the link. You will inspect interface counters and status to identify the root cause (e.g., errors, collisions, duplex mismatch, or an administratively/physically down interface), correct it, and confirm the link returns to healthy operation.

## Topology
```
   [R1] ---- Gig0/0 ---- Link ---- Gig0/0 ---- [R2]
    |                                           |
  LAN A (10.1.1.0/24)                   LAN B (10.1.2.0/24)
```

## Prerequisites
- Console/SSH access to R1 and R2
- Privileged EXEC (`enable`) access
- Cabling diagram or physical topology reference

---

## Task 1: Build the Topology

1. Physically (or virtually, e.g., in GNS3/Packet Tracer/EVE-NG) connect R1 and R2 via their `Gig0/0` interfaces.
2. Assign IP addresses:
   ```
   R1(config)# interface GigabitEthernet0/0
   R1(config-if)# ip address 10.1.1.1 255.255.255.0
   R1(config-if)# no shutdown

   R2(config)# interface GigabitEthernet0/0
   R2(config-if)# ip address 10.1.2.1 255.255.255.0
   R2(config-if)# no shutdown
   ```
3. Confirm both interfaces are administratively up:
   ```
   R1# show ip interface brief
   R2# show ip interface brief
   ```

## Task 2: Configure Devices According to Lab Requirements

1. Set interface descriptions for documentation clarity:
   ```
   R1(config-if)# description Link to R2
   R2(config-if)# description Link to R1
   ```
2. Explicitly set speed and duplex to match on both ends (avoid relying solely on auto-negotiation when troubleshooting):
   ```
   R1(config-if)# speed 1000
   R1(config-if)# duplex full

   R2(config-if)# speed 1000
   R2(config-if)# duplex full
   ```
   > If both ends must auto-negotiate, ensure **both** sides are set to `speed auto` / `duplex auto` — a mismatch (one fixed, one auto) is a leading cause of late collisions and CRC errors.
3. Apply any additional lab-specific configuration (VLANs, ACLs, routing protocols) as required by your environment.

## Task 3: Verify Service Behavior and Connectivity

### 3.1 Check Interface Status and Counters
```
R1# show interfaces GigabitEthernet0/0
```
Review key fields:
- **Line protocol status**: should read `up/up`. `up/down` suggests a Layer 2 issue (e.g., encapsulation or keepalive mismatch); `down/down` suggests a physical/cabling problem.
- **Duplex/Speed**: confirm they match the configuration and the remote end.
- **Input/output errors**: look at `CRC`, `frame`, `runts`, `giants`, `input errors`, `output errors`, and `collisions`.

| Counter | Likely Cause |
|---|---|
| CRC errors | Bad cable, electrical interference, or duplex mismatch |
| Late collisions | Duplex mismatch |
| Runts/Giants | Cabling issue, faulty NIC, or MTU mismatch |
| Input queue drops | Interface congestion / oversubscription |
| Interface resets | Flapping link, power issue, or failing hardware |

### 3.2 Clear and Re-Baseline Counters
```
R1# clear counters GigabitEthernet0/0
```
Wait a short interval, then re-check to see if errors are actively incrementing:
```
R1# show interfaces GigabitEthernet0/0 | include errors|CRC|collision
```

### 3.3 Confirm Layer 3 Reachability
```
R1# ping 10.1.2.1
R1# traceroute 10.1.2.1
```

### 3.4 Extended Diagnostics
```
R1# show interfaces GigabitEthernet0/0 counters errors
R1# show controllers GigabitEthernet0/0
R1# show interfaces status
```
Use `show controllers` to check for physical-layer signal issues if errors persist despite correct configuration.

### 3.5 Confirm CDP/LLDP Neighbor Visibility (Sanity Check)
```
R1# show cdp neighbors detail
R1# show lldp neighbors detail
```
Confirms the physical link is correctly connected to the intended remote device.

---

## Success Criteria
- Both interfaces show `up/up` in `show ip interface brief`.
- Speed and duplex match on both ends with no mismatch warnings.
- Error/collision counters are at zero or not actively incrementing after a fresh `clear counters`.
- End-to-end ping and traceroute between R1 and R2 succeed.
- CDP/LLDP confirms the expected neighbor relationship.

## Troubleshooting Checklist (Quick Reference)
- [ ] Interface administratively up (`no shutdown` applied)
- [ ] Cable/physical layer verified (`show controllers`, link lights)
- [ ] Speed and duplex match on both ends
- [ ] No incrementing CRC, runts, giants, or collision counters
- [ ] Line protocol status is `up`
- [ ] CDP/LLDP shows the correct neighbor
- [ ] Layer 3 addressing correct and ping/traceroute succeed

## Cleanup
```
R1# copy running-config startup-config
R2# copy running-config startup-config
```