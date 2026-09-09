# Switching Technologies Labs

This section is for CCNA switching practice. Each lab below is a standalone, self-contained exercise with its own topology, tasks, and verification checklist. Several labs deliberately build directly on each other — VLAN Basics leaves inter-VLAN devices unable to reach each other on purpose, and Router on a Stick / Layer 3 Switching are the direct payoff to that setup — so working through them roughly in order will make the connections click into place rather than feel like isolated exercises.

## Learning Goals for This Section
By completing this section you should be able to:
- Create **VLANs**, assign access ports correctly, and explain why different-VLAN devices can't reach each other without a router.
- Configure **trunking**, understand DTP negotiation modes, and diagnose native VLAN mismatches.
- Explain **STP** root election, port roles, and convergence — for both traditional 802.1D and the much faster Rapid PVST+ — plus **MST** as the scalable alternative for many-VLAN networks.
- Configure **port security** and layer it with STP protections (BPDU Guard, Root Guard) and traffic-volume protections (storm control) for a genuinely hardened access layer.
- Route between VLANs two different ways — **router on a stick** and **Layer 3 switching (SVIs)** — and explain the bandwidth tradeoff between them.
- Approach **VLAN design** as a deliberate planning exercise, not just configuration syntax.

## Suggested Labs (Core Set)

| Lab | Focus | Recommended Order |
|---|---|---|
| **Lab 01: VLAN Basics** | Create VLANs, access ports, trunk between two switches, confirm isolation | 1 |
| **Lab 02: Trunking** | DTP negotiation modes, native VLAN mismatch detection and hardening | 2 |
| **Lab 03: STP** | Root election, port roles, PortFast/BPDU Guard, convergence timing | 3 |
| **Lab 02: Port Security** *(Security Labs section)* | MAC limiting, sticky learning, violation modes | 4 |
| **Router on a Stick** | Inter-VLAN routing via a single trunk link and router subinterfaces | 5 |

> Port Security lives in the Security Labs section of this series but is included here in the recommended order since it's core CCNA switching material — see that section for the full lab.

## Additional Labs in This Series

| Lab | Focus |
|---|---|
| **Access Port Configuration** | Switch spoofing VLAN hopping attack, `nonegotiate` hardening |
| **Advanced Port Security** *(Security Labs)* | Voice+data VLAN ports, MAC aging, BPDU Guard, storm control |
| **Rapid PVST+** | RSTP port roles/states, fast convergence, per-VLAN load balancing |
| **MSTP Configuration** | Region definition, VLAN-to-instance mapping, region mismatch |
| **Dynamic Access VLANs** | RADIUS-assigned VLAN per authenticated identity |
| **VLAN Trunk Pruning** | Manual vs. automatic (VTP) pruning of unnecessary VLAN flooding |
| **Layer 3 Switching** | SVIs, `ip routing`, routed ports — the scalable alternative to router-on-a-stick |
| **Storm Control** | Traffic-volume protection independent of STP, including a scenario STP can't catch at all |
| **Switch Security Best Practices** | Capstone: Root Guard, DHCP Snooping, Dynamic ARP Inspection combined |
| **VLAN Design** | Design-first lab: numbering, sizing, and management VLAN planning from a business scenario |

## How to Approach Each Lab
1. **Build the topology in Packet Tracer or GNS3** exactly as specified, and confirm the "before" state (often a deliberate failure — a blocked ping, an unformed trunk) before making any changes, so you have real evidence of what each configuration step actually fixed.
2. **Read `show vlan brief` and `show interfaces trunk` critically.** A VLAN existing in the VLAN database doesn't mean a given port is assigned to it, and a trunk being up doesn't mean the VLAN you care about is actually allowed or unpruned on it — check the specific detail, not just that the command ran without error.
3. **Distinguish Layer 2 problems from Layer 3 problems.** If a ping fails, first confirm VLAN assignment and trunk status (`show vlan brief`, `show interfaces trunk`) before troubleshooting IP addressing or routing — many "routing" problems in this section are actually Layer 2 issues underneath.
4. **Expect some tests to correctly fail.** Several labs in this section (VLAN Basics, Storm Control's "before" state) have a step where the *correct* result is a failure — don't treat every failed ping as a mistake to fix; read the task carefully to know which outcome is expected.

## Core Verification Commands (Reference)

**VLANs and Access Ports**
```
show vlan brief
show interfaces switchport
show interfaces status
```

**Trunking**
```
show interfaces trunk
show interfaces <if> switchport
show cdp neighbors detail
```

**STP**
```
show spanning-tree vlan <id>
show spanning-tree summary
show spanning-tree interface <if> detail
show spanning-tree inconsistentports
```

**Port Security / Storm Control**
```
show port-security interface <if>
show storm-control broadcast
```

**Inter-VLAN Routing**
```
show ip interface brief
show ip route
```

## A Note on Diagnosing Switching Problems
When something doesn't work as expected in this section, work through these in order:
1. **Is the port in the VLAN you think it's in?** (`show vlan brief`) A port left in the default VLAN, or assigned to the wrong one, is one of the most common root causes in this entire section.
2. **Is the trunk actually trunking, and carrying the VLAN you need?** (`show interfaces trunk`) Check both the negotiated mode and the allowed/pruned VLAN lists specifically — a trunk can be "up" while still not carrying the traffic you expect.
3. **Do both ends of a trunk agree on the native VLAN?** A mismatch doesn't always break connectivity outright, but causes exactly the kind of quiet, hard-to-diagnose problem worth checking for specifically (`show logging | include NATIVE_VLAN`).
4. **Is STP blocking a port you expected to be forwarding** (or, more subtly, is STP unable to see a problem at all, as in the Storm Control lab's BPDU-filtered scenario)? Check `show spanning-tree vlan <id>` for the specific port's role and state.
5. **Only after Layer 2 is confirmed healthy**, move on to IP addressing, gateway configuration, and routing table checks — troubleshooting Layer 3 on top of a broken Layer 2 foundation wastes time and produces confusing, misleading symptoms.