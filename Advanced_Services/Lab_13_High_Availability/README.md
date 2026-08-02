# Lab High Availability

## Objective

This lab introduces **High Availability (HA)** concepts by implementing a redundant network infrastructure between a Headquarters site and a Branch Office. The objective is to provide continuous network connectivity using redundant routers, HSRP (Hot Standby Router Protocol), and multiple WAN links.

---

## Network Topology

### Devices

- 5 Cisco 2911 Routers
- 2 Cisco 2960 Switches
- 4 PCs
- Serial WAN Links
- Copper Straight-Through Cables

### Topology Overview

```
                  Headquarters

        PC0                  PC1
         |                    |
      +-------- Switch0 -------+
               |         |
            Router0    Router1
          (HSRP Active)(HSRP Standby)
               \         /
                \       /
                 \     /
                  Router2
                (ISP / WAN)
                 /     \
                /       \
               /         \
          Router3      Router4
      (HSRP Active) (HSRP Standby)
              |          |
          +------ Switch1 ------+
           |                  |
         PC2                PC3

               Branch Office
```

---

## IP Addressing

### Headquarters LAN

| Device | Interface | Address |
|---------|-----------|----------|
| Router0 | G0/0/0 | 192.168.10.2 /24 |
| Router1 | G0/0/0 | 192.168.10.3 /24 |
| HSRP Virtual Gateway | - | **192.168.10.1** |
| PC0 | Fa0 | 192.168.10.10 /24 |
| PC1 | Fa0 | 192.168.10.20 /24 |

---

### Branch LAN

| Device | Interface | Address |
|---------|-----------|----------|
| Router3 | G0/0/0 | 192.168.20.2 /24 |
| Router4 | G0/0/0 | 192.168.20.3 /24 |
| HSRP Virtual Gateway | - | **192.168.20.1** |
| PC2 | Fa0 | 192.168.20.10 /24 |
| PC3 | Fa0 | 192.168.20.20 /24 |

---

### WAN Links

| Link | Network |
|------|---------|
| Router0 ↔ ISP | 10.0.0.0/30 |
| Router1 ↔ ISP | 10.0.0.4/30 |
| Router3 ↔ ISP | 10.0.0.8/30 |
| Router4 ↔ ISP | 10.0.0.12/30 |
| Router0 ↔ Router1 | 10.0.1.0/30 |
| Router3 ↔ Router4 | 10.0.1.4/30 |

---

## Tasks

- Build the High Availability topology.
- Configure IP addressing.
- Configure HSRP on Headquarters routers.
- Configure HSRP on Branch routers.
- Configure serial WAN links.
- Configure static routes between sites.
- Configure floating static routes for backup paths.
- Verify redundancy and failover.
- Test end-to-end connectivity.

---

## HSRP Configuration

### Headquarters

- Router0: Active Router
- Router1: Standby Router
- Virtual Gateway: **192.168.10.1**

### Branch Office

- Router3: Active Router
- Router4: Standby Router
- Virtual Gateway: **192.168.20.1**

---

## Routing

Static routing is used to reach the remote LANs.

Floating static routes provide backup routing paths with a higher administrative distance.

---

## Verification Commands

### Interface Status

```bash
show ip interface brief
```

### Routing Table

```bash
show ip route
```

### HSRP Status

```bash
show standby
```

```bash
show standby brief
```

### Running Configuration

```bash
show running-config
```

### Connectivity Test

```bash
ping <destination-ip>
```

### Traceroute

```bash
tracert <destination-ip>
```

---

## Failover Test

1. Start a continuous ping from **PC0** to **PC2**.
2. Shut down the active router interface:

```bash
conf t
interface GigabitEthernet0/0/0
shutdown
```

3. Verify that the standby router becomes active.

```bash
show standby brief
```

4. Confirm that network connectivity is restored automatically after failover.

---

## Expected Results

- All interfaces are operational.
- PCs can communicate across the WAN.
- HSRP virtual gateways respond to ping.
- Active and Standby router roles are correctly assigned.
- Static routes are present in the routing tables.
- Connectivity is maintained during router failover.
- Floating static routes are available as backup paths.

---

## Skills Practiced

- High Availability (HA)
- HSRP Configuration
- Gateway Redundancy
- Static Routing
- Floating Static Routes
- WAN Configuration
- IP Addressing
- Network Troubleshooting
- Connectivity Verification
- Cisco IOS Configuration

---

## Conclusion

This lab demonstrates how High Availability techniques improve network reliability by introducing gateway redundancy, redundant WAN connections, and backup routing paths. Using HSRP and redundant links, the network remains operational even when a router or link becomes unavailable, ensuring minimal service interruption.