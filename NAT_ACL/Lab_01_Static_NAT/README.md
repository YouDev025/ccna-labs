# Lab 01: Static NAT

## Objective

Configure a static (one-to-one) NAT translation between an inside private address and an outside public address, and verify that traffic is correctly translated as it crosses the NAT boundary.

## Background

Static NAT creates a permanent, one-to-one mapping between a specific inside local (private) address and an inside global (public) address. Unlike dynamic NAT or PAT, this mapping does not expire and is commonly used to make an internal resource — such as a server — permanently reachable from outside the network using a fixed public IP.

## Topology

A simple topology is sufficient for this lab:

```
[Inside Host/Server] --- [Router (NAT device)] --- [Outside Host]
      10.0.0.10                                       203.0.113.10
```

- **Inside interface:** connects to the private network (e.g., `10.0.0.0/24`)
- **Outside interface:** connects to the public/simulated internet network (e.g., `203.0.113.0/24`)

## Prerequisites

- Router with at least two interfaces (inside and outside)
- Basic IP connectivity already configured and verified on both interfaces
- An inside host/server with a known private IP address to be translated

## Steps

### 1. Define the static NAT mapping

Create a one-to-one mapping between the inside local address and the inside global address:

```
ip nat inside source static 10.0.0.10 203.0.113.10
```

### 2. Apply the correct interface configuration

Designate which interfaces are "inside" and which are "outside" for NAT purposes:

```
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.0
 ip nat outside
```

### 3. Verify with `show ip nat translations`

Generate traffic from the inside host (e.g., ping an outside host) and confirm the translation appears:

```
show ip nat translations
```

Expected output should show a static entry similar to:

```
Pro  Inside global      Inside local       Outside local      Outside global
---  203.0.113.10       10.0.0.10          ---                ---
```

## Additional Verification

- `show ip nat statistics` — confirms hits/misses and interface roles
- `debug ip nat` — useful for real-time translation troubleshooting (disable after testing)
- Ping/traceroute from the outside host to `203.0.113.10` to confirm the internal server is reachable via its public address

## Troubleshooting Tips

- Confirm `ip nat inside` and `ip nat outside` are applied to the correct interfaces.
- Ensure routing exists for the outside global address range.
- Check that no overlapping dynamic NAT/PAT pool includes the same address.
- Verify no ACLs are blocking traffic on either interface.

## Deliverables

- [ ] Router configuration (NAT mapping + interface config)
- [ ] Output of `show ip nat translations` showing the static entry
- [ ] Confirmation of successful ping/traceroute across the NAT boundary