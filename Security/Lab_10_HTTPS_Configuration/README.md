# Lab: HTTPS Configuration

## Objective
This lab isolates **one specific skill** the HTTP Service Verification lab only briefly touched: real **HTTPS/TLS configuration** on a Cisco IOS device — creating a proper **PKI trustpoint** for certificate management (rather than just accepting the default self-signed certificate), inspecting certificate details, and **enforcing a minimum TLS version** so weak/legacy protocol versions are rejected outright, not just discouraged.

---

## Topology

```
                          +-----------+
                          |    R1     |
                          +---------------+
                        Gi0/0  |  192.168.10.1/24
                                |
                              SW1
                                |
                          +---------+
                          |  PC1    |   Used for HTTPS testing with curl/openssl
                          |192.168.10.10/24|
                          +---------+
```

- **R1** hosts its own HTTPS management interface, which you'll configure with a proper trustpoint-managed certificate and a hardened minimum TLS version.
- **PC1** is used to inspect the certificate and test protocol negotiation from the client side.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| PC1    | NIC       | 192.168.10.10       | 255.255.255.0        |

---

## Tasks

### Task 1 — Build the Topology
1. Place R1, SW1, and PC1 as shown, and apply the addressing above.
2. Confirm PC1 can ping R1.

### Task 2 — Prerequisites
```
hostname R1
ip domain-name lab.local
crypto key generate rsa label R1-HTTPS-KEY modulus 2048
```
> Giving the key an explicit `label` (rather than using the default, unnamed key) makes it easier to reference specifically when creating the trustpoint in the next task, and avoids ambiguity if multiple keys ever exist on the device.

### Task 3 — Confirm the Default Behavior (Auto-Generated Self-Signed Certificate)
```
ip http secure-server
```
On first enablement, IOS automatically generates a basic self-signed certificate if no trustpoint is explicitly configured. Check it:
```
show crypto pki certificates
```
Note the certificate's subject, issuer (should be itself, confirming it's self-signed), and validity period — this is the "default" state most devices are left in, which this lab improves on.

### Task 4 — Create a Dedicated PKI Trustpoint
Rather than relying on the automatically-generated certificate, define an explicit trustpoint with controlled parameters:
```
crypto pki trustpoint R1-HTTPS-TRUSTPOINT
 enrollment selfsigned
 subject-name CN=r1.lab.local,O=LabOrg,C=US
 rsakeypair R1-HTTPS-KEY
 revocation-check none
```
> `enrollment selfsigned` is used here since this lab doesn't have a real external Certificate Authority available — in a production network, you would instead use `enrollment terminal` (manual CSR/certificate exchange) or `enrollment url <CA-URL>` (automatic enrollment, e.g., via SCEP) to get a certificate signed by a real, trusted CA instead of self-signing. `revocation-check none` disables CRL/OCSP checking, which is appropriate only because there's no real CA infrastructure in this lab to check against.

### Task 5 — Generate the Self-Signed Certificate from the Trustpoint
```
crypto pki enroll R1-HTTPS-TRUSTPOINT
```
Confirm the prompts (this generates and self-signs a certificate using the trustpoint's defined subject name and the specified key pair).

### Task 6 — Bind the New Certificate to the HTTPS Server
```
ip http secure-trustpoint R1-HTTPS-TRUSTPOINT
```
> This tells `ip http secure-server` to use the certificate issued from your explicitly-configured trustpoint, instead of the generic auto-generated one from Task 3.

### Task 7 — Verify the New Certificate Is in Use
```
show crypto pki certificates
```
Confirm the subject name now matches what you configured in Task 4 (`CN=r1.lab.local,O=LabOrg,C=US`), not a generic default.
```
! From PC1
curl -vk https://192.168.10.1 2>&1 | grep -i subject
```
Confirm the subject line returned to the client matches your configured certificate.

### Task 8 — Understand Why `-k` Was Needed (Certificate Trust)
```
! From PC1, without -k
curl -v https://192.168.10.1
```
This should fail certificate validation, since PC1 doesn't trust this self-signed certificate's issuer by default — exactly the same reason a browser would show a security warning. In a real deployment with a properly CA-signed certificate, and the CA's root certificate installed in the client's trust store, this step would succeed without needing to bypass validation at all. For this lab, confirm you understand *why* `-k` was necessary rather than treating it as a normal/ignorable flag.

### Task 9 — Inspect the Negotiated TLS Version and Cipher
```
! From PC1
curl -v https://192.168.10.1 -k 2>&1 | grep -i "SSL connection"
```
Note which TLS version and cipher suite were negotiated by default.

### Task 10 — Enforce a Minimum TLS Version
Restrict the HTTPS server to only accept modern, secure TLS versions:
```
ip http tls-version TLSv1.2
```
> Depending on IOS version/platform, this command may accept a specific minimum version or a defined set of allowed versions — check `ip http tls-version ?` on your platform for exact available options, and choose the most restrictive option that still meets your lab/course requirements (TLS 1.2 or higher is a reasonable modern minimum; TLS 1.0/1.1 are deprecated and should not be accepted).

### Task 11 — Verify Legacy TLS Versions Are Rejected
```
! From PC1, forcing an old TLS version
curl -v --tlsv1.0 --tls-max 1.0 https://192.168.10.1 -k
```
Confirm this connection **fails** — the server should refuse to negotiate a TLS version below the configured minimum. Then confirm a compliant connection still works:
```
curl -v --tlsv1.2 https://192.168.10.1 -k
```
Should succeed.

### Task 12 — Verify
```
show crypto pki trustpoints
show crypto pki certificates
show ip http server secure status
show run | section crypto pki
show run | include ip http
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 3 — default state | Auto-generated self-signed certificate present, generic subject |
| Task 7 — trustpoint-based certificate | Certificate subject matches the explicitly configured `CN=r1.lab.local,O=LabOrg,C=US` |
| Task 8 — validation without `-k` | Fails, correctly demonstrating untrusted self-signed certificate behavior |
| Task 9 — negotiated TLS/cipher | Confirms a specific version/cipher was used by default (baseline before restriction) |
| Task 11 — legacy TLS forced | Connection fails once minimum TLS 1.2 is enforced; a TLS 1.2 connection still succeeds |
| `show crypto pki trustpoints` | Lists `R1-HTTPS-TRUSTPOINT` with correct enrollment and key pair association |

---

## Challenge (Optional)
- If your lab platform includes a way to simulate a real CA (or a second router configured as a simple CA server), repeat Task 4's trustpoint using `enrollment terminal` or `enrollment url` instead of `enrollment selfsigned`, generate a real CSR, have it signed, and import the resulting certificate — then repeat Task 8's validation test and confirm it now succeeds *without* `-k`, once PC1 trusts the issuing CA.
- Research and document the specific security weaknesses of TLS 1.0 and TLS 1.1 (the versions this lab's Task 10 blocks) that led to their deprecation, and explain why simply supporting a newer version alongside old ones isn't sufficient — a client or attacker can often still force a downgrade unless the old versions are explicitly disabled, as demonstrated in Task 11.
- Combine this lab with the earlier HTTP Service Verification lab's access-class and local-authentication configuration, and confirm all three layers (network-level ACL restriction, authentication, and now proper certificate/TLS hardening) work together without conflicting.