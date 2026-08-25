# Lab: Default Route

## Objective
Configure and verify a **default route** ("gateway of last resort") in three contexts: a simple stub router with only one way out, a static default route pointing toward a simulated ISP, and **propagating** that default route into a dynamic routing protocol (EIGRP) so downstream routers automatically learn it instead of needing their own manually-configured default.

---

## Topology

```
                              "Internet" (simulated)
                                       |
                                Gi0/1  |  203.0.113.1/29
                       +---------------+
                       |     R-EDGE    |   Has the actual default route to the ISP
                       +---------------+
                        Gi0/0  |  10.0.1.1/30
                                |
                       +---------------+
                       |    R-CORE     |   Learns the default via EIGRP, has none configured locally
                       +---------------+
                        Gi0/1  |  192.168.1.1/24
                                |
                              SW1
                                |
                          +---------+
                          |  PC1    |
                          +---------+
```

- **R-EDGE** is the only router with a real path to the outside world — a classic **stub network** scenario, the most common real-world reason a default route exists at all.
- **R-CORE** has no direct outside connectivity of its own — it should learn how to reach "everything else" purely by trusting R-EDGE's advertised default route, not by being manually configured with one itself.

---

## IP Addressing Table

| Device  | Interface | IP Address       | Subnet Mask       |
|---------|-----------|-------------------|---------------------|
| R-EDGE  | Gi0/0     | 10.0.1.1            | 255.255.255.252      |
| R-EDGE  | Gi0/1     | 203.0.113.1          | 255.255.255.248      |
| R-CORE  | Gi0/0     | 10.0.1.2            | 255.255.255.252      |
| R-CORE  | Gi0/1     | 192.168.1.1        | 255.255.255.0        |
| PC1     | NIC       | 192.168.1.10        | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology
1. Place R-EDGE, R-CORE, SW1, and PC1 as shown, and apply the addressing above.
2. Confirm baseline directly-connected reachability before configuring anything else.

### Task 2 — Configure a Static Default Route on R-EDGE
```
ip route 0.0.0.0 0.0.0.0 203.0.113.2
```
> `0.0.0.0 0.0.0.0` matches **any** destination not covered by a more specific route already in the table — this is what makes it a default route/gateway of last resort, rather than a route to a specific network.

### Task 3 — Verify R-EDGE's Gateway of Last Resort
```
show ip route
```
Confirm the output explicitly states `Gateway of last resort is 203.0.113.2 to network 0.0.0.0` at the top of the route table, and that a `S* 0.0.0.0/0` entry appears in the route list itself.

### Task 4 — Configure EIGRP Between R-EDGE and R-CORE
```
! R-EDGE
router eigrp 100
 network 10.0.1.0 0.0.0.3
 no auto-summary

! R-CORE
router eigrp 100
 network 10.0.1.0 0.0.0.3
 network 192.168.1.0 0.0.0.255
 no auto-summary
 passive-interface GigabitEthernet0/1
```
Verify the adjacency:
```
show ip eigrp neighbors
```

### Task 5 — Confirm R-CORE Has No Default Route Yet
```
show ip route
```
Confirm there is currently **no** gateway of last resort on R-CORE, and no `0.0.0.0/0` entry — R-CORE can reach R-EDGE and its own LAN, but has no idea how to reach anything beyond that yet.
```
! From PC1
ping 203.0.113.1
```
This should fail — R-CORE has nowhere to send traffic destined outside its known networks.

### Task 6 — Propagate the Default Route via EIGRP
Rather than manually configuring a second static default route on R-CORE (which would need to be kept in sync with R-EDGE forever), have R-EDGE advertise its existing default route into EIGRP automatically:
```
! On R-EDGE
router eigrp 100
 redistribute static
```
> This redistributes R-EDGE's static default route (configured in Task 2) into EIGRP, advertising it to R-CORE as a learned route rather than requiring R-CORE to have its own separately-configured default.

### Task 7 — Verify R-CORE Learns the Default Route
```
show ip route
```
Confirm R-CORE now shows a gateway of last resort, learned via EIGRP (`D*EX 0.0.0.0/0`), pointing back toward R-EDGE.
```
! From PC1
ping 203.0.113.1
```
Should now succeed.

### Task 8 — Confirm the Benefit: Change R-EDGE's Default Route and Watch R-CORE Update Automatically
```
! On R-EDGE — simulate an ISP change
no ip route 0.0.0.0 0.0.0.0 203.0.113.2
ip route 0.0.0.0 0.0.0.0 Gi0/1
```
> (Using the exit interface directly here as a simple way to force a route table change for this test; in a real scenario this would be a genuine new next-hop IP from a new ISP.)
```
! On R-CORE
show ip route
```
Confirm R-CORE's default route updated automatically, without any configuration change made directly on R-CORE — this is the core benefit of propagating a default route through a dynamic protocol rather than statically configuring it on every downstream router individually.

### Task 9 — Verify
```
show ip route
show ip protocols
show run | section router eigrp
show run | include ip route
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — R-EDGE | `Gateway of last resort` shown, pointing to the ISP address |
| Task 5 — R-CORE before propagation | No gateway of last resort; ping to the outside address fails |
| Task 7 — R-CORE after propagation | Gateway of last resort learned via EIGRP (`D*EX`); ping succeeds |
| Task 8 — R-EDGE's route changed | R-CORE's default route updates automatically with no direct configuration on R-CORE |

---

## Challenge (Optional)
- Repeat Task 6's propagation concept using OSPF instead of EIGRP (`default-information originate`, configured on R-EDGE if it were the OSPF ASBR), and compare the command and resulting route type (`O*E2`) against EIGRP's approach.
- Add a **second, less-preferred** static default route on R-EDGE toward a backup ISP-simulating interface, using a higher administrative distance (a floating default route), and verify it activates automatically if the primary is removed — connecting this lab back to the Floating Static Routes lab's concepts.
- Explain in your lab notes the difference between `ip default-gateway` (used on a Layer 2 switch with no routing capability, for its own management traffic only) and `ip route 0.0.0.0 0.0.0.0` (a true default route on a router, used for all client traffic) — these are commonly confused despite serving very different purposes.