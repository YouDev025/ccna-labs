# Lab: VPN Security

## Objective
This lab focuses on **hardening the cryptographic parameters of a VPN itself**, distinct from the VPN ACLs lab (which focused on interesting-traffic/NAT-exemption/perimeter ACLs) and the Remote Access Security lab (which focused on user authentication and split tunneling). You will start with a **deliberately weak** IPsec configuration, confirm it technically works but understand why it's insecure, then harden it step by step: modern encryption/hashing, **Perfect Forward Secrecy (PFS)**, **Dead Peer Detection (DPD)**, and finally confirm that a peer still proposing only weak parameters is correctly **rejected** once hardening is in place.

---

## Topology

```
   Site A                                                          Site B
 192.168.10.0/24                                                192.168.20.0/24
        |                                                                |
      SW-A                                                             SW-B
        |                                                                |
  Gi0/0 | 192.168.10.1/24                                192.168.20.1/24 | Gi0/0
   +-----------+                                                  +-----------+
   |    R1     |  Gi0/1                                    Gi0/1   |    R2     |
   +-----------+  203.0.113.1/30 -------- ISP1 -------- 203.0.113.2/30 +-----------+
        |                                                                |
      PC1                                                              PC2
  192.168.10.10                                                   192.168.20.10
```

- Same basic site-to-site shape as the VPN ACLs lab, but the focus here is entirely on the crypto parameters used to build the tunnel, not the traffic-selection ACLs around it.

---

## IP Addressing Table

| Device | Interface | IP Address       | Subnet Mask       |
|--------|-----------|-------------------|---------------------|
| R1     | Gi0/0     | 192.168.10.1       | 255.255.255.0        |
| R1     | Gi0/1     | 203.0.113.1          | 255.255.255.252      |
| R2     | Gi0/0     | 192.168.20.1       | 255.255.255.0        |
| R2     | Gi0/1     | 203.0.113.2          | 255.255.255.252      |
| PC1    | NIC       | 192.168.10.10        | 255.255.255.0        |
| PC2    | NIC       | 192.168.20.10        | 255.255.255.0        |

---

## Part 1 — A Deliberately Weak VPN (Understand the Risk)

### Task 1 — Build the Topology
Place R1, R2, PC1, PC2, and apply the addressing above. Configure basic routing so R1 and R2 can reach each other's public interface.

### Task 2 — Configure a Weak ISAKMP Policy
```
! On both R1 and R2
crypto isakmp policy 10
 encryption des
 hash md5
 authentication pre-share
 group 1
crypto isakmp key WeakLabKey123 address <peer's Gi0/1 address>
```
> This deliberately uses **DES** (a 56-bit cipher broken decades ago), **MD5** (a hash function with known collision weaknesses), and **DH group 1** (a 768-bit key exchange group considered far too weak for any real security today) — every one of these is a real historical default or common misconfiguration that this lab exists to correct.

### Task 3 — Configure a Weak Transform Set
```
crypto ipsec transform-set WEAK-TSET esp-des esp-md5-hmac

crypto map WEAK-VPN-MAP 10 ipsec-isakmp
 set peer <other router's Gi0/1 address>
 set transform-set WEAK-TSET
 match address VPN-TRAFFIC
```
```
ip access-list extended VPN-TRAFFIC
 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255   (mirrored on R2 with source/destination reversed)
```
```
interface GigabitEthernet0/1
 crypto map WEAK-VPN-MAP
```

### Task 4 — Verify the Weak Tunnel Technically Works
```
! From PC1
ping 192.168.20.10
```
```
! On R1 or R2
show crypto isakmp sa
show crypto ipsec sa
```
Confirm the tunnel forms and traffic passes successfully — **this is the point of Part 1**: a weak configuration is not necessarily a *broken* one. It functions completely normally from a connectivity standpoint, which is exactly why weak crypto choices are dangerous — nothing about normal operation warns you that the protection is inadequate.

### Task 5 — Understand What Each Weakness Actually Costs You
In your lab notes, briefly note the real-world risk of each choice from Task 2/3:
- **DES**: a 56-bit key is small enough to be brute-forced with modern hardware in a practical timeframe — this cipher should never be used for anything sensitive today.
- **MD5**: known collision attacks make it unsuitable as a security-critical hash, even though it remains fine for non-security checksums.
- **DH group 1**: a 768-bit prime is far below what's considered safe against modern factoring capability, weakening the key exchange itself, independent of the cipher used afterward.

---

## Part 2 — Hardening to Modern Parameters

### Task 6 — Replace the ISAKMP Policy with Strong Parameters
```
! On both R1 and R2
no crypto isakmp policy 10
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 19
crypto isakmp key StrongLabKey456 address <peer's Gi0/1 address>
```
> DH group 19 uses a 256-bit elliptic curve, offering strong security with better performance than an equivalently strong traditional (MODP) group — a good modern default where supported.

### Task 7 — Replace the Transform Set with Strong Parameters
```
crypto ipsec transform-set STRONG-TSET esp-aes 256 esp-sha256-hmac
crypto map WEAK-VPN-MAP 10 ipsec-isakmp
 set transform-set STRONG-TSET
```

### Task 8 — Add Perfect Forward Secrecy (PFS)
Without PFS, if a long-term key were ever compromised, an attacker could potentially decrypt **previously captured** traffic. PFS ensures each session uses a fresh, independent key exchange, so compromising one session's key material doesn't retroactively expose past sessions:
```
crypto map WEAK-VPN-MAP 10 ipsec-isakmp
 set pfs group19
```
> The PFS group here doesn't need to match the ISAKMP policy's DH group, but using the same or an equally strong group is common practice.

### Task 9 — Add Dead Peer Detection (DPD)
DPD allows a router to detect a genuinely unresponsive peer (not just an idle one) and tear down a stale tunnel proactively, rather than holding onto dead security associations indefinitely:
```
crypto isakmp keepalive 10 3
```
> This sends a keepalive every 10 seconds, and considers the peer dead after 3 consecutive missed responses — tune these values based on how quickly you need failure detection versus how much keepalive overhead is acceptable.

### Task 10 — Verify the Hardened Tunnel
```
! From PC1
ping 192.168.20.10
```
```
! On R1 or R2
show crypto isakmp sa detail
show crypto ipsec sa
```
Confirm the tunnel re-establishes successfully using the new, strong parameters — encryption, hash, and DH group should all reflect the Task 6/7 values, not the original weak ones.

---

## Part 3 — Confirm Weak Proposals Are Now Rejected

### Task 11 — Simulate a Peer Still Offering Only Weak Parameters
To test that hardening actually enforces a minimum standard (rather than just being the parameters you happen to have configured), temporarily revert **only R2** back to the weak policy from Part 1, leaving R1 fully hardened:
```
! On R2 only
no crypto isakmp policy 10
crypto isakmp policy 10
 encryption des
 hash md5
 authentication pre-share
 group 1
crypto map WEAK-VPN-MAP 10 ipsec-isakmp
 set transform-set WEAK-TSET
```

### Task 12 — Verify the Tunnel Now Fails to Form
```
! Clear any existing SA first
clear crypto isakmp
clear crypto sa
```
```
! From PC1
ping 192.168.20.10
```
This should now **fail** — R1 (hardened) and R2 (weak) can no longer agree on a mutually acceptable ISAKMP policy, since R1 no longer offers/accepts the weak parameters R2 is proposing.
```
! On R1
show crypto isakmp sa
debug crypto isakmp   (optional, generate a connection attempt then 'undebug all')
```
Confirm no security association forms, and (if using the debug) that the negotiation failure is specifically due to no matching policy being found — this is the hardening working as intended, not a misconfiguration to be "fixed" by weakening R1 back down.

### Task 13 — Restore R2 to the Hardened Configuration
```
! On R2
no crypto isakmp policy 10
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 19
crypto map WEAK-VPN-MAP 10 ipsec-isakmp
 set transform-set STRONG-TSET
```
```
! From PC1
ping 192.168.20.10
```
Should succeed again, confirming both sides are back to a consistent, strong configuration.

### Task 14 — Final Verification
```
show crypto isakmp policy
show crypto ipsec transform-set
show crypto map
show crypto isakmp sa detail
show run | section crypto
```

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 4 — weak tunnel | Forms and passes traffic successfully, despite using DES/MD5/DH1 — demonstrating that "it works" is not the same as "it's secure" |
| Task 10 — hardened tunnel | Forms successfully using AES-256/SHA-256/DH19, with PFS and DPD configured |
| Task 12 — mismatched hardening | Tunnel fails to form when one side is hardened and the other still proposes only weak parameters |
| Task 13 — both sides hardened | Tunnel re-establishes successfully once both sides match on strong parameters |
| `show crypto isakmp sa detail` (after Task 10) | Reflects the strong encryption/hash/DH group actually in use, confirming the hardening took effect and isn't just configured-but-unused |

---

## Challenge (Optional)
- Research and compare **IKEv1** (used throughout this lab) against **IKEv2**, specifically noting at least two security or reliability improvements IKEv2 offers (e.g., built-in DPD/liveness checking rather than the separate keepalive command used in Task 9, and better resistance to certain DoS patterns during negotiation).
- Configure a second, **fallback** ISAKMP policy (a different sequence number, e.g., policy 20) on both routers offering slightly different but still-strong parameters, and confirm the routers negotiate down to whichever mutually-acceptable policy is highest-priority (lowest sequence number) that both sides actually support — demonstrating how multiple policies allow flexibility without reintroducing weak options.
- Write a short migration plan (in writing) for an organization that discovers it has several existing site-to-site VPNs still using DES/MD5/DH1 in production — specifically addressing how you would coordinate a hardening rollout across multiple sites without causing a simultaneous outage everywhere, referencing this lab's Task 11/12 observation about mismatched policies breaking connectivity entirely.