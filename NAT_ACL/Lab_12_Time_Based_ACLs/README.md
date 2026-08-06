# Lab: Time-Based ACLs

## Objective
Practice configuring and verifying **time-based extended ACLs** on a Cisco IOS router. Time-based ACLs let you permit or deny traffic only during specific time windows (e.g., business hours), using a `time-range` object applied inside an extended ACL entry.

---

## Topology

```
                         Internet / Server Farm
                               |
                        Gi0/1  |  203.0.113.1/29
                       +---------------+
                       |      R1       |
                       +---------------+
                        Gi0/0  |  192.168.20.1/24
                               |
                             SW1
                       +------+------+
                       |             |
                  +---------+   +---------+
                  |  PC1    |   |  PC2    |
                  |.10      |   |.11      |
                  +---------+   +---------+

              +-----------+
              |  SRV1     |   (simulates an outside web/game server)
              |203.0.113.10/29|
              +-----------+
```

- **R1** is the access router enforcing the time-based policy on `Gi0/0` (inside) or `Gi0/1` (outside), depending on direction chosen.
- **PC1** and **PC2** represent employee workstations on the inside LAN.
- **SRV1** simulates an external site (e.g., a streaming/social-media/game server) that should only be reachable during non-business hours.

---

## IP Addressing Table

| Device | Interface | IP Address     | Subnet Mask       | Default Gateway |
|--------|-----------|-----------------|--------------------|-----------------|
| R1     | Gi0/0     | 192.168.20.1     | 255.255.255.0      | —               |
| R1     | Gi0/1     | 203.0.113.1       | 255.255.255.248    | —               |
| SRV1   | NIC       | 203.0.113.10       | 255.255.255.248    | 203.0.113.1     |
| PC1    | NIC       | 192.168.20.10       | 255.255.255.0      | 192.168.20.1    |
| PC2    | NIC       | 192.168.20.11       | 255.255.255.0      | 192.168.20.1    |

---

## Scenario
Company policy states that access to `SRV1` (representing a recreational/streaming site) from the inside LAN is only allowed **outside of business hours**:
- **Blocked:** Monday–Friday, 08:00–18:00
- **Allowed:** all other times (evenings, nights, weekends)

You will implement this using a `time-range` object and an extended ACL applied inbound on R1's LAN interface.

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, PC1, PC2, and SRV1 as shown above.
2. Cable R1 Gi0/0 to SW1, and SW1 to PC1/PC2. Cable R1 Gi0/1 to SRV1 (directly or via a second switch).
3. Apply the addressing from the table above to all devices.
4. Verify baseline connectivity: PC1 and PC2 can ping R1 and SRV1 with **no ACL applied yet**.

### Task 2 — Set the Router's Clock
Time-based ACLs depend on the router's clock (or NTP). Set it manually for testing:
```
clock set 09:00:00 15 August 2026
```
> Adjust the date/time so you can easily test both inside and outside the "business hours" window during the lab.

### Task 3 — Create the Time Range
Define a reusable time-range object representing business hours:
```
time-range BUSINESS-HOURS
 periodic weekdays 08:00 to 18:00
```
> `weekdays` covers Monday–Friday automatically. Use `periodic daily`, or day names like `monday tuesday`, if you need different coverage.

### Task 4 — Build the Time-Based Extended ACL
Create an extended ACL that **denies** traffic to SRV1 only during business hours, and **permits** everything else (including SRV1 traffic outside that window, via the implicit/explicit permit):
```
ip access-list extended BLOCK-RECREATIONAL
 deny ip 192.168.20.0 0.0.0.255 host 203.0.113.10 time-range BUSINESS-HOURS
 permit ip any any
```

### Task 5 — Apply the ACL
Apply it inbound on the LAN-facing interface, so it's evaluated as soon as traffic enters R1 from PC1/PC2:
```
interface GigabitEthernet0/0
 ip access-group BLOCK-RECREATIONAL in
```

### Task 6 — Test During Business Hours
With the clock set inside 08:00–18:00 on a weekday:
- From PC1/PC2, ping or attempt to reach SRV1 (203.0.113.10) → **should fail**.
- General traffic to other destinations (e.g., R1 itself, or another test host) → **should still succeed**.

### Task 7 — Test Outside Business Hours
Change the router's clock to a time outside the window (e.g., 19:00, or a Saturday):
```
clock set 19:00:00 15 August 2026
```
- From PC1/PC2, ping SRV1 again → **should now succeed**.

### Task 8 — Verify
On R1:
```
show clock
show time-range BUSINESS-HOURS
show access-lists BLOCK-RECREATIONAL
show ip interface GigabitEthernet0/0 | include access list
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| `show clock` | Reflects the time you set for each test phase |
| `show time-range BUSINESS-HOURS` | Shows the range as **active** during business hours, **inactive** otherwise |
| `show access-lists BLOCK-RECREATIONAL` | Displays match counters incrementing on the `deny` line only while the range is active |
| Ping PC1/PC2 → SRV1 during business hours | Fails (denied by time-based ACL) |
| Ping PC1/PC2 → SRV1 outside business hours | Succeeds |
| Ping PC1/PC2 → R1 (any time) | Succeeds (not matched by the deny statement) |
| `show ip interface Gi0/0` | Confirms `BLOCK-RECREATIONAL` is applied **inbound** |

---

## Challenge (Optional)
- Modify the `time-range` to also block access during a lunch-hour exception (e.g., allow 12:00–13:00 even on weekdays) using an `absolute` or a second `periodic` statement combined with ACL ordering.
- Add a second time-range that permits IT staff (a specific host, e.g., PC2) unrestricted access at all times by placing a `permit` entry for that host **above** the time-based deny.
- Replace `periodic weekdays` with an `absolute start ... end ...` range to model a **temporary** policy (e.g., blocking access only during a one-week event) and verify it expires correctly.