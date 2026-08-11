# Lab: DHCPv6

## Objective
Configure and verify **IPv6 address assignment** using all three models IPv6 supports, so you can clearly see how they differ:
1. **Stateless SLAAC only** — clients self-generate addresses from Router Advertisements, no DHCPv6 involved at all.
2. **Stateless DHCPv6** — clients still self-generate their address via SLAAC, but also query DHCPv6 for *extra* information (like DNS servers) that RAs alone don't provide.
3. **Stateful DHCPv6** — a DHCPv6 server assigns the full address, the same model as IPv4 DHCP.

The key skill this lab builds is reading and controlling the **M and O flags** in Router Advertisements, since those flags are what tell a client which of the three models to use — a detail that trips people up because DHCPv6 behavior looks "broken" if the RA flags don't match what you configured on the DHCPv6 pool.

---

## Topology

```
              +-----------+
              |    R1     |   Router + DHCPv6 server
              +-----------+
                    |
              Gi0/0  |  2001:DB8:80::1/64
                    |
                  SW1
          +---------+---------+
          |                   |
     +---------+         +---------+
     |  PC1    |         |  PC2    |
     +---------+         +---------+
```

- **R1** is both the IPv6 default gateway (sending Router Advertisements) and the DHCPv6 server.
- **PC1** and **PC2** are used at different points in the lab to test each of the three addressing models, by changing R1's RA flags and DHCPv6 configuration between tasks.

---

## Addressing Table

| Device | Interface | Address                    | Notes |
|--------|-----------|-------------------------------|-------|
| R1     | Gi0/0     | 2001:DB8:80::1/64              | Also LAN default gateway |
| PC1    | NIC       | (varies per task)               | IPv6-enabled, no static address |
| PC2    | NIC       | (varies per task)               | IPv6-enabled, no static address |

**DHCPv6 pool range for stateful mode:** 2001:DB8:80::100 – 2001:DB8:80::1FF
**DNS server to hand out (all modes):** 2001:4860:4860::8888 (Google Public DNS IPv6, or a lab-local DNS server if available)
**Domain name:** lab.local

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, PC1, and PC2 as shown.
2. Enable IPv6 routing and address R1's LAN interface:
   ```
   ipv6 unicast-routing
   interface GigabitEthernet0/0
    ipv6 address 2001:DB8:80::1/64
    no shutdown
   ```
3. Set PC1 and PC2 to obtain IPv6 addressing automatically (no static configuration) — this stays constant for the whole lab; only R1's configuration changes between phases.

---

### Phase 1 — Stateless SLAAC Only (No DHCPv6)

#### Task 2 — Confirm Default Behavior
By default, once `ipv6 unicast-routing` and an interface address are configured, R1 already sends Router Advertisements with the **M (Managed)** and **O (Other config)** flags both set to 0 — meaning "use SLAAC, don't bother with DHCPv6 at all."
```
show ipv6 interface GigabitEthernet0/0 | include flag
```

#### Task 3 — Verify SLAAC-Only Addressing
On **PC1**, check the assigned address:
```
ipconfig /all      (Windows)
```
Confirm PC1 has self-generated a global address in the `2001:DB8:80::/64` prefix (typically using EUI-64 or a randomized interface identifier, depending on the OS), and that **no DNS server** was learned — this is the expected limitation of pure SLAAC, which is exactly what Phase 2 fixes.

---

### Phase 2 — Stateless DHCPv6 (SLAAC for Address, DHCPv6 for Extras Only)

#### Task 4 — Configure a Stateless DHCPv6 Pool
```
ipv6 dhcp pool STATELESS-POOL
 dns-server 2001:4860:4860::8888
 domain-name lab.local
```

#### Task 5 — Bind the Pool and Set the O Flag
```
interface GigabitEthernet0/0
 ipv6 dhcp server STATELESS-POOL
 ipv6 nd other-config-flag
```
> `ipv6 nd other-config-flag` sets the **O flag** in Router Advertisements, telling clients "your address is fine from SLAAC, but come ask DHCPv6 for additional settings like DNS." The **M flag** is deliberately left at 0 here — do not set `ipv6 nd managed-config-flag` in this phase.

#### Task 6 — Verify Stateless DHCPv6
On **PC2** (or reset/renew PC1's IPv6 configuration):
```
ipconfig /all
```
Confirm:
- The IPv6 address is still self-generated via SLAAC (same prefix, still no DHCPv6-assigned address).
- A **DNS server** is now present, learned via DHCPv6 even though the address itself wasn't.

---

### Phase 3 — Stateful DHCPv6 (Full Address Assignment)

#### Task 7 — Configure a Stateful DHCPv6 Pool
```
ipv6 dhcp pool STATEFUL-POOL
 address prefix 2001:DB8:80::/64 lifetime 3600 1800
 dns-server 2001:4860:4860::8888
 domain-name lab.local
```
> `lifetime 3600 1800` sets the valid and preferred lifetimes for assigned addresses (in seconds) — analogous to a DHCP lease time.

#### Task 8 — Bind the New Pool and Set the M Flag
```
interface GigabitEthernet0/0
 no ipv6 dhcp server STATELESS-POOL
 no ipv6 nd other-config-flag
 ipv6 dhcp server STATEFUL-POOL
 ipv6 nd managed-config-flag
```
> Setting the **M flag** tells clients "don't self-generate an address — request one from DHCPv6 instead." Cisco IOS clients (and most modern OSes) will also implicitly treat M=1 as including "and get other config too," so `other-config-flag` isn't required in addition to `managed-config-flag`.

#### Task 9 — Verify Stateful DHCPv6
Renew IPv6 addressing on PC1 and PC2:
```
ipconfig /release6
ipconfig /renew6
```
Confirm both PCs now receive an address specifically from within the configured range (2001:DB8:80::100 – ::1FF), rather than a self-generated SLAAC address, along with the DNS server and domain name.

#### Task 10 — Verify on R1
```
show ipv6 dhcp pool
show ipv6 dhcp binding
show ipv6 dhcp interface GigabitEthernet0/0
show ipv6 interface GigabitEthernet0/0 | include flag
show ipv6 nd interface GigabitEthernet0/0
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Phase 1 — PC1 address | Self-generated SLAAC address; no DNS server learned |
| Phase 1 — `show ipv6 interface ... \| include flag` | Both M and O flags show as **not set** |
| Phase 2 — PC2 address | Still self-generated SLAAC address (unchanged behavior) |
| Phase 2 — PC2 DNS server | Now present, learned via stateless DHCPv6 |
| Phase 2 — RA flags | O flag set, M flag still not set |
| Phase 3 — PC1/PC2 address | Assigned from the DHCPv6 pool range (::100–::1FF), not self-generated |
| Phase 3 — RA flags | M flag set |
| `show ipv6 dhcp binding` on R1 (Phase 3 only) | Lists PC1 and PC2 with their assigned addresses and lease/lifetime details |
| `show ipv6 dhcp pool` | Shows correct pool configuration and active lease counts for whichever pool is currently bound |

---

## Challenge (Optional)
- Configure **DHCPv6 Prefix Delegation (PD)** on R1 so it could hand out a `/64` (or smaller) prefix to a downstream router rather than individual host addresses — describe in your lab notes how this differs from the host-level stateful assignment done in Phase 3, and where PD is typically used (e.g., an ISP delegating a prefix to a home/branch router).
- Add an **excluded range** within the stateful pool (e.g., reserve ::100–::10F for statically-addressed servers) and confirm the DHCPv6 server never assigns from that range — mirroring the IPv4 DHCP exclusion concept from the DHCP lab.
- Deliberately mismatch the M/O flags against the bound pool type (e.g., set the M flag while only a stateless pool is bound) and document the resulting inconsistent or broken client behavior — this is a common real-world misconfiguration and worth experiencing deliberately in a lab rather than for the first time in production.