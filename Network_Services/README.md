# Network Services Labs

This section covers services commonly used in enterprise networks. Each lab below is a standalone, self-contained exercise with its own topology, addressing plan, step-by-step tasks, and verification checklist — build them in order if you're new to these services, or jump to whichever one you need to practice.

## Learning Goals for This Section
By completing this section you should be able to:
- Configure and verify **DHCP** (IPv4 and IPv6) address assignment, including exclusions and lease behavior.
- Configure and verify **DNS** name resolution, both as a server and as a client, and understand basic DNS hardening.
- Configure and verify **NTP** time synchronization, including redundancy and authentication.
- Configure and verify **Syslog** logging, and — just as importantly — **read and interpret** the log output you generate.
- Configure and verify **SNMP** monitoring, progressing from basic community strings to properly secured SNMPv3.
- Recognize how these services **depend on each other** — accurate time makes logs trustworthy; DHCP depends on the router's clock for lease timing; DNS underpins nearly every other service's usability.

## Suggested Labs

| Lab | Focus | Recommended Order |
|---|---|---|
| **DHCP** | IPv4 pool configuration, exclusions, binding verification | 1 |
| **DNS** | DNS server basics, client resolution, static host entries | 2 |
| **NTP** | Basic client/server sync, hierarchy, `show ntp associations` | 3 |
| **Syslog Configuration** | Severity levels, remote logging, timestamps | 4 |
| **SNMP Configuration** | v2c basics, ACL restriction, upgrading to SNMPv3 | 5 |

> Additional labs available in this series if you want to go further: **DHCPv6** (SLAAC vs. stateless vs. stateful, M/O flags), **DNS Security** (open-resolver prevention, sinkholing), **NTP Authentication** (rogue-source rejection, key mismatch), **Syslog Analysis** (reading and correlating logs across devices), **Network Time Services** (redundancy, failover, and why accurate time matters to other services), and **HTTP Service Verification** (securing router management and testing hosted web services).

## How to Approach Each Lab
1. **Build the topology first**, with no service configured, and confirm baseline IP reachability. A service that "doesn't work" is often actually a routing or cabling problem underneath it.
2. **Configure exactly what the lab specifies** before improvising — later verification steps are written assuming the tasks were done in order.
3. **Verify the service at more than one layer.** A successful `ping` only proves basic IP reachability — it does not prove DHCP handed out a correct lease, DNS resolved to the right address, NTP is actually synchronized (not just configured), or SNMP is answering with valid data. Always follow up with the service-specific `show` command.
4. **Read the actual output**, don't just check that a command didn't error. `show ntp associations` needs a `*`/`sys.peer` marker to mean "synchronized," not just "configured." `show ip dhcp binding` needs an entry with a sensible lease time. `show logging` needs to contain the message you expected, not just any output.

## Core Verification Commands (Reference)

**DHCP**
```
show ip dhcp binding
show ip dhcp pool
show ip dhcp server statistics
show ip dhcp conflict
```

**DNS**
```
show hosts
show running-config | include ip name-server
nslookup <name>
```

**NTP**
```
show ntp status
show ntp associations
show ntp associations detail
show clock detail
```

**Syslog**
```
show logging
show logging | include <keyword>
show run | section logging
```

**SNMP**
```
show snmp
show snmp community
show snmp user
show snmp host
```

**General connectivity (use before troubleshooting any service above)**
```
ping <destination>
show ip interface brief
show ip route
```

## A Note on Troubleshooting Method
When a service in this section doesn't behave as expected, work through these in order rather than guessing:
1. Is basic IP reachability working, independent of the service? (Ping the relevant device directly.)
2. Is the service actually enabled and correctly configured? (`show run | section <service>`)
3. Is a security control (ACL, authentication, community string, access-class) blocking the request before it ever reaches the intended logic?
4. Does the service's own `show` command confirm success with specific detail — a real binding, a real association, a real resolved address — not just "no errors reported"?
5. If multiple devices are involved, are their clocks synchronized enough that timestamps across devices can be meaningfully compared? (This is why NTP is placed early in the recommended order — several of the other labs' verification steps are easier to trust once it's working.)