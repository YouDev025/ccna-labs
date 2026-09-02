# Lab: Remote Access Security

## Objective
This lab covers **remote-access VPN** — a single remote user's device connecting into the corporate network from an untrusted location (home, a coffee shop, a hotel) — which is a genuinely different problem from the **site-to-site** VPN covered in the VPN ACLs lab (two fixed office routers connecting their networks together). You'll configure a router as a VPN gateway that authenticates individual remote users, hands out an internal address from a dedicated pool, and correctly implements **split tunneling** so only corporate-bound traffic actually goes through the tunnel — general internet traffic from the remote user continues to go directly out their own local connection, not through the corporate network.

---

## Topology

```
   Corporate LAN                                              "Internet" (simulated)                Remote Worker
   192.168.50.0/24                                                                                    "Home" network
        |                                                                                              203.0.113.10/29
      SW-CORP                                                                                                |
        |                                                                                            +-----------+
  Gi0/0 | 192.168.50.1/24                                                                             | PC-REMOTE |
        +---------------+                                                                             +-----------+
        |   R1-VPNGW    |  Gi0/1                                                       Gi0/1
        +---------------+  198.51.100.1/29 ---------------- ISP1 ---------------- 203.0.113.1/29 (ISP1)
                                                          (198.51.100.2/29)
```

- **R1-VPNGW** is the corporate VPN gateway — remote users connect to its public-facing address and, once authenticated, are treated as if they were on the internal network (within the limits of split tunneling).
- **PC-REMOTE** simulates a remote worker's laptop, connecting in over the "Internet" segment (ISP1) rather than being physically on the corporate LAN.
- **SRV1** (on the corporate LAN) represents an internal resource the remote user needs to reach — e.g., a file server or internal application.

---

## IP Addressing Table

| Device     | Interface | IP Address       | Subnet Mask       |
|------------|-----------|-------------------|---------------------|
| R1-VPNGW   | Gi0/0     | 192.168.50.1       | 255.255.255.0        |
| R1-VPNGW   | Gi0/1     | 198.51.100.1        | 255.255.255.248      |
| ISP1       | Gi0/0     | 198.51.100.2        | 255.255.255.248      |
| ISP1       | Gi0/1     | 203.0.113.1          | 255.255.255.248      |
| PC-REMOTE  | NIC       | 203.0.113.10          | 255.255.255.248      |
| SRV1       | NIC       | 192.168.50.20        | 255.255.255.0        |

**Remote-access address pool (handed out to connecting VPN clients):** 192.168.99.100 – 192.168.99.150

---

## Tasks

### Task 1 — Build the Topology
1. Place R1-VPNGW, ISP1, SW-CORP, SRV1, and PC-REMOTE as shown.
2. Apply the addressing above and configure basic routing (default routes toward ISP1 on both R1-VPNGW and PC-REMOTE's side) so the "Internet" segment is functional before configuring any VPN components.
3. Confirm R1-VPNGW can reach PC-REMOTE's address across the simulated Internet, and that no VPN yet exists between them (a direct ping to SRV1 from PC-REMOTE should currently fail, since it's not routable without the tunnel).

### Task 2 — Understand Why This Is Different from Site-to-Site
In the VPN ACLs lab, both endpoints were fixed corporate routers with known, static public addresses — the "interesting traffic" ACL simply matched two known subnets. Here, **PC-REMOTE's identity is the user**, not a fixed subnet — R1-VPNGW needs a way to authenticate an individual person (not just trust "anything from this specific peer IP"), and needs to hand that person a **usable internal address** dynamically, since a remote worker's home IP address has nothing to do with the corporate addressing scheme.

### Task 3 — Configure ISAKMP Phase 1 for Remote Access
```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
```
> Unlike the site-to-site lab, the peer's address isn't known in advance, so the pre-shared key in the next task is tied to a **group name** rather than a specific peer IP.

### Task 4 — Configure a Client Configuration Group
```
ip local pool REMOTE-POOL 192.168.99.100 192.168.99.150

crypto isakmp client configuration group REMOTE-USERS
 key RemoteAccessGroupKey123
 pool REMOTE-POOL
 acl SPLIT-TUNNEL-ACL
```
> The **group name and key** (`REMOTE-USERS` / `RemoteAccessGroupKey123`) are what the VPN client software is configured with — this is what identifies a connection as belonging to this remote-access policy, since (unlike site-to-site) there's no fixed peer address to match against instead.

### Task 5 — Configure the Split-Tunnel ACL
Define exactly which traffic should go through the tunnel — only corporate-bound traffic, **not** the remote user's general internet browsing:
```
ip access-list extended SPLIT-TUNNEL-ACL
 permit ip 192.168.50.0 0.0.0.255 any
```
> This ACL, referenced in Task 4, tells the client software (once it receives this policy from the gateway) to only route traffic destined for `192.168.50.0/24` through the tunnel — everything else continues out the remote user's own local internet connection directly. Without this, a full-tunnel configuration would route **all** of the remote user's traffic (including their general web browsing) through the corporate network, which is usually undesirable for both performance and corporate liability reasons.

### Task 6 — Configure User Authentication (Xauth) via Local AAA
Beyond the group key (which just identifies the policy to apply), individual users must authenticate with their own credentials:
```
aaa new-model
aaa authentication login REMOTE-XAUTH local
aaa authorization network REMOTE-AUTHOR local
username remoteuser1 secret RemoteUserPass456

crypto isakmp xauth timeout 30
```

### Task 7 — Configure IPsec Transform Set and Dynamic Crypto Map
Because the remote user's address isn't known in advance (unlike a site-to-site peer), a **dynamic** crypto map is used instead of a static one:
```
crypto ipsec transform-set REMOTE-TSET esp-aes 256 esp-sha256-hmac

crypto dynamic-map REMOTE-DYNMAP 10
 set transform-set REMOTE-TSET

crypto map REMOTE-VPN-MAP client authentication list REMOTE-XAUTH
crypto map REMOTE-VPN-MAP isakmp authorization list REMOTE-AUTHOR
crypto map REMOTE-VPN-MAP client configuration address respond
crypto map REMOTE-VPN-MAP 10 ipsec-isakmp dynamic REMOTE-DYNMAP
```

### Task 8 — Apply the Crypto Map to the Outside Interface
```
interface GigabitEthernet0/1
 crypto map REMOTE-VPN-MAP
```

### Task 9 — Connect from PC-REMOTE
Using a VPN client (software client configured with R1-VPNGW's public address, the group name/key from Task 4, and `remoteuser1`'s credentials from Task 6), initiate a connection. If your lab platform doesn't support a real software VPN client, treat Tasks 9–11 conceptually and focus verification on the router-side configuration itself (Task 13).

### Task 10 — Verify the Tunnel and Address Assignment
```
! On R1-VPNGW
show crypto isakmp sa
show crypto ipsec sa
show crypto session
```
Confirm a session appears for PC-REMOTE's public address, and that it has been assigned an address from the `REMOTE-POOL` range (192.168.99.100–150) — this assigned address is what SRV1 will see as the "source" for the remote user's corporate-bound traffic.

### Task 11 — Verify Split Tunneling Behavior
From PC-REMOTE (or by inspecting the client's routing table once connected):
- Traffic to `192.168.50.20` (SRV1) should route through the tunnel and succeed.
- Traffic to a general internet address (e.g., a public DNS server) should **not** route through the tunnel — it should continue to exit directly via PC-REMOTE's own local connection, exactly as if the VPN weren't active for that specific destination.

### Task 12 — Verify SRV1's View of the Connection
```
! On SRV1 (or via a packet capture if available)
```
Confirm traffic arriving from the remote user shows a source address from the `REMOTE-POOL` range, **not** PC-REMOTE's real public address (203.0.113.10) — the tunnel and address assignment together make the remote user appear to be "inside" the corporate addressing scheme for the duration of the session.

### Task 13 — Verify Router-Side Configuration
```
show crypto isakmp policy
show crypto isakmp client configuration group REMOTE-USERS
show ip local pool REMOTE-POOL
show crypto map
show run | section crypto
show run | section aaa
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 1 baseline | PC-REMOTE cannot reach SRV1 before the VPN is configured |
| Task 10 — tunnel establishment | `show crypto session` shows an active session for PC-REMOTE, address assigned from `REMOTE-POOL` |
| Task 11 — corporate traffic | Routes through the tunnel; reaches SRV1 successfully |
| Task 11 — general internet traffic | Does **not** route through the tunnel; confirms split tunneling is working, not full tunneling |
| Task 12 — SRV1's view | Sees the pool-assigned address, not PC-REMOTE's real public address |
| `show crypto isakmp client configuration group REMOTE-USERS` | Confirms pool and split-tunnel ACL are correctly associated with the group |

---

## Challenge (Optional)
- Convert `SPLIT-TUNNEL-ACL` to also include a second internal subnet (simulating a second corporate site reachable from this gateway) and confirm both are tunneled correctly while general internet traffic still bypasses the tunnel.
- Research and compare this lab's IPsec-based remote-access approach against a modern **SSL/TLS-based remote-access VPN** (e.g., Cisco AnyConnect over SSL rather than IPsec) — explain at least one practical advantage SSL-based remote access has in environments with restrictive outbound firewalls (hint: it typically uses TCP 443, a port almost never blocked, versus IPsec's use of UDP 500/4500 and ESP, which are more commonly blocked by strict client-side or public Wi-Fi firewalls).
- Add a second remote user account and confirm both users can connect simultaneously, each receiving a distinct address from the shared pool, and that revoking one user's local credentials (as practiced conceptually in the AAA labs) immediately prevents only that user from establishing new connections without affecting the other.