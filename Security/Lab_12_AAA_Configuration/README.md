# Lab: AAA Configuration

## Objective
This lab goes beyond the introductory AAA lab's single-server-plus-local-fallback design into a more **production-realistic** AAA deployment: a **RADIUS server group with two redundant servers** (testing failover *between servers*, not just server-to-local), **command authorization** (controlling which privilege-level commands a user can run, not just whether they can log in), and **full accounting** of both EXEC sessions and individual commands.

---

## Topology

```
              +-----------+       +-----------+
              | RADIUS1   |       | RADIUS2   |   Two redundant RADIUS servers
              |(primary)  |       |(secondary)|
              |192.168.90.10/24|  |192.168.90.11/24|
              +-----------+       +-----------+
                       \             /
                            SW1
                              |
                        Gi0/1  |  192.168.90.1/24
                       +---------------+
                       |      R1       |
                       +---------------+
                 Gi0/0  |
                         |
                  +---------+
                  |  PC1    |   Used to test login, authorization, and accounting
                  |192.168.10.10/24|
                  +---------+
```

- **RADIUS1** and **RADIUS2** represent two independent authentication servers — a realistic design never depends on a single server.
- **R1** is configured to try both servers in order, with local credentials as a final tier-3 fallback only if **both** are unreachable.

---

## IP Addressing Table

| Device   | Interface | IP Address       | Subnet Mask       |
|----------|-----------|-------------------|---------------------|
| R1       | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1       | Gi0/1     | 192.168.90.1       | 255.255.255.0        |
| RADIUS1  | NIC       | 192.168.90.10       | 255.255.255.0        |
| RADIUS2  | NIC       | 192.168.90.11       | 255.255.255.0        |
| PC1      | NIC       | 192.168.10.10       | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology and Prerequisites
1. Place R1, SW1, RADIUS1, RADIUS2, and PC1 as shown, and apply the addressing above.
2. Confirm R1 can ping both RADIUS servers, and PC1 can ping R1.
3. Configure SSH prerequisites and a local fallback account (as in the SSH and introductory AAA labs):
   ```
   hostname R1
   ip domain-name lab.local
   crypto key generate rsa modulus 2048
   username localadmin privilege 15 secret LocalFallbackPass789
   aaa new-model
   ```

### Task 2 — Define Both RADIUS Servers
```
radius server RADIUS1
 address ipv4 192.168.90.10 auth-port 1812 acct-port 1813
 key RadiusSharedSecret111

radius server RADIUS2
 address ipv4 192.168.90.11 auth-port 1812 acct-port 1813
 key RadiusSharedSecret222
```
> Note each server can use a **different** shared key — they don't need to match each other, only their own respective server-side configuration.

### Task 3 — Group Both Servers Together with Defined Order
```
aaa group server radius RADIUS-GROUP
 server name RADIUS1
 server name RADIUS2
```
> The order servers are added to the group determines the order R1 will try them in — RADIUS1 first, RADIUS2 second, in this configuration.

### Task 4 — Create a Three-Tier Method List
```
aaa authentication login VTY-LOGIN group RADIUS-GROUP local
```
> This tries RADIUS1, then (if unreachable) RADIUS2, then (only if **both** are unreachable) falls back to the local database — three tiers total, a more resilient design than the two-tier setup in the introductory AAA lab.

### Task 5 — Apply and Verify Basic Login
```
line vty 0 4
 login authentication VTY-LOGIN
 transport input ssh
```
If your platform supports a real RADIUS server, configure a test account on RADIUS1 and confirm login succeeds via it:
```
! From PC1
ssh -l <radius-test-user> 192.168.10.1
```

### Task 6 — Test Failover from RADIUS1 to RADIUS2
Simulate RADIUS1 becoming unreachable:
```
! On R1 (simulate by blocking/removing reachability to RADIUS1 specifically)
access-list 99 deny host 192.168.90.10
access-list 99 permit any
! (Illustrative only — apply via an interface ACL or simply disconnect RADIUS1 in your lab platform, whichever is available)
```
Attempt login using a test account that exists on **RADIUS2** but not RADIUS1:
```
! From PC1
ssh -l <radius2-test-user> 192.168.10.1
```
Confirm this succeeds — demonstrating failover from the first server to the second, distinct from falling all the way back to local. If real RADIUS servers aren't available in your platform, treat this as a conceptual test and focus verification on Task 7 (full failure to local) instead, which is easier to simulate by simply making the whole 192.168.90.0/24 segment unreachable.

### Task 7 — Test Full Failure to Local (Both Servers Unreachable)
```
! On R1
interface GigabitEthernet0/1
 shutdown
```
```
! From PC1
ssh -l localadmin 192.168.10.1
```
Confirm this succeeds using the local account — only now, with **both** RADIUS servers unreachable, does the third tier activate. Restore the interface:
```
interface GigabitEthernet0/1
 no shutdown
```

### Task 8 — Configure Command Authorization
Authentication (Tasks 1–7) only controls **whether** someone can log in — it says nothing about what they're allowed to do once they're in. Add authorization to control which commands require server approval:
```
aaa authorization exec VTY-EXEC group RADIUS-GROUP local
aaa authorization commands 15 VTY-CMDS group RADIUS-GROUP local
```
```
line vty 0 4
 authorization exec VTY-EXEC
 authorization commands 15 VTY-CMDS
```
> `commands 15` specifically governs privilege-level-15 (full/enable-level) commands — a common design is to authorize only the highest privilege level's commands centrally, while lower-level (read-only/monitoring) commands are left unrestricted for efficiency, though your course/design may call for authorizing every level.

### Task 9 — Verify Authorization Behavior
If your RADIUS server can be configured to explicitly permit or deny specific commands for a test account, confirm:
- A permitted command executes normally.
- A denied command is rejected, even though the user is fully authenticated and logged in — demonstrating that authentication and authorization are genuinely separate checks. If your platform can't simulate server-side command authorization rules, note this limitation and verify only that the authorization method list is correctly applied (`show run | include authorization`).

### Task 10 — Configure Full Accounting
```
aaa accounting exec VTY-ACCT start-stop group RADIUS-GROUP
aaa accounting commands 15 VTY-CMD-ACCT start-stop group RADIUS-GROUP
```
```
line vty 0 4
 accounting exec VTY-ACCT
 accounting commands 15 VTY-CMD-ACCT
```
> `start-stop` sends a record when a session/command begins and another when it ends, giving the most complete audit trail — some designs use `stop-only` instead to reduce accounting traffic, at the cost of less detailed timing information.

### Task 11 — Verify Accounting
If your RADIUS server can display accounting records, log in, run a few commands, and confirm records appear showing the session and command history. If not available, verify the configuration is correctly applied:
```
show run | include accounting
```

### Task 12 — Final Verification
```
show aaa servers
show aaa method-lists all
show radius server-group RADIUS-GROUP
show run | section aaa
show run | section radius
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 5 — RADIUS1 login | Succeeds via the primary server (if available) |
| Task 6 — RADIUS1 unreachable | Falls to RADIUS2 successfully, without dropping all the way to local |
| Task 7 — both servers unreachable | Falls to local `localadmin` account, confirming the third tier |
| Task 8/9 — command authorization | Method lists applied; permitted/denied commands behave correctly if server-side rules are testable |
| Task 10/11 — accounting | Method lists applied; records visible if server-side reporting is available |
| `show aaa servers` | Shows both RADIUS1 and RADIUS2 with connection statistics |
| `show radius server-group RADIUS-GROUP` | Confirms both servers are members, in the correct order |

---

## Challenge (Optional)
- Reverse the order servers are added to `RADIUS-GROUP` and confirm (via `show aaa servers` statistics, or observed behavior) that R1 now tries RADIUS2 first — demonstrating that group order, not server creation order, determines preference.
- Research and compare **RADIUS** (used in this lab) against **TACACS+** (used in the introductory AAA lab) at a conceptual level — specifically, that RADIUS combines authentication and authorization into a single response while TACACS+ separates them into distinct exchanges, and that TACACS+ encrypts the entire packet body while RADIUS traditionally only encrypts the password field. Explain why this makes TACACS+ generally preferred for device administration (like this lab) while RADIUS remains dominant for network access (802.1X, VPN, Wi-Fi).
- Design a method list strategy (in writing) for a network with junior helpdesk staff (who should only run monitoring/`show` commands) and senior network engineers (full access), using privilege levels and command authorization together — you don't need to implement it if your RADIUS platform can't simulate the server-side rules, but specify exactly what `aaa authorization commands` levels and server-side permissions you would configure.