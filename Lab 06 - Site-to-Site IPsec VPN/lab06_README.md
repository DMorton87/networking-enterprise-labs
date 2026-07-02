# Lab 06 — Site-to-Site IPsec VPN

A hands-on Cisco Packet Tracer lab implementing a site-to-site IPsec VPN tunnel between two office locations over a simulated internet. Covers IKEv1 Phase 1 and Phase 2 negotiation, pre-shared key authentication, ESP encryption, and systematic troubleshooting of ISAKMP negotiation failures using CLI state output.

---

## Overview

Two geographically separated sites need to communicate securely over a public network. Traffic between the Site A LAN (`10.1.0.0/24`) and the Site B LAN (`10.2.0.0/24`) must be encrypted end-to-end — the ISP in the middle should see nothing but ESP-encapsulated packets addressed to the router WAN IPs.

This lab implements that using IKEv1 in main mode with AES-256 encryption, SHA hashing, DH group 5, and pre-shared key authentication. The tunnel is triggered by interesting traffic matching a crypto ACL and verified using `show crypto isakmp sa` and `show crypto ipsec sa` output.

---

## Objectives

- Configure IKEv1 Phase 1 (ISAKMP policy) identically on both VPN gateways
- Configure Phase 2 (transform set and crypto map) and apply to the correct WAN interface
- Define interesting traffic using a named extended ACL
- Verify Phase 1 reaches `QM_IDLE` state and Phase 2 shows incrementing encaps/decaps counters
- Use simulation mode to confirm WAN traffic is ESP-encapsulated rather than cleartext
- Diagnose and resolve real ISAKMP negotiation failures using state output

---

## Topology

```
   [ Site A — HQ ]                                    [ Site B — branch ]
   10.1.0.0/24                                         10.2.0.0/24

   PC-A1  PC-A2                                        PC-B1  PC-B2
     |      |                                             |      |
    SW-A (2960)                                         SW-B (2960)
        |                                                   |
       R1 (ISR4331)                                        R2 (ISR4331)
       G0/0/0 — 10.1.0.1/24  (LAN)                       G0/0/0 — 10.2.0.1/24  (LAN)
       G0/0/1 — 203.0.113.2/30 (WAN)                     G0/0/1 — 203.0.113.6/30 (WAN)
            |                                                  |
            └──────────── [ ISP (ISR4321) ] ──────────────────┘
                          203.0.113.1/30 — 203.0.113.5/30

                    ══════════════════════════════
                         IPsec tunnel (ESP)
                    ══════════════════════════════
```

---

## How IPsec works — two-phase negotiation

Before any traffic flows, the two routers negotiate the tunnel in two phases:

| Phase | Protocol | Purpose |
|-------|----------|---------|
| Phase 1 | IKE (ISAKMP) | Authenticate each router and establish a secure management channel |
| Phase 2 | IPsec | Negotiate encryption parameters for the actual data tunnel |

Phase 1 produces an ISAKMP SA. Phase 2 produces an IPsec SA. Both phases must be configured identically on each router — any mismatch causes negotiation to stall silently at a specific main mode exchange, which is diagnosable from the state field in `show crypto isakmp sa`.

The first packet that triggers the tunnel is always dropped while IKE negotiates. This is expected — send a second ping after the first drops.

---

## Device list

| Device | PT model | Role |
|--------|----------|------|
| R1 | ISR4331 | Site A VPN gateway |
| R2 | ISR4331 | Site B VPN gateway |
| ISP | ISR4321 | Simulates internet transit |
| SW-A | 2960-24TT | Site A LAN switch |
| SW-B | 2960-24TT | Site B LAN switch |
| PC-A1, PC-A2 | PC-PT | Site A LAN clients |
| PC-B1, PC-B2 | PC-PT | Site B LAN clients |

---

## IP addressing

| Device | Interface | Address | Purpose |
|--------|-----------|---------|---------|
| R1 | G0/0/0 | 10.1.0.1/24 | LAN gateway — Site A |
| R1 | G0/0/1 | 203.0.113.2/30 | WAN — outside |
| ISP | G0/0/0 | 203.0.113.1/30 | Link to R1 |
| ISP | G0/0/1 | 203.0.113.5/30 | Link to R2 |
| R2 | G0/0/0 | 10.2.0.1/24 | LAN gateway — Site B |
| R2 | G0/0/1 | 203.0.113.6/30 | WAN — outside |
| PC-A1 | Fa0 | 10.1.0.10/24 | Site A client |
| PC-A2 | Fa0 | 10.1.0.11/24 | Site A client |
| PC-B1 | Fa0 | 10.2.0.10/24 | Site B client |
| PC-B2 | Fa0 | 10.2.0.11/24 | Site B client |

---

## Cabling

| Cable | From | Interface | To | Interface |
|-------|------|-----------|----|-----------|
| 1 | R1 | G0/0/1 | ISP | G0/0/0 |
| 2 | R2 | G0/0/1 | ISP | G0/0/1 |
| 3 | R1 | G0/0/0 | SW-A | Fa0/1 |
| 4 | R2 | G0/0/0 | SW-B | Fa0/1 |
| 5 | SW-A | Fa0/2 | PC-A1 | Fa0 |
| 6 | SW-A | Fa0/3 | PC-A2 | Fa0 |
| 7 | SW-B | Fa0/2 | PC-B1 | Fa0 |
| 8 | SW-B | Fa0/3 | PC-B2 | Fa0 |

> Note: The ISR4331 in Packet Tracer uses G0/0/0 for the first interface and G0/0/1 for the second — different from the ISR4221/2911 naming convention. Verify interface assignments with `show ip interface brief` before applying crypto maps.

---

## Key configurations

### ISP router

```
interface GigabitEthernet0/0/0
 ip address 203.0.113.1 255.255.255.252
 description TO_R1
 no shutdown

interface GigabitEthernet0/0/1
 ip address 203.0.113.5 255.255.255.252
 description TO_R2
 no shutdown
```

### R1 — basic interfaces and routing

```
interface GigabitEthernet0/0/0
 ip address 10.1.0.1 255.255.255.0
 description LAN_SITE_A
 no shutdown

interface GigabitEthernet0/0/1
 ip address 203.0.113.2 255.255.255.252
 description WAN_TO_ISP
 no shutdown

ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### R2 — basic interfaces and routing

```
interface GigabitEthernet0/0/0
 ip address 10.2.0.1 255.255.255.0
 description LAN_SITE_B
 no shutdown

interface GigabitEthernet0/0/1
 ip address 203.0.113.6 255.255.255.252
 description WAN_TO_ISP
 no shutdown

ip route 0.0.0.0 0.0.0.0 203.0.113.5
```

### Phase 1 — ISAKMP policy (identical on both routers)

```
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
```

### Pre-shared key — peer address must point at the REMOTE router

```
! On R1 — points at R2's WAN IP
crypto isakmp key VPNKEY123 address 203.0.113.6

! On R2 — points at R1's WAN IP
crypto isakmp key VPNKEY123 address 203.0.113.2
```

### Phase 2 — transform set (identical on both routers)

```
crypto ipsec transform-set VPN_TS esp-aes 256 esp-sha-hmac
 mode tunnel
```

### Interesting traffic ACL — mirrored on each router

```
! On R1 — traffic from Site A to Site B
ip access-list extended VPN_TRAFFIC
 permit ip 10.1.0.0 0.0.0.255 10.2.0.0 0.0.0.255

! On R2 — mirror image: traffic from Site B to Site A
ip access-list extended VPN_TRAFFIC
 permit ip 10.2.0.0 0.0.0.255 10.1.0.0 0.0.0.255
```

### Crypto map — applied to WAN interface on both routers

```
! On R1
crypto map VPN_MAP 10 ipsec-isakmp
 set peer 203.0.113.6
 set transform-set VPN_TS
 match address VPN_TRAFFIC

interface GigabitEthernet0/0/1
 crypto map VPN_MAP

! On R2
crypto map VPN_MAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set VPN_TS
 match address VPN_TRAFFIC

interface GigabitEthernet0/0/1
 crypto map VPN_MAP
```

---

## Verification

```
! Trigger the tunnel — first ping drops while IKE negotiates, second should succeed
! Run from PC-A1 command prompt
ping 10.2.0.10

! Phase 1 verification — look for QM_IDLE state
show crypto isakmp sa

! Phase 2 verification — look for pkts encaps and pkts decaps both non-zero
show crypto ipsec sa

! Confirm crypto map is applied to the correct interface
show crypto map

! Confirm interesting traffic ACL is matching
show ip access-lists VPN_TRAFFIC

! Confirm routing is correct before troubleshooting crypto
show ip route
ping 203.0.113.6
```

### What to look for

| Check | Expected result |
|-------|----------------|
| `show crypto isakmp sa` | `QM_IDLE` — Phase 1 fully negotiated |
| `show crypto ipsec sa` | `pkts encaps` and `pkts decaps` both non-zero |
| `show crypto map` | Crypto map listed under correct WAN interface |
| PC-A1 ping PC-B1 | Second ping succeeds after first drops |
| Simulation mode on WAN link | Packet type shows ESP, not ICMP |

### ISAKMP state reference — what each state means

| State | Meaning |
|-------|---------|
| `MM_NO_STATE` | No negotiation started — routing or crypto map issue |
| `MM_KEY_EXCH` | Stalled at key exchange — ISAKMP policy mismatch |
| `MM_SA_SETUP` | Stalled at authentication — pre-shared key mismatch |
| `QM_IDLE` | Phase 1 complete — tunnel is up |

---

## Concepts demonstrated

| Concept | IOS command |
|---------|-------------|
| IKEv1 Phase 1 policy | `crypto isakmp policy` |
| Pre-shared key authentication | `crypto isakmp key` |
| Phase 2 transform set | `crypto ipsec transform-set` |
| Interesting traffic definition | `ip access-list extended` |
| Crypto map | `crypto map ipsec-isakmp` |
| Applying crypto map | `interface ... crypto map` |
| Phase 1 verification | `show crypto isakmp sa` |
| Phase 2 verification | `show crypto ipsec sa` |
| Clearing stale SAs | `clear crypto isakmp`, `clear crypto sa` |

---

## Lessons learned — troubleshooting log

This lab required diagnosing three separate IPsec misconfigurations, each caught by reading ISAKMP state output rather than guessing. The progression from `MM_KEY_EXCH` to `MM_SA_SETUP` to `QM_IDLE` reflects each fix taking effect.

---

### Issue 1 — Empty ISAKMP table, no negotiation starting

**Symptom:** `show crypto isakmp sa` returned a completely empty table after triggering interesting traffic.

**Diagnostic:** Ran `show crypto map` to confirm the crypto map was applied. Output showed the map was applied to `GigabitEthernet0/0/1` — which is the correct WAN interface on the ISR4331. However, `show running-config | section isakmp` revealed:

```
crypto isakmp key VPNKEY123 address 203.0.113.2
```

R1 was pointing its pre-shared key at **its own WAN IP** rather than the remote peer's WAN IP.

**Fix:**
```
no crypto isakmp key VPNKEY123 address 203.0.113.2
crypto isakmp key VPNKEY123 address 203.0.113.6
```

**Key takeaway:** The pre-shared key command requires the **remote peer's** IP address, not the local router's own IP. The command compiles without error either way, making this mistake easy to introduce and hard to spot without reading the config carefully.

---

### Issue 2 — R2 pre-shared key pointing at wrong peer

**Symptom:** After fixing R1, the ISAKMP state progressed to `MM_SA_SETUP (deleted)` — further along but still failing at authentication.

**Diagnostic:** Ran `show running-config | section crypto` on R2 and found:

```
crypto isakmp key VPNKEY123 address 203.0.113.6
```

R2 was pointing its pre-shared key at **its own WAN IP** (`203.0.113.6`) instead of R1's WAN IP (`203.0.113.2`). The same mistake as Issue 1, mirrored on the other router.

**Fix:**
```
no crypto isakmp key VPNKEY123 address 203.0.113.6
crypto isakmp key VPNKEY123 address 203.0.113.2
```

**Key takeaway:** On a site-to-site VPN, both routers must have their pre-shared key commands pointing at each other. It is easy to configure both routers pointing at themselves, or both pointing at the same IP — neither produces an error, both break authentication silently.

---

### Issue 3 — Pre-shared key string mismatch

**Symptom:** After fixing both peer address issues, ISAKMP state still showed `MM_SA_SETUP (deleted)` — stalling at authentication despite both routers now pointing at the correct peers.

**Root cause:** `MM_SA_SETUP` specifically indicates an authentication failure. With peer addresses now correct, the only remaining cause is the key strings not matching between the two routers. A character-level mismatch in the key string was found on closer inspection.

**Fix:** Re-entered the pre-shared key on both routers with the correct identical string and cleared the stale SAs:
```
clear crypto isakmp
clear crypto sa
```

**Key takeaway:** Pre-shared key strings are case-sensitive and must be byte-for-byte identical on both routers. IOS accepts any key string without validation. The ISAKMP state field is the only diagnostic tool available — `MM_SA_SETUP` means authentication failed, which means the key strings do not match. There is no more specific error message.

---

### Note on interface naming — ISR4331 vs older models

The ISR4331 in Packet Tracer uses `GigabitEthernet0/0/0` and `GigabitEthernet0/0/1` for its first two interfaces. Older models (2911, 1941) use `GigabitEthernet0/0` and `GigabitEthernet0/1`. Always verify with `show ip interface brief` before applying a crypto map — applying it to the wrong interface produces no error but the tunnel will never form.

---

## Files

| File | Description |
|------|-------------|
| `lab06.pkt` | Packet Tracer source file (PT 8.x) |
| `topology.png` | Network diagram |
| `screenshots/isakmp-sa.png` | `show crypto isakmp sa` showing QM_IDLE |
| `screenshots/ipsec-sa.png` | `show crypto ipsec sa` showing encaps/decaps counters |
| `configs/R1.txt` | Final running configuration |
| `configs/R2.txt` | Final running configuration |
| `configs/ISP.txt` | Final running configuration |

---

## Skills demonstrated

`IKEv1 IPsec` · `ISAKMP policy` · `Pre-shared key authentication` · `ESP encryption` · `Crypto maps` · `Interesting traffic ACL` · `Phase 1 and Phase 2 negotiation` · `ISAKMP state diagnosis` · `VPN troubleshooting` · `Simulation mode verification` · `ISR4331 interface naming`
