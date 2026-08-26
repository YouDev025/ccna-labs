# Lab 01: SSH

## Objective
Replace insecure Telnet access with **SSH** on a Cisco IOS router: configure the prerequisites (hostname, domain name, RSA keys), enable SSH-only VTY access, require local authentication, and verify both that SSH works and that the older, insecure Telnet option no longer does.

---

## Topology

```
                          +-----------+
                          |    R1     |
                          +-----------+
                        Gi0/0 | 192.168.10.1/24
                                |
                              SW1
                    +----------+----------+
                    |                     |
              +-----------+         +-----------+
              |   PC1     |         |   PC2     |
              |(admin,    |         |(untrusted /|
              | trusted)  |         | test host) |
              +-----------+         +-----------+
```

- **R1** is the device being secured.
- **PC1** represents a trusted admin workstation, used to confirm SSH access works correctly.
- **PC2** represents an untrusted host, used later to confirm access restrictions (if added) behave as expected.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |
| PC2    | NIC       | 192.168.10.11       | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, PC1, and PC2 as shown, and apply the addressing above.
2. Confirm PC1 and PC2 can both ping R1.

### Task 2 — Confirm the Insecure Starting State
By default, VTY lines often allow Telnet with no encryption. Check:
```
show run | section line vty
```
If Telnet access works with only a basic password (or none at all), confirm this insecure baseline:
```
! From PC1
telnet 192.168.10.1
```
Note that any credentials or command output exchanged here would be sent completely unencrypted — this is exactly what SSH is replacing.

### Task 3 — Set Hostname and Domain Name
SSH requires both a hostname and a domain name before RSA keys can be generated, since the key is generated using the device's fully-qualified domain name:
```
hostname R1
ip domain-name lab.local
```

### Task 4 — Generate RSA Keys
```
crypto key generate rsa
```
When prompted for key size, choose **2048 bits** — this is a reasonable modern minimum; older/smaller sizes (e.g., 512 or 768) are considered weak by today's standards and some newer IOS versions may reject them outright for SSH use.

### Task 5 — Configure Local User Authentication
SSH needs something to authenticate against — set up a local username/password (or, in a real deployment, tie this into AAA/RADIUS/TACACS+, covered in a separate lab if your course includes one):
```
username netadmin privilege 15 secret StrongAdminPass123
```
> Use `secret` (which hashes the password) rather than `password` (which historically stored it far more weakly, depending on additional configuration) — this distinction matters for local credential security.

### Task 6 — Configure VTY Lines for SSH Only
```
line vty 0 4
 transport input ssh
 login local
```
> `transport input ssh` removes Telnet as an option entirely — only SSH connections will be accepted on these lines going forward. `login local` tells the router to authenticate against the local username database configured in Task 5, rather than a simple line password.

### Task 7 — Set the SSH Version and Timeout
```
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
```
> SSH version 1 has known security weaknesses and should never be used if version 2 is available — explicitly forcing version 2 avoids accidentally falling back to it.

### Task 8 — Verify SSH Is Enabled and Correctly Configured
```
show ip ssh
```
Confirm it reports SSH enabled, version 2, and the timeout/retry values you set.

### Task 9 — Test SSH Access from PC1
```
! From PC1
ssh -l netadmin 192.168.10.1
```
Enter the password configured in Task 5 when prompted. Confirm you reach the router's CLI successfully.

### Task 10 — Confirm Telnet No Longer Works
```
! From PC1
telnet 192.168.10.1
```
This should now be refused/fail immediately, confirming Task 6's `transport input ssh` successfully removed Telnet as an option.

### Task 11 — Verify Active Sessions
While still connected via SSH from Task 9, check on R1 (via console, or a second SSH session if your platform supports concurrent access):
```
show ssh
show users
```
Confirm the active SSH session from PC1 is listed, including the encryption details being used.

### Task 12 — Final Verification
```
show ip ssh
show run | section line vty
show run | include username
show run | include hostname
show run | include ip domain-name
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 baseline | Telnet works with weak/no real authentication before hardening |
| Task 4 — RSA key generation | Succeeds only after hostname and domain name are set; 2048-bit key generated |
| Task 8 — `show ip ssh` | Reports SSH enabled, version 2 |
| Task 9 — PC1 SSH | Succeeds with the configured username/password |
| Task 10 — Telnet after hardening | Fails/refused |
| Task 11 — `show ssh` | Lists the active session from PC1 |

---

## Challenge (Optional)
- Restrict VTY access to only PC1's address using an access-class ACL (`access-list 10 permit host 192.168.10.10` + `line vty 0 4` / `access-class 10 in`), and confirm PC2 can no longer even attempt an SSH connection, while PC1 remains unaffected — connecting this lab to the ACL Placement/Standard ACLs labs.
- Research and briefly document the difference between `login local` (used in this lab) and a full **AAA** configuration (`aaa new-model` plus RADIUS/TACACS+) for authenticating SSH sessions, and explain a scenario where centralized AAA would be clearly preferable to local usernames (e.g., many devices, frequent staff turnover, centralized audit logging requirements).
- Delete the RSA keys (`crypto key zeroize rsa`) and observe what happens to existing and new SSH connection attempts — then regenerate the keys and confirm SSH access is restored, documenting the brief service interruption this causes as a reason to plan key regeneration carefully in a production environment.