# Security Labs

This section covers basic device and network hardening, then builds toward more advanced topics. Each lab below is a standalone, self-contained exercise with its own topology, tasks, and verification checklist.

## Learning Goals for This Section
By completing this section you should be able to:
- Replace insecure Telnet with **SSH**, including key management and public-key authentication.
- Restrict which devices can even connect to a port using **port security**, including realistic voice+data scenarios.
- Centralize authentication with **AAA**, understand method lists and fallback behavior, and distinguish authentication from authorization.
- Configure **legal banners** and **brute-force login protection** correctly, understanding their real (and sometimes counterintuitive) operational effects.
- Explain the difference between **stateless** and **stateful** filtering, and configure genuine stateful inspection (ZBFW).
- Secure **VPNs** at both the traffic-policy level and the cryptographic-parameter level.
- Configure **identity-based** network access control (802.1X), distinct from simple MAC-based trust.
- Produce a **per-user-attributed audit trail** of configuration changes and login events — not just "something happened," but who did it.

## Suggested Labs (Core Set)

| Lab | Focus | Recommended Order |
|---|---|---|
| **Lab 01: SSH** | Replace Telnet, RSA host keys, local authentication | 1 |
| **Lab 02: Port Security** | MAC address limiting, sticky learning, violation modes | 2 |
| **Lab 03: AAA** | Local AAA, TACACS+ with local fallback, method lists | 3 |
| **Banner and Login** | Legal banners, exec-timeout, brute-force protection (`login block-for`) | 4 |

## Additional Labs in This Series

| Lab | Focus |
|---|---|
| **HTTPS Configuration** | PKI trustpoints, certificate management, TLS version enforcement |
| **Advanced Port Security** | Voice+data VLAN ports, MAC aging, BPDU Guard, storm control |
| **AAA Configuration** | RADIUS server groups with redundancy, command authorization, accounting |
| **SSH Key Management** | Host key fingerprint verification, public-key user authentication, key rotation |
| **Device Hardening** | Capstone: combines the core labs plus disabling unused services and unused ports |
| **Remote Access Security** | Client-based remote-access VPN, AAA/Xauth, split tunneling |
| **Firewall Concepts** | Stateless ACLs vs. reflexive ACLs vs. Zone-Based Policy Firewall |
| **VPN Security** | Weak vs. strong IKE/IPsec parameters, PFS, DPD, rejecting weak proposals |
| **Identity Access** | 802.1X, MAC Authentication Bypass (MAB), auth-fail VLAN |
| **Security Logging** | Per-user configuration-change auditing (`archive log config`), login event logging |

## How to Approach Each Lab
1. **Build the topology first** and confirm basic connectivity before applying any security control — you need a known-good baseline to know whether a later problem is caused by the security configuration or something else.
2. **Verify restrictions actually restrict**, not just that legitimate access still works. A security lab isn't complete until you've confirmed the *thing you're blocking* is actually blocked — test both sides, every time.
3. **Use distinct, named accounts** rather than shared credentials wherever a lab involves more than one "user" — several labs in this series (AAA, Security Logging) specifically depend on this to produce meaningful, individually-attributable results.
4. **Read `show run` critically.** A command being present in the configuration doesn't always mean it's actively doing what you expect — cross-check with the feature's own status command (`show ip ssh`, `show port-security interface`, `show crypto isakmp sa detail`, `show authentication sessions`) to confirm the *effective* state, not just the *configured* state.

## Core Verification Commands (Reference)

**SSH / Management Access**
```
show ip ssh
show ssh
show run | section line vty
```

**Port Security**
```
show port-security
show port-security interface <if>
show port-security address
```

**AAA**
```
show run | section aaa
show aaa servers
show aaa method-lists all
```

**Login / Banners**
```
show login
show login failures
show run | include banner
```

**General**
```
show running-config
show logging
```

## A Note on Verifying "Secure Management Practices"
When confirming a device is genuinely hardened rather than just configured, check for all of the following — each is a real gap this section's labs specifically address:
1. **Is Telnet actually disabled**, not just SSH added alongside it? (`show run | section line vty` should show `transport input ssh`, not `transport input all` or `telnet`.)
2. **Are credentials individual**, not shared? A shared admin account defeats authentication's entire purpose of proving *who* did something.
3. **Does a restriction fail closed**, not open? If a AAA server or a security feature becomes unavailable, confirm what actually happens — silently granting access is a common and dangerous failure mode to test for explicitly, not assume.
4. **Is there a record of what changed and who changed it**, not just that the device is currently configured a certain way? A configuration snapshot alone can never answer "who did this."