# Lab: DHCP Troubleshooting

## Objective
Practice diagnosing and resolving common DHCP issues — including scope misconfiguration, DHCP relay (helper-address) problems, exhausted pools, and VLAN/interface mismatches — then verify clients successfully obtain valid IP configuration.

## Scenario
A router, **R1**, acts as the DHCP server for two client VLANs, connected through switch **SW1**. Client **PC1** (VLAN 10) is not receiving an IP address, and client **PC2** (VLAN 20) is receiving an address from the wrong subnet. You will inspect the DHCP configuration on R1 and the relay/VLAN configuration on SW1 to identify and correct the fault.

## Topology
```
        VLAN 10 (10.1.10.0/24)         VLAN 20 (10.1.20.0/24)
              |                               |
            PC1                             PC2
              \                             /
               \                           /
                [SW1] --- trunk --- [R1 (DHCP server / router-on-a-stick)]
                                       Gi0/0.10 -> VLAN 10
                                       Gi0/0.20 -> VLAN 20
```

## Prerequisites
- Console/SSH access to R1 and SW1
- Privileged EXEC (`enable`) access
- Test clients (PCs or simulated hosts) on VLAN 10 and VLAN 20

---

## Task 1: Build the Topology

1. Configure SW1 with VLANs 10 and 20, and access ports for PC1 and PC2:
   ```
   SW1(config)# vlan 10
   SW1(config-vlan)# name CLIENTS_A
   SW1(config-vlan)# exit
   SW1(config)# vlan 20
   SW1(config-vlan)# name CLIENTS_B
   SW1(config-vlan)# exit

   SW1(config)# interface FastEthernet0/1
   SW1(config-if)# switchport mode access
   SW1(config-if)# switchport access vlan 10

   SW1(config)# interface FastEthernet0/2
   SW1(config-if)# switchport mode access
   SW1(config-if)# switchport access vlan 20
   ```
2. Configure the trunk from SW1 to R1:
   ```
   SW1(config)# interface GigabitEthernet0/1
   SW1(config-if)# switchport mode trunk
   SW1(config-if)# switchport trunk allowed vlan 10,20
   ```
3. Configure router-on-a-stick subinterfaces on R1:
   ```
   R1(config)# interface GigabitEthernet0/0.10
   R1(config-subif)# encapsulation dot1Q 10
   R1(config-subif)# ip address 10.1.10.1 255.255.255.0

   R1(config)# interface GigabitEthernet0/0.20
   R1(config-subif)# encapsulation dot1Q 20
   R1(config-subif)# ip address 10.1.20.1 255.255.255.0

   R1(config)# interface GigabitEthernet0/0
   R1(config-if)# no shutdown
   ```

## Task 2: Configure Devices According to Lab Requirements

1. Exclude the gateway (and any statically-assigned) addresses from the DHCP pools:
   ```
   R1(config)# ip dhcp excluded-address 10.1.10.1 10.1.10.10
   R1(config)# ip dhcp excluded-address 10.1.20.1 10.1.20.10
   ```
2. Create DHCP pools matching each VLAN's subnet:
   ```
   R1(config)# ip dhcp pool VLAN10_POOL
   R1(dhcp-config)# network 10.1.10.0 255.255.255.0
   R1(dhcp-config)# default-router 10.1.10.1
   R1(dhcp-config)# dns-server 8.8.8.8
   R1(dhcp-config)# exit

   R1(config)# ip dhcp pool VLAN20_POOL
   R1(dhcp-config)# network 10.1.20.0 255.255.255.0
   R1(dhcp-config)# default-router 10.1.20.1
   R1(dhcp-config)# dns-server 8.8.8.8
   R1(dhcp-config)# exit
   ```
3. If the DHCP server is **not** directly on the client's subnet (a separate DHCP server elsewhere in the network), configure a relay/helper address on the client-facing SVI or subinterface instead:
   ```
   R1(config-subif)# ip helper-address 10.1.10.1
   ```

---

## Task 3: Verify Service Behavior and Connectivity

### 3.1 Confirm DHCP Pools and Bindings on R1
```
R1# show ip dhcp pool
R1# show ip dhcp binding
R1# show ip dhcp server statistics
```
Check for:
- **Pool subnet matches the VLAN subnet** — a common fault is a pool configured for the wrong network (e.g., VLAN 20 pool accidentally using `10.1.10.0/24`), which explains PC2 getting an address from the wrong subnet.
- **Address exhaustion** — `show ip dhcp server statistics` shows addresses in use vs. pool size.
- **Excluded-address range doesn't accidentally exclude the entire usable range.**

### 3.2 Confirm Interface/VLAN Configuration
```
R1# show ip interface brief
R1# show vlans
SW1# show vlan brief
SW1# show interfaces trunk
```
Confirm:
- Subinterfaces are `up/up` with correct dot1Q tags.
- SW1's trunk allows both VLAN 10 and VLAN 20.
- PC1 and PC2's access ports are in the correct VLANs.

### 3.3 Check for DHCP Snooping / Security Features Blocking Requests
If DHCP snooping is enabled on SW1, confirm the uplink to R1 is trusted:
```
SW1# show ip dhcp snooping
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# ip dhcp snooping trust
```
An untrusted uplink will silently drop DHCP server responses, which looks identical to "PC1 is not receiving an IP address."

### 3.4 Test from the Client Side
On PC1 and PC2 (or simulated host), release and renew the lease:
```
PC1> ipconfig /release
PC1> ipconfig /renew
```
Or on Cisco IOS-based test clients:
```
PC1# ip dhcp client renew
```
Confirm the assigned address, subnet mask, default gateway, and DNS server are correct for the VLAN.

### 3.5 Packet-Level Verification (If Available)
Use `debug` cautiously (lab environment only, not production):
```
R1# debug ip dhcp server events
```
Look for `DHCPDISCOVER`, `DHCPOFFER`, `DHCPREQUEST`, and `DHCPACK` messages to confirm the four-way handshake completes for each client. Absence of a `DHCPDISCOVER` reaching R1 points to a Layer 2/trunk/relay problem upstream, not the DHCP configuration itself.

---

## Success Criteria
- PC1 receives an IP address in `10.1.10.0/24` with gateway `10.1.10.1`.
- PC2 receives an IP address in `10.1.20.0/24` with gateway `10.1.20.1`.
- `show ip dhcp binding` on R1 lists active leases for both clients with correct subnets.
- No DHCP snooping or trunk misconfiguration is blocking DORA (Discover-Offer-Request-Ack) messages.
- Both clients can ping their default gateway and each other's subnet (if routing permits).

## Troubleshooting Checklist (Quick Reference)
- [ ] DHCP pool subnet matches the client VLAN's subnet
- [ ] Excluded-address range is correct (not over-excluding)
- [ ] Subinterfaces/SVIs are `up/up` with correct VLAN tagging
- [ ] Trunk allows all required VLANs between switch and router
- [ ] `ip helper-address` configured if DHCP server is off-subnet
- [ ] DHCP snooping trust configured on uplink ports (if snooping is enabled)
- [ ] Pool not exhausted (`show ip dhcp server statistics`)
- [ ] DORA process completes (verified via `debug ip dhcp server events`)

## Cleanup
```
R1# copy running-config startup-config
SW1# copy running-config startup-config
```

> **Note:** Disable any `debug` commands used during troubleshooting before finishing the lab, as they can be CPU-intensive:
> ```
> R1# undebug all
> ```