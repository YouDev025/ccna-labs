# Lab 03: ACLs (Access Control Lists)

## Objective

Control traffic flow through a router using standard and extended Access Control Lists (ACLs), and verify that permitted and denied traffic behaves as expected.

## Background

ACLs are ordered lists of permit/deny statements used to filter traffic based on criteria such as source/destination address, protocol, and port number.

- **Standard ACLs** filter based on source address only (numbered 1–99, 1300–1999). They should be applied as close to the destination as possible.
- **Extended ACLs** filter based on source/destination address, protocol, and port (numbered 100–199, 2000–2699). They should be applied as close to the source as possible.

This lab covers both types so you can compare their behavior and placement.

## Topology

```
[Host A]---[R1 Fa0/0]---[R1]---[R1 Fa0/1]---[Host B]
10.0.0.10                                    10.0.1.10
```

- **Fa0/0:** connects to the network containing Host A (`10.0.0.0/24`)
- **Fa0/1:** connects to the network containing Host B (`10.0.1.0/24`)

## Prerequisites

- Router with at least two interfaces, each connected to a separate host/network
- Basic IP connectivity and routing already configured and verified between the two networks
- Two test hosts able to ping each other before any ACL is applied

## Steps

### 1. Create a standard ACL to permit or deny traffic

Example: deny Host A from reaching the rest of the network, while permitting everything else:

```
access-list 10 deny host 10.0.0.10
access-list 10 permit any
```

### 2. Apply it to the correct interface

Standard ACLs should be placed close to the destination. Apply it inbound on the interface facing Host B, or outbound as appropriate for your topology:

```
interface FastEthernet0/1
 ip access-group 10 out
```

*(Optional extension)* Create an extended ACL to block a specific protocol/port, such as denying Telnet from Host A to Host B while permitting other traffic:

```
access-list 110 deny tcp host 10.0.0.10 host 10.0.1.10 eq 23
access-list 110 permit ip any any
```

Apply extended ACLs close to the source:

```
interface FastEthernet0/0
 ip access-group 110 in
```

### 3. Verify behavior with `ping` and `show access-lists`

From Host A, attempt to ping Host B (should fail per the standard ACL example above):

```
ping 10.0.1.10
```

From another, non-denied host, confirm connectivity still works.

Check the ACL match counters:

```
show access-lists
```

Expected output should show hit counts on the deny and/or permit lines, similar to:

```
Standard IP access list 10
    10 deny   host 10.0.0.10 (3 matches)
    20 permit any (12 matches)
```

## Additional Verification

- `show ip interface FastEthernet0/1` — confirms which ACL is applied, and in which direction
- `debug ip packet` (use cautiously, low-traffic labs only) — to observe real-time permit/deny decisions
- Test with a second host on the same subnet as the denied host to confirm the ACL isn't overly broad

## Troubleshooting Tips

- Remember ACLs process top-down and stop at the first match — check statement order.
- Every ACL has an implicit `deny any` at the end; make sure a `permit` statement exists for traffic you want to allow.
- Standard ACLs only match source address — if traffic isn't behaving as expected, confirm you don't actually need an extended ACL.
- Double-check inbound vs. outbound direction on the interface; this is a common source of unexpected behavior.

## Deliverables

- [ ] Router configuration (ACL statements + interface application)
- [ ] Output of `show access-lists` showing match counters
- [ ] Ping results demonstrating both denied and permitted traffic