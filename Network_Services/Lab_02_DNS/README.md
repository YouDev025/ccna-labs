# Lab 02: DNS

## Objective
Configure and verify **DNS name resolution** in a small network: set up a router as a simple DNS server (using IOS's built-in DNS server feature) with static host records, configure a client to use it, configure a second router as a DNS *client* so its own CLI commands can resolve names, and confirm resolution works end-to-end with `ping`, `nslookup`, and `show` commands.

---

## Topology

```
              +-----------+
              |   DNS1    |   DNS server (authoritative for lab.local)
              |192.168.60.2/24|
              +-----------+
                    |
              +-----------+
              |   SRV1    |   192.168.60.20/24  (srv1.lab.local)
              +-----------+
                    |
                  SW-DMZ
                    |
            Gi0/1   |  192.168.60.1/24
           +---------------+
           |      R1       |   (DNS client — resolves names for its own CLI)
           +---------------+
            Gi0/0  |  192.168.50.1/24
                    |
                  SW-LAN
                    |
              +-----------+
              |   PC1     |   DNS server = 192.168.60.2 (via R1)
              |192.168.50.10/24|
              +-----------+
```

- **DNS1** is configured as a simple DNS server using Cisco IOS's `ip dns server` feature, authoritative for the fictitious domain `lab.local`.
- **SRV1** and **R1** each have a hostname registered in DNS1.
- **R1** is also configured as a DNS *client* itself (`ip domain-lookup` + `ip name-server`), so its own `ping`/`traceroute` commands can resolve names — a separate concept from serving DNS to others.
- **PC1** is a regular client pointed at DNS1 for its own name resolution.

---

## IP Addressing and Naming Table

| Device | Interface | IP Address       | Subnet Mask       | Hostname (DNS record) |
|--------|-----------|-------------------|--------------------|--------------------------|
| DNS1   | NIC       | 192.168.60.2       | 255.255.255.0      | dns1.lab.local            |
| SRV1   | NIC       | 192.168.60.20       | 255.255.255.0      | srv1.lab.local            |
| R1     | Gi0/0     | 192.168.50.1       | 255.255.255.0      | —                         |
| R1     | Gi0/1     | 192.168.60.1       | 255.255.255.0      | r1.lab.local              |
| PC1    | NIC       | 192.168.50.10       | 255.255.255.0      | pc1.lab.local (optional)  |

**Domain name:** `lab.local`

---

## Tasks

### Task 1 — Build the Topology
1. Place DNS1, SRV1, R1, PC1, and the two switches as shown.
2. Apply the addressing above. Add a route or default gateway on DNS1/SRV1 pointing to R1's Gi0/1 (192.168.60.1) if they need to reach the other subnet; likewise, ensure R1 routes between both subnets (they're both directly connected here, so no additional static routes are required beyond `no shutdown` on both interfaces).
3. Verify baseline reachability by IP address only (no names yet): PC1 can ping R1, SRV1, and DNS1 by their numeric IPs.

### Task 2 — Configure DNS1 as a Simple DNS Server
Enable the DNS server feature and add static host records:
```
ip dns server
ip host srv1.lab.local 192.168.60.20
ip host r1.lab.local 192.168.60.1
ip host dns1.lab.local 192.168.60.2
```
> On a real router acting as a lightweight DNS server, `ip host <name> <address>` entries are what the server answers queries with. This is intentionally simple — enterprise networks would normally use a dedicated DNS server product (Windows DNS, BIND, etc.), but IOS's built-in feature is enough to demonstrate the resolution process end-to-end in a lab.

### Task 3 — Point PC1 at DNS1
On PC1, set the DNS server address to `192.168.60.2` (either via static IP configuration or, if you're also running the DHCP lab, by adding `dns-server 192.168.60.2` to that pool). Confirm with:
```
ipconfig /all
```

### Task 4 — Test Resolution from PC1
```
nslookup srv1.lab.local
ping srv1.lab.local
```
Both should resolve to `192.168.60.20` and the ping should succeed.

### Task 5 — Configure R1 as a DNS *Client*
This is a separate, commonly-confused concept from Task 2: R1 being a DNS server for others does not automatically mean R1 itself uses DNS for its own commands. Enable that separately:
```
ip domain-lookup
ip domain-name lab.local
ip name-server 192.168.60.2
```

### Task 6 — Test Resolution from R1's Own CLI
```
ping srv1.lab.local
```
R1 should now resolve the name via DNS1 and successfully ping SRV1 — confirming R1 is correctly using DNS1 as its resolver for its own operations, not just serving records to others.

### Task 7 — Compare Against a Local Static Host Table (No DNS Server Involved)
As a contrast, configure a **local, static** name-to-address mapping directly on R1 — this resolves a name without querying DNS1 at all:
```
ip host branch-test.lab.local 192.168.60.99
```
```
ping branch-test.lab.local
```
This will resolve instantly (from R1's local table) but the ping itself will fail (no device is using that IP) — the point of this task is to demonstrate that `ip host` entries are checked **before** an external DNS query is ever sent, which is useful to know when troubleshooting "wrong" or unexpected resolution results in production.

### Task 8 — Verify
On R1:
```
show hosts
show running-config | include ip name-server
show running-config | include ip domain
```
On DNS1:
```
show running-config | include ip host
show running-config | include ip dns server
```
On PC1:
```
ipconfig /all
nslookup srv1.lab.local
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Baseline (IP-only) connectivity | PC1, R1, SRV1, DNS1 all reachable by IP before any DNS config |
| PC1 `nslookup srv1.lab.local` | Resolves to 192.168.60.20 |
| PC1 `ping srv1.lab.local` | Succeeds |
| R1 `ping srv1.lab.local` (after Task 5) | Succeeds — confirms R1 itself is using DNS1 as a resolver |
| R1 `show hosts` | Lists both dynamically-learned entries (from DNS1) and the static `branch-test.lab.local` entry from Task 7, distinguishing cache source |
| `ping branch-test.lab.local` on R1 | Name resolves instantly but ping fails (no live host at that address) — confirms local host table takes precedence over DNS |
| DNS1 `show running-config \| include ip host` | Shows all three static host records configured in Task 2 |

---

## Challenge (Optional)
- Add a **CNAME-style alias** using a second `ip host` entry pointing a different name to the same IP as `srv1.lab.local` (IOS doesn't support true CNAME records, so this demonstrates the practical limitation vs. a real DNS server), and discuss in your lab notes when you'd need a full DNS product instead of IOS's built-in feature.
- Configure a **second DNS server** address on PC1 as a backup (`ip name-server 192.168.60.2 8.8.8.8` equivalent on R1, or a secondary DNS field on PC1), then disable DNS1 entirely and confirm PC1/R1 fail over to the secondary resolver for external names (this will only work for names DNS1 doesn't own, since 8.8.8.8 won't know about `lab.local`).
- Clear R1's DNS cache (`clear host *`) and use `debug ip domain` while pinging a name, to watch the resolution process happen in real time — remember to `undebug all` afterward.