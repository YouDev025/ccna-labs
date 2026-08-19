# Lab 01: Static Routing

## Objective
Configure **static routes** between multiple networks across a three-router topology, understand the difference between a **next-hop** static route and an **exit-interface** static route, configure a **default route**, and verify reachability while directly comparing the routing table **before and after** configuration.

---

## Topology

```
  192.168.1.0/24                 10.0.12.0/30                 10.0.23.0/30                 192.168.3.0/24
  (Branch A LAN)                (A <-> B link)                (B <-> C link)                (Branch C LAN)
        |                                                                                            |
      SW-A                                                                                          SW-C
        |                                                                                            |
  Gi0/0 | .1                                                                                  Gi0/1 | .1
  +-----------+     Gi0/1      Gi0/0     +-----------+     Gi0/1      Gi0/0     +-----------+
  |    R-A    |-----------------------------|    R-B    |-----------------------------|    R-C    |
  +-----------+     10.0.12.1  10.0.12.2    +-----------+     10.0.23.2  10.0.23.1     +-----------+
        |                                        |
      PC-A1                                 192.168.2.0/24
  192.168.1.10                              (Branch B LAN, Gi0/2 .1)
                                                    |
                                                  SW-B
                                                    |
                                                  PC-B1
                                              192.168.2.10
```

- **R-A**, **R-B**, **R-C** form a simple chain, each with its own local LAN.
- No dynamic routing protocol is used anywhere in this lab — every route between non-directly-connected networks must be added **manually**.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       | Default Gateway |
|--------|-----------|-----------------|---------------------|-----------------|
| R-A    | Gi0/0     | 192.168.1.1      | 255.255.255.0        | —               |
| R-A    | Gi0/1     | 10.0.12.1          | 255.255.255.252      | —               |
| R-B    | Gi0/0     | 10.0.12.2          | 255.255.255.252      | —               |
| R-B    | Gi0/1     | 10.0.23.2          | 255.255.255.252      | —               |
| R-B    | Gi0/2     | 192.168.2.1      | 255.255.255.0        | —               |
| R-C    | Gi0/0     | 10.0.23.1          | 255.255.255.252      | —               |
| R-C    | Gi0/1     | 192.168.3.1      | 255.255.255.0        | —               |
| PC-A1  | NIC       | 192.168.1.10      | 255.255.255.0        | 192.168.1.1     |
| PC-B1  | NIC       | 192.168.2.10      | 255.255.255.0        | 192.168.2.1     |
| PC-C1  | NIC       | 192.168.3.10      | 255.255.255.0        | 192.168.3.1     |

---

## Tasks

### Task 1 — Build the Topology and Assign Addressing
1. Place R-A, R-B, R-C, the three switches, and PC-A1, PC-B1, PC-C1 as shown.
2. Cable the inter-router links and each PC to its local switch.
3. Apply the IP addressing from the table above to every device, and `no shutdown` all router interfaces.

### Task 2 — Record the "Before" Routing Table
Before adding any static routes, capture each router's routing table as your baseline:
```
show ip route
```
On **R-A**, **R-B**, and **R-C**, you should see only **directly connected (`C`)** and **local (`L`)** routes — one pair for each interface that's up. Save this output in your lab notes; you'll compare it to the "after" state later.

### Task 3 — Test Baseline Reachability (Expect Failures)
```
! From PC-A1
ping 192.168.3.10
```
This should **fail** — R-A has no route to the 192.168.3.0/24 network yet, and even if it did, R-C wouldn't know how to route the return traffic back. This confirms the "before" state concretely, not just from reading the route table.

### Task 4 — Add Static Routes on R-A
R-A needs routes to reach both remote LANs (Branch B and Branch C) and the far inter-router link:
```
ip route 192.168.2.0 255.255.255.0 10.0.12.2
ip route 192.168.3.0 255.255.255.0 10.0.12.2
ip route 10.0.23.0 255.255.255.252 10.0.12.2
```
> All three point to the same next hop (R-B's address, 10.0.12.2) since that's R-A's only path out of its own local segment.

### Task 5 — Add Static Routes on R-B
R-B needs a route to Branch A (behind R-A) and a route to Branch C (behind R-C):
```
ip route 192.168.1.0 255.255.255.0 10.0.12.1
ip route 192.168.3.0 255.255.255.0 10.0.23.1
```

### Task 6 — Add Static Routes on R-C
R-C needs routes back to Branch A, Branch B, and the far inter-router link:
```
ip route 192.168.1.0 255.255.255.0 10.0.23.2
ip route 192.168.2.0 255.255.255.0 10.0.23.2
ip route 10.0.12.0 255.255.255.252 10.0.23.2
```

### Task 7 — Record the "After" Routing Table
```
show ip route
show ip route static
```
Compare this output directly against Task 2's baseline. You should now see additional **`S`** (static) entries on each router for every network you added — and no more, no less than what you configured. If a route is missing or shows an unexpected next hop, fix it now before continuing.

### Task 8 — Verify End-to-End Reachability
```
! From PC-A1
ping 192.168.2.10
ping 192.168.3.10

! From PC-B1
ping 192.168.1.10
ping 192.168.3.10

! From PC-C1
ping 192.168.1.10
ping 192.168.2.10
```
All six pings should now succeed, confirming full any-to-any reachability across all three branch LANs.

### Task 9 — Convert One Route to an Exit-Interface Static Route (Compare Behavior)
Static routes can be defined with a **next-hop IP** (what you've used so far) or an **exit interface**. On a point-to-point link like `Gi0/1` between R-A and R-B, an exit-interface route is also valid and behaves slightly differently under the hood (IOS treats it as directly connected for recursive lookup purposes, which affects how it interacts with some other routing features you may cover in later courses). Replace one route on R-A to demonstrate this:
```
no ip route 192.168.2.0 255.255.255.0 10.0.12.2
ip route 192.168.2.0 255.255.255.0 GigabitEthernet0/1
```
Retest:
```
! From PC-A1
ping 192.168.2.10
```
Confirm it still succeeds, and check `show ip route 192.168.2.0` to see how the route is now displayed differently (it will show as directly tied to the exit interface rather than via a next-hop IP).
> Exit-interface static routes are only safe to use on genuinely point-to-point links (like this one). On a shared/multi-access network (like an Ethernet segment with several routers), an exit-interface-only static route can cause the router to send unnecessary ARP requests for every destination on that route — always prefer a next-hop IP on multi-access networks.

### Task 10 — Configure a Default Route Instead of Individual Routes (Optional Simplification, R-C Only)
As a contrast, R-C only ever needs to send traffic toward R-B for anything not on its own local LAN — this makes it a good candidate to simplify with a single default route instead of two separate specific routes:
```
no ip route 192.168.1.0 255.255.255.0 10.0.23.2
no ip route 192.168.2.0 255.255.255.0 10.0.23.2
no ip route 10.0.12.0 255.255.255.252 10.0.23.2
ip route 0.0.0.0 0.0.0.0 10.0.23.2
```
Retest all of R-C's earlier reachability tests from Task 8 — they should all still succeed, now using a single default route instead of three specific ones. Compare `show ip route` on R-C before and after this change and note in your lab notes when a default route is (and isn't) an appropriate simplification — specifically, that it works here because R-C has only one way off its local network, which won't be true for every router in every topology.

### Task 11 — Verify
```
show ip route
show ip route static
show ip route 192.168.3.0
traceroute 192.168.3.10   (from PC-A1, to see the actual hop-by-hop path taken)
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 "before" routing table | Only `C`/`L` (connected/local) entries on each router |
| Task 3 baseline ping | Fails |
| Task 7 "after" routing table | Additional `S` entries appear, matching exactly what was configured |
| Task 8 — all six inter-branch pings | All succeed |
| Task 9 — exit-interface route | `ping 192.168.2.10` from PC-A1 still succeeds; `show ip route 192.168.2.0` displays the route tied to Gi0/1 instead of a next-hop IP |
| Task 10 — default route on R-C | All of R-C's previous reachability tests still succeed using a single `0.0.0.0/0` route instead of three specific ones |
| `traceroute` from PC-A1 to PC-C1 | Shows the expected hop-by-hop path: PC-A1 → R-A → R-B → R-C → PC-C1 |

---

## Challenge (Optional)
- Deliberately misconfigure one static route with the wrong next-hop address (e.g., a typo'd last octet) and use `show ip route` plus `traceroute` to diagnose exactly where traffic is going wrong instead of reaching its intended destination — document the specific symptom this kind of mistake produces (traffic reaching the wrong router, or timing out entirely, depending on the error).
- Add a **floating static route** (a backup route with a higher administrative distance) on R-A pointing to Branch C via a different, currently-unused path, if your topology has one available — and verify it stays out of the routing table entirely (`show ip route`) until the primary route is removed or fails, at which point it should take over automatically.
- Convert this entire lab's static configuration to a dynamic routing protocol (e.g., OSPF, if covered in your course) on all three routers, and compare the resulting `show ip route` output and configuration effort against the static approach used here — discuss in your lab notes when static routing remains preferable despite dynamic routing's automation (e.g., small/stable topologies, stub networks, precise traffic-engineering control).