# Lab: VLAN Design

## Objective
Unlike the earlier VLAN labs (which handed you a topology and VLAN plan already decided), this lab starts from a **business requirements scenario** and has you make the actual design decisions — VLAN numbering, naming, sizing, and management-plane separation — before implementing and verifying them. Good VLAN design is a genuinely different skill from VLAN configuration syntax, and this lab is built to exercise it specifically.

---

## Scenario
**Riverside Logistics** is moving into a new single-floor office and needs their network segmented sensibly. You've been given the following requirements:

- **Warehouse Operations** (~40 devices, mostly handheld scanners and a few workstations) — needs its own segment, expected to grow moderately over the next two years.
- **Office Staff** (~25 devices, standard workstations) — general administrative use.
- **IP Phones** (~30 devices, one per office staff desk plus a few warehouse phones) — should be logically separate from data traffic even where they share a physical port with a PC.
- **Guest Wi-Fi** (variable, unpredictable count, should be assumed to grow significantly) — must be fully isolated from all internal traffic.
- **Servers** (~8 devices) — a small internal server room, should be isolated from general user segments.
- **Network Management** — every switch's management interface needs its own segment, separate from all user-facing traffic, reachable only by IT staff.

You have **three switches** available: one per physical area (Warehouse, Office, Server Room), all interconnected.

---

## Part 1 — Design (Do This Before Any Configuration)

### Task 1 — Decide on a VLAN Numbering Scheme
Before assigning specific numbers, decide on a **consistent, documented convention** — real organizations often group VLAN numbers by function or site (e.g., 10s for data, 100s for voice, 900s for management/infrastructure) so that, months later, someone can infer a VLAN's rough purpose from its number alone without checking documentation. Write out your proposed numbering convention and the reasoning behind it before assigning actual VLAN IDs in the next task.

### Task 2 — Assign Specific VLAN IDs and Names
Using your convention from Task 1, assign a VLAN ID and a clear, consistent name to each of the six requirements above. There's no single correct numbering, but your choices should be **internally consistent** and each name should make the VLAN's purpose obvious without needing to cross-reference a separate document. Document this as a table (VLAN ID | Name | Purpose | Approx. device count).

### Task 3 — Decide on Subnet Sizing Per VLAN
For each VLAN, choose an appropriately-sized subnet — not simply defaulting every VLAN to a /24 regardless of actual device count. Consider:
- Warehouse Operations (~40 devices, moderate growth expected) — how much headroom is reasonable without being wasteful?
- Guest Wi-Fi (unpredictable, expected significant growth) — should this get **more** headroom than a precisely-known department, and why?
- Servers (~8 devices, unlikely to grow much) — does this need anywhere near a /24?

Document your chosen subnet (address range and mask) for each VLAN, with a one-line justification for the sizing decision specifically (not just "because that's a common size").

### Task 4 — Decide Where the Management VLAN Fits
The requirements specify management traffic must be separate from all user-facing VLANs. Decide:
- Should the management VLAN be routable at all beyond the local network, or restricted to IT staff only via ACL, and why?
- Should it use VLAN 1 (the default) or a dedicated, deliberately unused-looking number? Justify your choice by referencing what you know about VLAN 1's default behavior and common hardening advice (this connects directly to the native VLAN hardening practice from the Trunking lab, even though management VLAN and native VLAN are not the same thing — explain the distinction in your own words as part of this task).

### Task 5 — Decide How Voice and Data Coexist Per Port
Given that IP phones and PCs will frequently share a single physical port (a phone with a PC daisy-chained behind it, as in the Advanced Port Security lab), decide how you'll configure this at the port level — reference what you already know about `switchport voice vlan` and per-port maximum MAC counts, and write out your intended port-level template before implementing it.

---

## Part 2 — Implementation

### Task 6 — Build the Topology
1. Set up three switches (WAREHOUSE-SW, OFFICE-SW, SERVER-SW) interconnected in whatever physical arrangement makes sense (a triangle, or a hub-and-spoke through a fourth core switch, is fine — document your choice).
2. Create every VLAN from your Task 2 table on all switches that need it (not every VLAN needs to exist everywhere — Servers, for instance, likely only needs to exist on SERVER-SW and wherever the routing/gateway device lives).

### Task 7 — Configure Access Ports Per Your Design
Apply your Task 5 voice+data template to representative office ports, and standard single-VLAN access configuration (hardened per the Access Port Configuration lab's baseline) to warehouse and server ports.

### Task 8 — Configure Trunks Between Switches
Configure trunks carrying only the VLANs actually needed on each specific link (not a blanket "allow everything") — refer back to the Trunking lab's `switchport trunk allowed vlan` practice, and set an explicit, non-default native VLAN consistently across all trunk links.

### Task 9 — Implement the Management VLAN
Configure each switch's management (SVI) interface in your chosen management VLAN, and apply an ACL restricting management access (SSH) to only a designated IT subnet, per your Task 4 decision.

### Task 10 — Verify the Design Holds Up
```
show vlan brief
show interfaces trunk
show ip interface brief   (or show interfaces vlan <mgmt-vlan> as applicable)
show run | section switchport
```
On each switch, confirm:
- Only the VLANs actually needed on that switch/link are present/allowed.
- The management VLAN is correctly isolated and only reachable per your ACL.
- Voice+data ports show both a voice and access VLAN correctly.

### Task 11 — Stress-Test Your Own Design
Answer honestly, referencing your actual configuration:
1. If Guest Wi-Fi grew to 150 devices next quarter, would your Task 3 subnet sizing accommodate that without renumbering? If not, was that an acceptable tradeoff given the "significant growth" requirement, or should you revise it now?
2. If a new department were added tomorrow, does your Task 1 numbering convention make it obvious where the new VLAN should go, or would it require deviating from your own scheme?
3. Could a device connected to a warehouse port ever reach the server VLAN directly at Layer 2, given your trunk allowed-VLAN configuration from Task 8? Verify this directly rather than assuming.

### Task 12 — Final Verification
```
show vlan brief
show interfaces trunk
show run
```
Compile your Task 2–5 design decisions and your Task 11 self-review into a single design document alongside the configuration — this document is as much a deliverable of this lab as the working configuration itself.

---

## Verification Checklist

| Check | Expected Result |
|---|---|
| Task 2 — VLAN plan | Consistent numbering convention applied; every VLAN has a clear name |
| Task 3 — subnet sizing | Each VLAN's subnet size is justified by actual/expected device count, not uniformly defaulted |
| Task 9 — management VLAN | Isolated from user VLANs; restricted by ACL to IT subnet only |
| Task 10 — implementation matches design | `show vlan brief` and `show interfaces trunk` reflect exactly what was planned in Part 1, on every switch |
| Task 11 — self-review | Honest answers grounded in actual verification, not assumption |

---

## Challenge (Optional)
- Redesign the scenario assuming Riverside Logistics will open a **second site** next year, connected via WAN — revise your VLAN numbering convention (Task 1) to accommodate a site-identifier component (e.g., VLAN ranges per site) without renumbering anything at the existing site.
- Calculate the actual usable host counts for each subnet you chose in Task 3, and identify whether any VLAN was sized so tightly that even modest, unplanned growth (a handful of extra devices) would force a redesign — revise any that are too tight.
- Present your design (informally, in writing) as if defending it to a colleague who proposed simply using one large /22 for the entire office with no VLAN segmentation at all — write out the specific arguments you'd make for why the segmented design in this lab is worth the added complexity.