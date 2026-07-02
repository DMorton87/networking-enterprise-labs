# Lab 07 — DHCP Snooping & Dynamic ARP Inspection

A hands-on Cisco Packet Tracer lab implementing Layer 2 security controls against two of the most common and dangerous attacks on switched networks — rogue DHCP servers and ARP poisoning. Covers DHCP snooping binding table construction, trusted/untrusted port design, Dynamic ARP Inspection validation, and the security reasoning behind each configuration decision.

---

## Overview

Layer 3 controls like ACLs and IPsec are blind to Layer 2 attacks. DHCP Snooping and Dynamic ARP Inspection (DAI) are the switch-level answer, and they work together as a pair:

- **DHCP Snooping** — the switch inspects DHCP traffic and only allows offers and acknowledgements from explicitly trusted ports. It also builds a binding table mapping each client's MAC address, IP address, VLAN, and interface.
- **Dynamic ARP Inspection** — uses the DHCP snooping binding table to validate every ARP packet on untrusted ports. If a host sends an ARP claiming an IP that doesn't match its binding table entry, the switch drops the packet silently.

The two attacks this lab defends against:

| Attack | Method | Impact |
|--------|--------|--------|
| Rogue DHCP server | Responds to DHCP discoveries faster than the legitimate server | Clients get fake gateway — all traffic redirected through attacker |
| ARP poisoning | Sends gratuitous ARPs claiming to be the gateway | Poisons ARP cache of other hosts — man-in-the-middle position |

Both are silent, both are effective against unprotected networks, and both are stopped by the controls in this lab.

---

## Objectives

- Enable DHCP snooping on all switches and restrict trusted ports to uplinks and legitimate server ports only
- Verify the snooping binding table populates correctly after clients receive DHCP leases
- Enable DAI on all switches and confirm ARP packets from the rogue host are dropped
- Understand the trust port design trade-off and implement the more secure access-layer model
- Document the difference between PT's DAI drop reporting and real-world counter behaviour

---

## Topology

```
       [ Router ]            [ DHCP server ]
       10.0.0.1/24           10.0.0.100/24
       Fa0/1 ← trusted       Fa0/2 ← trusted
            \                    /
             \                  /
              [ SW-DIST (3560) ]
              DHCP snooping + DAI
              Gi0/1 ← trusted    Gi0/2 ← trusted
             /                        \
      [ SW1 (2960) ]              [ SW2 (2960) ]
      DHCP snooping + DAI         DHCP snooping + DAI
      Gi0/1 trusted (uplink)      Gi0/1 trusted (uplink)
      Fa0/1-3 untrusted           Fa0/1-3 untrusted
        |       |       |           |       |       |
       PC1     PC2  Rogue DHCP    PC3     PC4   Rogue host
                    (blocked)               (ARP blocked)
```

---

## Design decision — where to enforce trust

A key question in this lab is which ports should be marked trusted on SW-DIST. The initial instinct is to trust the trunk ports toward SW1 and SW2 so that legitimate DHCP responses can flow downstream to clients. However this creates a gap — if a rogue DHCP server were connected to SW1 and managed to get offers onto the trunk, those offers would pass through SW-DIST unchecked.

The more secure and correct model is:

- **SW-DIST**: Trust only the ports directly connected to the legitimate router and DHCP server. Leave trunk ports toward SW1 and SW2 untrusted.
- **SW1 and SW2**: Trust only the uplink toward SW-DIST. Leave all access ports untrusted.

With this model, a rogue DHCP server plugged into any access port is blocked at the access switch immediately — before the offer ever reaches the trunk. Legitimate DHCP offers from the real server are validated at SW-DIST (trusted ingress on Fa0/2) and flow downstream without issue, because DHCP snooping inspects offers on **ingress** at the untrusted port — not on egress — so the offer passing outbound through SW-DIST's untrusted trunk port is not re-inspected.

| Switch | Trusted ports | Untrusted ports |
|--------|--------------|-----------------|
| SW-DIST | Fa0/1 (router), Fa0/2 (DHCP server) | Gi0/1, Gi0/2 (toward access switches) |
| SW1 | Gi0/1 (uplink to SW-DIST) | Fa0/1, Fa0/2, Fa0/3 (all client ports) |
| SW2 | Gi0/1 (uplink to SW-DIST) | Fa0/1, Fa0/2, Fa0/3 (all client ports) |

---

## Device list

| Device | PT model | Role |
|--------|----------|------|
| SW-DIST | 3560-24PS | Distribution switch — DHCP snooping + DAI |
| SW1 | 2960-24TT | Access switch — VLAN 10 |
| SW2 | 2960-24TT | Access switch — VLAN 10 |
| Router | ISR4331 | Default gateway — 10.0.0.1 |
| DHCP server | Server-PT | Legitimate DHCP server — 10.0.0.100 |
| PC1–PC4 | PC-PT | DHCP clients |
| Rogue DHCP | Server-PT | Simulates rogue DHCP server — blocked by snooping |
| Rogue host | PC-PT | Simulates ARP poisoner — blocked by DAI |

---

## IP addressing

| Device | Address | Note |
|--------|---------|------|
| Router | 10.0.0.1/24 | Static — default gateway |
| DHCP server | 10.0.0.100/24 | Static — legitimate server |
| PC1–PC4 | DHCP — 10.0.0.10+ | Dynamic from legitimate server |
| Rogue DHCP | — | Offers 192.168.99.x — should never reach clients |
| Rogue host | 10.0.0.200/24 | Static — no snooping binding, ARP dropped by DAI |

---

## Cabling

| Cable | From | Interface | To | Interface | Trust state |
|-------|------|-----------|----|-----------|-------------|
| 1 | Router | G0/0/0 | SW-DIST | Fa0/1 | Trusted |
| 2 | DHCP server | Fa0 | SW-DIST | Fa0/2 | Trusted |
| 3 | SW-DIST | Gi0/1 | SW1 | Gi0/1 | Untrusted on SW-DIST, trusted on SW1 |
| 4 | SW-DIST | Gi0/2 | SW2 | Gi0/1 | Untrusted on SW-DIST, trusted on SW2 |
| 5 | SW1 | Fa0/1 | PC1 | Fa0 | Untrusted |
| 6 | SW1 | Fa0/2 | PC2 | Fa0 | Untrusted |
| 7 | SW1 | Fa0/3 | Rogue DHCP | Fa0 | Untrusted |
| 8 | SW2 | Fa0/1 | PC3 | Fa0 | Untrusted |
| 9 | SW2 | Fa0/2 | PC4 | Fa0 | Untrusted |
| 10 | SW2 | Fa0/3 | Rogue host | Fa0 | Untrusted |

---

## Key configurations

### Step 1 — VLANs (all switches)

```
vlan 10
 name USERS
```

### Step 2 — Trunks and access ports on SW-DIST

```
! Trunks toward access switches
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10

interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10

! Access ports for router and DHCP server
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10
```

### Step 3 — Access ports on SW1 and SW2 (identical)

```
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10

interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10
```

### Step 4 — DHCP Snooping on SW-DIST

```
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

! Trust only legitimate server-facing ports
! Trunk ports toward SW1 and SW2 are intentionally left untrusted
interface FastEthernet0/1
 ip dhcp snooping trust

interface FastEthernet0/2
 ip dhcp snooping trust
```

### Step 5 — DHCP Snooping on SW1 and SW2 (identical)

```
ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

! Trust only the uplink toward SW-DIST
! All access ports remain untrusted by default
interface GigabitEthernet0/1
 ip dhcp snooping trust
```

### Step 6 — Dynamic ARP Inspection on SW-DIST

```
ip arp inspection vlan 10

! Trust ports connected to devices with static IPs
! Router and DHCP server have no snooping binding entries
! so their ARP packets must be explicitly trusted
interface FastEthernet0/1
 ip arp inspection trust

interface FastEthernet0/2
 ip arp inspection trust

! Trunk ports toward access switches — trusted so validated
! ARP from legitimate clients can reach the rest of the network
interface GigabitEthernet0/1
 ip arp inspection trust

interface GigabitEthernet0/2
 ip arp inspection trust
```

### Step 7 — Dynamic ARP Inspection on SW1 and SW2 (identical)

```
ip arp inspection vlan 10

! Trust only the uplink — all access ports are untrusted by default
interface GigabitEthernet0/1
 ip arp inspection trust
```

### Step 8 — DHCP server pool

Configure in Services → DHCP on the legitimate Server-PT:

| Field | Value |
|-------|-------|
| Pool name | USERS |
| Default gateway | 10.0.0.1 |
| DNS server | 8.8.8.8 |
| Start IP | 10.0.0.10 |
| Subnet mask | 255.255.255.0 |

Configure the rogue DHCP server with a pool offering `192.168.99.x` addresses — if any PC receives a `192.168.99.x` lease, snooping has failed.

---

## Verification

```
! Confirm snooping is active and trusted ports are correct
show ip dhcp snooping

! View the binding table — one entry per DHCP client
! Shows MAC, IP, VLAN, lease time, and interface
show ip dhcp snooping binding

! Confirm DAI is active for VLAN 10
show ip arp inspection

! View ARP drop counters per interface
show ip arp inspection statistics

! View forwarded vs dropped per VLAN
show ip arp inspection vlan 10
```

### What to look for

| Check | Expected result |
|-------|----------------|
| `show ip dhcp snooping binding` | One entry per PC — MAC, 10.0.0.x, VLAN 10, interface |
| PC1–PC4 DHCP lease | All receive 10.0.0.x — never 192.168.99.x |
| Rogue DHCP offer | Dropped at SW1 Fa0/3 — never reaches clients |
| `show ip arp inspection statistics` | Drop counter increments when rogue host sends ARP |
| Rogue host ping attempt | ARP dropped at SW2 — ping never completes |
| PC1 ping PC3 | Succeeds — legitimate traffic unaffected |

---

## Concepts demonstrated

| Concept | IOS command |
|---------|-------------|
| DHCP snooping global | `ip dhcp snooping` |
| Snooping per VLAN | `ip dhcp snooping vlan X` |
| Trusted port | `ip dhcp snooping trust` |
| Disable option 82 | `no ip dhcp snooping information option` |
| View binding table | `show ip dhcp snooping binding` |
| DAI per VLAN | `ip arp inspection vlan X` |
| DAI trusted port | `ip arp inspection trust` |
| DAI statistics | `show ip arp inspection statistics` |

---

## Lessons learned

### Issue 1 — Trust port placement on SW-DIST

**Initial configuration:** Trunk ports on SW-DIST toward SW1 and SW2 were marked trusted to allow legitimate DHCP responses to flow downstream to clients.

**Problem identified:** Trusting trunk ports toward access switches creates a gap. A rogue DHCP server connected to an access port could theoretically get offers onto the trunk, where they would pass through SW-DIST unchecked.

**Corrected approach:** Trust only the ports directly connected to the legitimate router and DHCP server on SW-DIST. Leave trunk ports untrusted. DHCP snooping validates offers on ingress at the untrusted port — not on egress — so legitimate offers validated at SW-DIST's Fa0/2 flow downstream through the untrusted trunk without being re-inspected or dropped. The rogue DHCP server plugged into SW1 Fa0/3 is stopped at SW1 itself before the offer ever reaches the trunk.

**Key takeaway:** DHCP snooping is most effective when enforced at the access layer, closest to the client. Trusting uplinks toward the distribution layer is correct; trusting downlinks toward access switches is not.

---

### Issue 2 — Rogue host IP address and ARP triggering

**Initial approach:** Configured the rogue host with IP `10.0.0.1` to impersonate the gateway.

**Problem:** Packet Tracer may suppress the automatic gratuitous ARP when a host is assigned an IP that conflicts with another device already on the network, making DAI difficult to test.

**Fix:** Assign the rogue host a non-conflicting static IP (`10.0.0.200/24`) and trigger ARP activity manually by pinging a legitimate host. Since `10.0.0.200` has no DHCP snooping binding entry (it was assigned a static IP, not a DHCP lease), DAI drops its ARP regardless — which is the correct real-world behaviour for any host not in the binding table.

**Key takeaway:** In a real network, DAI drops ARP packets from any host whose IP/MAC combination is not in the snooping binding table. Static IP hosts on protected VLANs must either be on trusted ports or have static ARP inspection entries configured manually using `ip arp inspection filter`.

---

### Issue 3 — DAI drop counter behaviour in Packet Tracer

**Symptom:** After confirming in simulation mode that ARP packets from the rogue host were being dropped, `show ip arp inspection statistics` showed zero drops.

**Root cause:** Packet Tracer's DAI counter reporting is inconsistent in some builds. The drop is occurring — simulation mode showed the ARP frame being processed and discarded at the switch with the message "The active VLAN interface is not up. The ARP process ignores the frame" — but the statistics counter does not always increment.

**Workaround:** Use simulation mode packet tracing as the primary verification method for DAI in PT. The step-by-step PDU trace is actually more instructive than a counter number — it shows exactly which switch processed the frame, at which step it was dropped, and why. This output is worth screenshotting for documentation.

**Key takeaway:** In a production environment, DAI drop counters are reliable and should be monitored as part of normal security operations. A sudden spike in DAI drops on an access port is a strong signal of an active ARP poisoning attempt and warrants immediate investigation.

---

## Known limitations

### Static IP hosts require manual ARP inspection entries

Any host with a static IP has no DHCP snooping binding. On untrusted ports, DAI will drop its ARP packets regardless of whether it is a legitimate host. In production, static IP devices on protected VLANs are handled using ARP access control lists:

```
arp access-list STATIC_HOSTS
 permit ip host 10.0.0.1 mac host <router-mac>
 permit ip host 10.0.0.100 mac host <server-mac>

ip arp inspection filter STATIC_HOSTS vlan 10
```

This lab avoids the issue by placing the router and DHCP server on trusted ports, but in a larger network with static IP hosts scattered across access ports this would be required.

### DHCP snooping does not prevent DHCP starvation

DHCP snooping stops rogue servers but does not prevent a client from sending large numbers of DISCOVER packets with spoofed MAC addresses to exhaust the IP pool. Rate limiting addresses this:

```
interface FastEthernet0/1
 ip dhcp snooping limit rate 10
```

This limits DHCP packets to 10 per second on untrusted ports. Exceeding the limit puts the port into err-disabled state.

---

## Files

| File | Description |
|------|-------------|
| `lab07.pkt` | Packet Tracer source file (PT 8.x) |
| `topology.png` | Network diagram |
| `configs/SW-DIST.txt` | Final running configuration |
| `configs/SW1.txt` | Final running configuration |
| `configs/SW2.txt` | Final running configuration |

---

## Skills demonstrated

`DHCP snooping` · `Dynamic ARP Inspection` · `Trust port design` · `Binding table` · `Layer 2 attack prevention` · `Rogue DHCP mitigation` · `ARP poisoning mitigation` · `Access layer security` · `Simulation mode verification` · `DAI static ARP entries` · `DHCP rate limiting`
