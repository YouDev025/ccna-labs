# Lab: SSH Key Management

## Objective
This lab isolates **one specific skill** the basic SSH lab only briefly touched: managing SSH **keys** across their full lifecycle. You'll verify **host key fingerprints** from the client side (and see what a changed fingerprint looks like — the exact signal that should make someone suspicious of a man-in-the-middle attack), configure **public-key user authentication** so a client can log in without a password at all, and practice **key rotation** for both host keys and user keys.

---

## Topology

```
                          +-----------+
                          |    R1     |
                          +-----------+
                        Gi0/0  |  192.168.10.1/24
                                |
                              SW1
                                |
                          +---------+
                          |  PC1    |   Generates its own SSH key pair for public-key auth
                          |192.168.10.10/24|
                          +---------+
```

- **R1** is the SSH server, already assumed to have the basic prerequisites from the SSH lab (hostname, domain name, an RSA host key, and `transport input ssh` on the VTY lines).
- **PC1** will generate its own client-side SSH key pair and use it to authenticate to R1 without a password.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |

---

## Part 1 — Host Key Fingerprint Verification

### Task 1 — Build the Topology and Confirm SSH Prerequisites
1. Place R1, SW1, and PC1, and apply the addressing above.
2. Ensure R1 has the basics from the SSH lab already configured:
   ```
   hostname R1
   ip domain-name lab.local
   crypto key generate rsa modulus 2048
   username netadmin privilege 15 secret StrongAdminPass123
   line vty 0 4
    transport input ssh
    login local
   ```

### Task 2 — View R1's Host Key Fingerprint
```
show crypto key mypubkey rsa
```
Note the key details shown (modulus size, key data). This is the **host key** — the credential R1 uses to prove its own identity to clients, distinct from any user-authentication credential.

### Task 3 — Connect from PC1 and Observe First-Connection Trust
```
! From PC1
ssh -l netadmin 192.168.10.1
```
On first connection, most SSH clients display the server's host key fingerprint and ask whether to trust it — this is **trust-on-first-use (TOFU)**: the client has no independent way to verify this is genuinely R1 the very first time, so it simply remembers the fingerprint for future comparison. Accept it, log in with the password, and confirm success.

### Task 4 — Verify the Client Remembered the Fingerprint
Disconnect and reconnect:
```
! From PC1
ssh -l netadmin 192.168.10.1
```
Confirm this second connection does **not** re-prompt for fingerprint trust — the client silently compared the presented fingerprint against what it saved in Task 3, found a match, and proceeded directly to the password prompt.

### Task 5 — Simulate a Changed Host Key (What a MITM Attack Would Look Like)
Regenerate R1's host key, simulating either a legitimate key rotation done carelessly, or an actual attacker presenting a different key while impersonating R1:
```
! On R1
crypto key zeroize rsa
crypto key generate rsa modulus 2048
```
```
! From PC1
ssh -l netadmin 192.168.10.1
```
Confirm the client now shows a **prominent warning** — typically something like "REMOTE HOST IDENTIFICATION HAS CHANGED" — rather than silently proceeding. This is the entire point of fingerprint verification: an unexpected change is flagged loudly specifically because it's the exact signature of a potential man-in-the-middle attack, not just an SSH client being fussy over nothing.

### Task 6 — Confirm This Was a Legitimate Change (Not Actually an Attack)
In this lab, you know the key change was your own doing in Task 5, so update your client's trusted fingerprint record to accept the new one (most clients require explicitly removing the old entry, e.g., from a `known_hosts` file, before they'll proceed). In your lab notes, explain what you would do **differently** if you had **not** just changed R1's key yourself and received this same warning unexpectedly — specifically, why proceeding anyway would defeat the entire purpose of fingerprint verification.

---

## Part 2 — Public-Key User Authentication

### Task 7 — Generate a Key Pair on PC1
```
! On PC1 (using OpenSSH client tools, or your platform's equivalent)
ssh-keygen -t rsa -b 2048 -f pc1_lab_key
```
This produces a private key (`pc1_lab_key`, kept secret on PC1) and a public key (`pc1_lab_key.pub`, safe to share and install on servers you want to log into).

### Task 8 — Install PC1's Public Key on R1
```
! On R1
ip ssh pubkey-chain
 username netadmin
  key-hash ssh-rsa <hash-of-the-public-key, or use the key-string method depending on IOS version>
```
> The exact syntax for importing a public key varies by IOS version — some platforms accept a direct `key-string` block with the raw public key data, others expect a pre-computed hash. Check `ip ssh pubkey-chain ?` and `username <name> key-hash ?` on your specific platform, and use whichever method it supports, ensuring the key ultimately matches PC1's `pc1_lab_key.pub` content exactly.

### Task 9 — Test Public-Key Login (No Password)
```
! From PC1
ssh -i pc1_lab_key -l netadmin 192.168.10.1
```
Confirm this succeeds **without ever prompting for a password** — R1 verified PC1's identity purely by confirming PC1 holds the private key matching the public key installed in Task 8.

### Task 10 — Confirm Password Login Still Works as a Fallback (or Doesn't, If You Choose to Restrict It)
```
! From PC1, this time without specifying the key
ssh -l netadmin 192.168.10.1
```
By default, both methods (password and public-key) typically remain available side by side unless you explicitly restrict the login method. Confirm password login still works here, then discuss in your lab notes when you might want to **disable** password authentication entirely for a given account once public-key auth is confirmed working — trading convenience/fallback for a stronger security posture (immune to password-guessing attacks entirely, since there's no password to guess).

---

## Part 3 — Key Rotation and Revocation

### Task 11 — Rotate R1's Host Key Deliberately and Document the Process
Unlike Task 5's disruptive/unannounced rotation, perform a **planned** host key rotation and write out, step by step, what you would communicate to users beforehand in a real environment (e.g., "expect a one-time fingerprint-changed warning on your next connection after this date/time — verify the new fingerprint against this out-of-band published value before accepting it"):
```
crypto key zeroize rsa
crypto key generate rsa modulus 2048 label R1-HOSTKEY-2026
```
> Using a `label` this time (rather than the default unnamed key from Task 1) makes it easier to reference this specific key going forward, and to potentially maintain multiple keys during a transition period if your platform supports it.

### Task 12 — Revoke PC1's Public Key
Simulate PC1 being decommissioned, or its private key being compromised — remove its trusted public key from R1 so it can no longer authenticate via key-based login:
```
! On R1
ip ssh pubkey-chain
 username netadmin
  no key-hash ssh-rsa <the hash entered in Task 8>
```
Confirm:
```
! From PC1
ssh -i pc1_lab_key -l netadmin 192.168.10.1
```
This should now **fail** (or fall back to prompting for a password, if password auth is still enabled for this account) — confirming the specific key was successfully revoked without needing to change the account's password or disable the account entirely.

### Task 13 — Verify
```
show crypto key mypubkey rsa
show ip ssh
show run | section ip ssh pubkey-chain
show run | include username
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — first connection | Client prompts for fingerprint trust (TOFU) |
| Task 4 — second connection | No re-prompt; fingerprint matched silently |
| Task 5 — host key changed | Client shows a prominent host-key-changed warning on next connection |
| Task 9 — public-key login | Succeeds with no password prompt |
| Task 10 — password login | Still succeeds independently, unless deliberately restricted |
| Task 12 — key revoked | PC1's key-based login fails/falls back after removal, without affecting the account's password-based access |

---

## Challenge (Optional)
- Configure a **second** user account with public-key authentication **only** (no password/secret configured at all, or password login explicitly disabled for that line/method list), and confirm that account can only ever log in via the matching private key — document the operational tradeoff this creates if that specific private key is ever lost.
- Research how SSH **certificate-based** authentication (distinct from simple public-key authentication) improves on the manual per-key installation process used in this lab, particularly for an environment with many devices and many users, where individually installing every user's public key on every device doesn't scale well.
- Write a short key-rotation policy (a paragraph or short list) for a hypothetical small network, specifying: how often host keys should be rotated, how users would be notified in advance, and what verification step (referencing Task 6) users should perform before accepting a new fingerprint rather than blindly trusting it.