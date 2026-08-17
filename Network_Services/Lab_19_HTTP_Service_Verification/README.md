# Lab: HTTP Service Verification

## Objective
Configure and verify **HTTP-based services** in two related but distinct contexts: (1) securing the router's **own built-in HTTP/HTTPS management interface**, and (2) verifying reachability and correct behavior of a **hosted web service** on a server elsewhere in the network. You'll practice basic service-reachability testing with `telnet`/`curl` against specific TCP ports — a fundamental verification technique that applies far beyond just HTTP.

---

## Topology

```
                          +-----------+
                          |   SRV1    |   Hosts a web service on TCP 80/443
                          |192.168.95.20/24|
                          +-----------+
                                |
                              SW1
                                |
                        Gi0/1   |  192.168.95.1/24
           +---------------+
           |      R1       |   (also runs its own HTTP/HTTPS management service)
           +---------------+
     Gi0/0  |
             |
      +---------+                    +---------+
      |  PC1    |                    |  PC2    |
      |(trusted)|                    |(untrusted / test)|
      +---------+                    +---------+
```

- **R1** has its own built-in web-based management interface (IOS's `ip http server` / `ip http secure-server` feature) that must be secured, not left wide open.
- **SRV1** hosts a separate web application/service that clients need to reach — this is the "real" service being verified, distinct from R1's management interface.
- **PC1** represents a trusted admin workstation; **PC2** represents an untrusted/test host used to confirm access restrictions actually work.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1     | Gi0/1     | 192.168.95.1       | 255.255.255.0        |
| SRV1   | NIC       | 192.168.95.20       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |
| PC2    | NIC       | 192.168.10.11       | 255.255.255.0        |

---

## Part 1 — Securing the Router's Own HTTP Management Interface

### Task 1 — Build the Topology
1. Place R1, SW1, SRV1, PC1, and PC2 as shown, and apply the addressing above.
2. Confirm baseline reachability: PC1 and PC2 can both ping R1 and SRV1.

### Task 2 — Confirm the Insecure Default State
By default on many IOS images, `ip http server` may already be enabled with no access restriction. Check:
```
show ip http server status
```
If enabled, confirm (as a baseline test) that **both** PC1 and PC2 can currently reach R1's web management page:
```
! From PC2 (should currently succeed, which is the problem)
curl http://192.168.10.1
```

### Task 3 — Disable Plaintext HTTP, Enable HTTPS Only
Plaintext HTTP management sends credentials and session data unencrypted — this should never be used for real device management:
```
no ip http server
ip http secure-server
```
> Generating a self-signed certificate may be required the first time `ip http secure-server` is enabled — allow this to complete before continuing.

### Task 4 — Restrict Management Access by Source Address
Even with HTTPS enabled, don't allow every host on the LAN to attempt access — restrict it to known admin hosts:
```
access-list 60 permit host 192.168.10.10
ip http access-class 60
```

### Task 5 — Require Local Authentication
```
ip http authentication local
username netadmin privilege 15 secret AdminPass123
```

### Task 6 — Verify Router Management Access Restrictions
```
! From PC1 (should succeed)
curl -k https://192.168.10.1

! From PC2 (should fail/timeout — blocked by the ACL before authentication is even reached)
curl -k https://192.168.10.1
```
> The `-k` flag skips certificate validation, appropriate here since R1 is using a self-signed certificate in the lab.

### Task 7 — Verify Plaintext HTTP Is Fully Disabled
```
! From PC1
curl http://192.168.10.1
```
This should fail/refuse the connection entirely — confirming Task 3 successfully removed the insecure plaintext option, not just added HTTPS alongside it.

---

## Part 2 — Verifying a Hosted Web Service (SRV1)

### Task 8 — Confirm SRV1's Web Service Is Running
On SRV1 (or in your simulation platform's equivalent), confirm a basic web/HTTP service is active and listening on port 80 (and 443, if configured).

### Task 9 — Test Basic TCP Reachability Before Testing HTTP Itself
A key verification habit: confirm the **port is even reachable** before troubleshooting the application layer — this narrows down whether a failure is a network/ACL problem or an application problem:
```
! From PC1
telnet 192.168.95.20 80
```
A successful telnet connection (even with no meaningful banner) confirms TCP-level reachability to port 80 — press Ctrl+] then type `quit` (or your platform's equivalent) to close it cleanly.

### Task 10 — Test the HTTP Application Layer Itself
```
curl -v http://192.168.95.20
```
Confirm you receive an actual HTTP response (status line, headers, and body/content), not just a raw TCP connection — this confirms the web application itself is functioning correctly, not merely that the port is open.

### Task 11 — Test HTTPS, If Configured on SRV1
```
curl -vk https://192.168.95.20
```
Note any certificate warnings (expected with a self-signed lab certificate) and confirm the encrypted session still completes and returns content.

### Task 12 — Simulate and Diagnose a Service Failure
Temporarily block access to SRV1's web service from PC1 using an ACL on R1, to practice recognizing what a **network-layer** failure looks like versus what an application-layer failure (Task 8's service being down) would look like:
```
ip access-list extended BLOCK-HTTP-TEST
 deny tcp host 192.168.10.10 host 192.168.95.20 eq 80
 permit ip any any
interface GigabitEthernet0/0
 ip access-group BLOCK-HTTP-TEST in
```
Retest:
```
telnet 192.168.95.20 80
curl http://192.168.95.20
```
Both should now fail — importantly, note that `telnet` (basic TCP-level test) fails identically here to how it would if the web service itself were simply down. This is why Task 9's port-level test alone isn't enough to fully diagnose a problem — you also need to check ACLs/routing (network layer) versus the actual service status (application layer) as separate possibilities.

Remove the test ACL afterward:
```
interface GigabitEthernet0/0
 no ip access-group BLOCK-HTTP-TEST in
```

### Task 13 — Verify
```
show ip http server status
show ip http server secure status
show access-lists
show run | section ip http
show run | section access-list
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 baseline | Plaintext HTTP initially reachable from both PC1 and PC2 (before hardening) |
| Task 6 — PC1 HTTPS access | Succeeds, prompts for/accepts `netadmin` credentials |
| Task 6 — PC2 HTTPS access | Fails — blocked by `ip http access-class 60` before authentication is even attempted |
| Task 7 — plaintext HTTP after hardening | Fails entirely (connection refused) |
| Task 9 — telnet to SRV1:80 | Succeeds, confirming TCP-level reachability |
| Task 10 — curl to SRV1 | Returns a full HTTP response, confirming the application layer is working |
| Task 12 — with `BLOCK-HTTP-TEST` applied | Both telnet and curl fail identically, demonstrating that TCP-level failure alone doesn't distinguish a network problem from an application problem without further investigation |
| `show ip http server status` / `secure status` on R1 | Confirms `ip http server` disabled, `ip http secure-server` enabled, and the access-class applied |

---

## Challenge (Optional)
- Configure a **second admin host** and confirm it can be added to ACL 60 without disrupting PC1's existing access, practicing incremental ACL modification rather than rebuilding the whole list.
- Replace local authentication (Task 5) with **AAA-based authentication** (if covered in your course) for R1's HTTP management, and compare the configuration and verification steps against the simpler local-auth approach used here.
- Using `curl -v`, capture and review the actual HTTP response headers returned by SRV1, and identify at least one header that could reveal unnecessary information about the server (e.g., a `Server:` banner disclosing software/version) — discuss why minimizing such information disclosure is a common web-hardening practice, separate from network-level access control.