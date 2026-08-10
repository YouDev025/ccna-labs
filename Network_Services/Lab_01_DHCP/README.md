# Lab 01: DHCP

## Objective
Configure a Cisco IOS router as a **DHCP server**, correctly exclude reserved static addresses from the pool, and verify that clients receive the expected address, gateway, and DNS settings automatically.

---

## Topology

```
                          +-----------+
                          |   SRV1    |  Static IP (excluded from DHCP)
                          |192.1681.5|
                          +-----------+
                                |
                              SW1
                                |
                        Gi0/0   |  192.168.1.1/24
                       +---------------+
                       |      R1       |   (DHCP server)
                       +---------------+
                                |
                    +-----------+-----------+
                    |                       |
               +---------+             +---------+
               |  PC1    |             |  PC2    |
               |(DHCP)   |             |(DHCP)   |
               +---------+             +---------+
```

- **R1** acts as the DHCP server for the 192.168.1.0/24 LAN.
- **PC1** and **PC2** are configured for automatic (DHCP) addressing.
- **SRV1** has a fixed static IP that must never be handed out to a DHCP client, since it's already in permanent use.

---

## IP Addressing Plan

| Device | Interface | Address Type | IP Address       | Subnet Mask       |
|--------|-----------|---------------|-------------------|--------------------|
| R1     | Gi0/0     | Static         | 192.168.1.1        | 255.255.255.0      |
| SRV1   | NIC       | Static         | 192.168.1.5        | 255.255.255.0      |
| PC1    | NIC       | DHCP           | (assigned)         | (assigned)          |
| PC2    | NIC       | DHCP           | (assigned)         | (assigned)          |

**DHCP pool range:** 192.168.1.0/24
**Excluded range:** 192.168.1.1 – 192.168.1.10 (covers the router's own address and any statically-assigned devices like SRV1, with a little headroom for future static hosts)
**Default gateway to hand out:** 192.168.1.1
**DNS server to hand out:** 8.8.8.8 (or a lab-local DNS server address if one is available in your topology)
**Lease time:** 8 hours (for testing purposes; production networks often use 24 hours or longer)

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, SRV1, PC1, and PC2 as shown above.
2. Cable R1 Gi0/0 to SW1, and SW1 to SRV1, PC1, and PC2.
3. Configure R1's `Gi0/0` with the static address 192.168.1.1/24 and `no shutdown` the interface.
4. Configure SRV1 with its static address (192.168.1.5/24, gateway 192.168.1.1) — do **not** configure PC1 or PC2 with any address yet; they will get one from DHCP.

### Task 2 — Exclude Reserved Addresses
Before creating the pool, tell R1 which addresses it must **never** hand out — this must be done first, since DHCP pools are evaluated against exclusions at assignment time regardless of configuration order, but doing it first avoids accidentally leasing out a reserved address during testing:
```
ip dhcp excluded-address 192.168.1.1 192.168.1.10
```

### Task 3 — Configure the DHCP Pool
```
ip dhcp pool LAN-POOL
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.1
 dns-server 8.8.8.8
 lease 0 8 0
```
> `lease 0 8 0` means 0 days, 8 hours, 0 minutes. Use `lease infinite` only for special cases — it's generally discouraged since it prevents the server from ever reclaiming an address automatically.

### Task 4 — Enable DHCP Service (If Disabled)
On most IOS versions DHCP service is enabled by default, but confirm it explicitly:
```
service dhcp
```

### Task 5 — Request an Address on the Clients
On **PC1** and **PC2**, set their network adapters to obtain an IP address automatically (DHCP), then request/renew a lease:
- Windows: `ipconfig /release` then `ipconfig /renew`, or simply `ipconfig /all` if already leased.
- Packet Tracer PCs: use the **IP Configuration** tab and select **DHCP**, or run `ipconfig` from the command prompt.

### Task 6 — Verify on the Clients
```
ipconfig /all
```
Confirm each PC received:
- An IP address inside 192.168.1.11–192.168.1.254 (never inside the excluded range).
- Subnet mask 255.255.255.0.
- Default gateway 192.168.1.1.
- DNS server 8.8.8.8.

### Task 7 — Verify on R1
```
show ip dhcp binding
show ip dhcp pool LAN-POOL
show ip dhcp server statistics
show ip dhcp conflict
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show ip interface brief` on R1 | Gi0/0 `up/up`, 192.168.1.1 |
| PC1 `ipconfig /all` | Valid address in the pool range, correct mask/gateway/DNS |
| PC2 `ipconfig /all` | A **different** address than PC1, same mask/gateway/DNS |
| Neither PC1 nor PC2 receives an address in 192.168.1.1–192.168.1.10 | Confirms the exclusion is working |
| `show ip dhcp binding` on R1 | Lists both PC1 and PC2 with their leased addresses, lease expiration, and MAC/client-ID |
| `show ip dhcp pool LAN-POOL` | Shows total addresses, leased count, and current utilization |
| PC1 → PC2 ping | Succeeds (confirms both received correct, usable addressing) |
| PC1/PC2 → SRV1 ping | Succeeds (confirms the excluded static address and DHCP-assigned addresses coexist correctly on the same subnet) |

---

## Challenge (Optional)
- Add a second DHCP pool for a different VLAN/subnet on a subinterface or second physical interface, and confirm R1 serves the correct pool to each subnet based on the interface the request arrives on.
- Configure a **DHCP reservation** so a specific PC (identified by its MAC address) always receives the same IP address from the pool, even though it's technically using DHCP rather than static addressing:
  ```
  ip dhcp pool PC1-RESERVATION
   host 192.168.1.50 255.255.255.0
   client-identifier <PC1's client-ID/MAC, formatted per IOS requirements>
  ```
- Deliberately misconfigure the exclusion range (make it too small) and observe what happens if a DHCP client is offered an address that's actually already statically assigned to another device — use `show ip dhcp conflict` to see how the router detects and logs the resulting address conflict.