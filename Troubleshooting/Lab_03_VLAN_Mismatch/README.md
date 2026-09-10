# Lab 03: VLAN Mismatch Troubleshooting

## Objective
Identify and correct a VLAN trunking mismatch between two switches that is preventing end-to-end connectivity, and verify the fix restores traffic flow.

## Scenario
Two access-layer switches, **SW1** and **SW2**, are connected via a trunk link. Hosts on VLAN 10 (PCs) and VLAN 20 (servers) can no longer reach their counterparts across the switches. You suspect a trunk misconfiguration — either an allowed-VLAN mismatch, a native VLAN mismatch, or a trunk encapsulation issue.

## Topology
```
   PC1 (VLAN 10)         Server1 (VLAN 20)
        |                        |
     [SW1] ---- Trunk Link ---- [SW2]
        |                        |
   PC2 (VLAN 10)         Server2 (VLAN 20)
```

## Prerequisites
- Access to SW1 and SW2 (console, Telnet, or SSH)
- Basic familiarity with Cisco IOS CLI
- Privileged EXEC (`enable`) access on both switches

---

## Step 1: Inspect Switchport Trunk Configuration

On **each** switch, check the trunk interface configuration and current operational state.

```
SW1# show running-config interface GigabitEthernet0/1
SW1# show interfaces trunk
```

Repeat on SW2:
```
SW2# show running-config interface GigabitEthernet0/1
SW2# show interfaces trunk
```

Look specifically for:
- **Trunk mode**: `switchport mode trunk` — confirm both ends are hard-set to trunk (not `dynamic auto`/`dynamic desirable` on both, which can fail to negotiate).
- **Encapsulation**: `switchport trunk encapsulation dot1q` — must match if the hardware supports ISL/dot1q negotiation.
- **Allowed VLANs**: `switchport trunk allowed vlan <list>` — confirm both switches permit the same VLANs across the trunk.
- **Native VLAN**: `switchport trunk native vlan <id>` — must be identical on both ends.

> **Common symptom of a native VLAN mismatch:** CDP will log a warning similar to:
> `%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on GigabitEthernet0/1`

Check the logs:
```
SW1# show logging | include NATIVE_VLAN
```

---

## Step 2: Compare VLAN Assignments on Both Ends

Confirm the VLAN database and access port assignments match your design on both switches.

```
SW1# show vlan brief
SW2# show vlan brief
```

Verify:
1. VLAN 10 and VLAN 20 exist in the VLAN database on **both** switches (a VLAN missing from one switch's database will silently drop traffic for that VLAN across the trunk).
2. Access ports connecting to PCs/servers are assigned to the correct VLAN:
   ```
   SW1# show interfaces FastEthernet0/1 switchport
   ```
3. The trunk's allowed-VLAN list includes both VLAN 10 and VLAN 20 on both switches:
   ```
   SW1# show interfaces GigabitEthernet0/1 trunk
   SW2# show interfaces GigabitEthernet0/1 trunk
   ```

**Typical mismatches to look for:**
| Symptom | Likely Cause |
|---|---|
| One VLAN works, the other doesn't | VLAN missing from `allowed vlan` list on one side |
| Intermittent/broadcast storm-like behavior | Native VLAN mismatch (untagged traffic leaking between VLANs) |
| Trunk not forming at all (`show interfaces trunk` shows nothing) | Mismatched trunk mode/encapsulation, or VTP domain mismatch |
| VLAN exists on one switch but not the other | VLAN not created, or VTP pruning removed it |

### Fixing an Allowed-VLAN Mismatch
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport trunk allowed vlan add 10,20
```

### Fixing a Native VLAN Mismatch
Set the same native VLAN on both ends (commonly VLAN 1 or a dedicated native VLAN, e.g., 99):
```
SW1(config-if)# switchport trunk native vlan 99
SW2(config-if)# switchport trunk native vlan 99
```

### Fixing a Missing VLAN
```
SW1(config)# vlan 20
SW1(config-vlan)# name SERVERS
SW1(config-vlan)# exit
```

---

## Step 3: Verify Connectivity After Fixing the Mismatch

1. Re-check trunk status on both switches:
   ```
   SW1# show interfaces trunk
   SW2# show interfaces trunk
   ```
   Confirm VLANs 10 and 20 appear in the "Vlans allowed and active in management domain" and "Vlans in spanning tree forwarding state" columns.

2. Confirm no native VLAN mismatch errors remain:
   ```
   SW1# show logging | include NATIVE_VLAN
   ```

3. Test end-to-end connectivity from the hosts:
   ```
   PC1> ping <PC2 IP address>        (VLAN 10 to VLAN 10)
   Server1> ping <Server2 IP address> (VLAN 20 to VLAN 20)
   ```

4. Confirm MAC address learning is correct on both switches:
   ```
   SW1# show mac address-table vlan 10
   SW1# show mac address-table vlan 20
   ```

5. (Optional) Use extended ping or traceroute to confirm the path traverses the trunk as expected:
   ```
   SW1# traceroute <destination IP>
   ```

---

## Success Criteria
- `show interfaces trunk` shows identical allowed-VLAN and native-VLAN configuration on both switches.
- No `NATIVE_VLAN_MISMATCH` messages appear in the logs.
- PC1 ↔ PC2 (VLAN 10) and Server1 ↔ Server2 (VLAN 20) ping successfully.
- MAC address tables on both switches show correct VLAN-to-port associations.

## Troubleshooting Checklist (Quick Reference)
- [ ] Trunk mode matches on both ends (`switchport mode trunk`)
- [ ] Encapsulation matches (if configurable)
- [ ] Allowed VLAN list matches on both ends
- [ ] Native VLAN matches on both ends
- [ ] VLANs exist in the VLAN database on both switches
- [ ] Access ports are assigned to the correct VLAN
- [ ] No VTP domain/mode conflicts causing VLAN pruning
- [ ] Spanning tree is forwarding (not blocking) the trunk for the affected VLANs

## Cleanup
Once verified, save the configuration:
```
SW1# copy running-config startup-config
SW2# copy running-config startup-config
```