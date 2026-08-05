# Lab 02: PAT (Port Address Translation)

## Objective

Configure Port Address Translation (PAT), also known as NAT overload, to allow multiple inside private hosts to share a single outside public IP address for outbound internet access.

## Background

PAT extends dynamic NAT by adding port numbers to the translation table, allowing many inside local addresses to be mapped to a single inside global address simultaneously. This is the most common form of NAT used in home and enterprise networks, since it conserves public IP addresses while still allowing many internal hosts to reach the internet.

## Topology

```
[Inside Hosts]  ---  [Router (NAT device)]  ---  [Outside Network]
10.0.0.0/24                                        203.0.113.1 (outside interface)
```

- **Inside interface:** connects to the private network (e.g., `10.0.0.0/24`)
- **Outside interface:** connects to the public/simulated internet network, using its own IP as the overload address (e.g., `203.0.113.1`)

## Prerequisites

- Router with at least two interfaces (inside and outside)
- Basic IP connectivity already configured and verified on both interfaces
- Multiple inside hosts (or simulated hosts) available for testing

## Steps

### 1. Define the access list for translated traffic

Create a standard access list to identify which inside source addresses are eligible for translation:

```
access-list 1 permit 10.0.0.0 0.0.0.255
```

### 2. Configure overload NAT

Enable PAT by referencing the access list and the outside interface, using the `overload` keyword to allow many-to-one translation:

```
ip nat inside source list 1 interface GigabitEthernet0/1 overload
```

Apply the inside/outside roles to the interfaces:

```
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.0
 ip nat outside
```

### 3. Verify with `show ip nat translations` and `ping`

From an inside host, generate traffic to an outside destination:

```
ping 203.0.113.254
```

Then check the translation table:

```
show ip nat translations
```

Expected output should show multiple inside hosts translated to the same global address, differentiated by port number, similar to:

```
Pro  Inside global         Inside local        Outside local       Outside global
icmp 203.0.113.1:1         10.0.0.10:1         203.0.113.254:1     203.0.113.254:1
icmp 203.0.113.1:2         10.0.0.11:2         203.0.113.254:2     203.0.113.254:2
```

## Additional Verification

- `show ip nat statistics` — confirms total translations, hits/misses, and interface roles
- `debug ip nat` — useful for real-time translation troubleshooting (disable after testing)
- Test from multiple inside hosts simultaneously to confirm they all share the single outside address

## Troubleshooting Tips

- Confirm `ip nat inside` and `ip nat outside` are applied to the correct interfaces.
- Double-check the access list matches the correct inside source range.
- Ensure a default route (or appropriate route) exists toward the outside network.
- If translations aren't appearing, confirm traffic is actually reaching the router by checking interface counters.

## Deliverables

- [ ] Router configuration (ACL, NAT overload statement, interface config)
- [ ] Output of `show ip nat translations` showing multiple hosts sharing one address
- [ ] Successful ping results from at least two different inside hosts