# Lab 05 — EtherChannel & STP Hardening with Centralised DHCP

A hands-on Cisco Packet Tracer lab implementing LACP EtherChannel bundles across a three-switch triangle topology, Rapid PVST+ with forced root bridge election, STP hardening features on all access and inter-switch ports, and centralised DHCP relay across two VLANs via a Layer 3 switch acting as a collapsed core.

---

## Overview

This lab demonstrates a **collapsed core** architecture — a single Layer 3 switch (SW1) handles inter-VLAN routing, STP root bridge duties, and DHCP relay for the entire network. SW2 and SW3 are Layer 2 access switches connected to SW1 and to each other via LACP EtherChannel bundles, forming a triangle with full redundancy at every link.

Without STP, the triangle topology would create a Layer 2 broadcast storm instantly. EtherChannel presents each bundle as a single logical link to STP, preserving redundancy while keeping the topology loop-free. STP hardening features then protect the stable topology from disruption by rogue devices or misconfiguration.

---

## Objectives

- Configure LACP EtherChannel bundles between all three switch pairs
- Enable Rapid PVST+ and force SW1 as root bridge for all VLANs via priority manipulation
- Apply root guard on the SW2-SW3 inter-switch link to prevent either switch claiming root
- Apply PortFast and BPDU guard on all access ports facing end hosts
- Configure SW1 SVIs for inter-VLAN routing between VLAN 10 and VLAN 20
- Deploy a centralised DHCP server on VLAN 99 and configure `ip helper-address` relay on all client-facing SVIs
- Verify EtherChannel negotiation, STP topology, and end-to-end DHCP reachability across VLANs

---

## Topology

```
                         [ SW1 — L3 switch ]
                         Root bridge — priority 4096
                         SVIs: VLAN 10, 20, 99
                         ip helper-address 10.99.99.10
                              |
                         Fa0/1 — VLAN 99
                              |
                       [ DHCP server ]
                        10.99.99.10/24

          Po1 (Fa0/1+Fa0/2)        Po2 (Fa0/3+Fa0/4)
         /                                            \
   [ SW2 ]                                        [ SW3 ]
   priority 8192                                  priority 8192
   VLAN 10 + 20                                   VLAN 10 + 20
      |    |                Po3 (Gi0/1+Gi0/2)        |    |
    PC1   PC2 ─────────────────────────────────── PC3   PC4
   VLAN10 VLAN20                                VLAN10 VLAN20
```

---

## Device list

| Device | PT model | Role |
|--------|----------|------|
| SW1 | 3560-24PS | Layer 3 switch — root bridge, SVIs, DHCP relay |
| SW2 | 2960-24TT | Access switch — VLAN 10 + 20 |
| SW3 | 2960-24TT | Access switch — VLAN 10 + 20 |
| DHCP server | Server-PT | Centralised DHCP for VLAN 10 and 20 |
| PC1, PC3 | PC-PT | VLAN 10 DHCP clients |
| PC2, PC4 | PC-PT | VLAN 20 DHCP clients |

---

## VLAN definitions

| VLAN ID | Name | Purpose |
|---------|------|---------|
| 10 | SALES | End user workstations |
| 20 | MARKETING | End user workstations |
| 99 | MGMT_DHCP | Management and centralised DHCP server |

---

## IP addressing

| Device / SVI | Address | VLAN |
|--------------|---------|------|
| SW1 Vlan10 | 10.10.10.1/24 | 10 |
| SW1 Vlan20 | 10.20.20.1/24 | 20 |
| SW1 Vlan99 | 10.99.99.1/24 | 99 |
| DHCP server | 10.99.99.10/24 | 99 |
| PC1, PC3 | DHCP — 10.10.10.10+ | 10 |
| PC2, PC4 | DHCP — 10.20.20.10+ | 20 |

---

## Cabling

| Cable | From | Interfaces | To | Interfaces | Type |
|-------|------|-----------|-----|-----------|------|
| EtherChannel 1 | SW1 | Fa0/1, Fa0/2 | SW2 | Fa0/1, Fa0/2 | Trunk |
| EtherChannel 2 | SW1 | Fa0/3, Fa0/4 | SW3 | Fa0/1, Fa0/2 | Trunk |
| EtherChannel 3 | SW2 | Gi0/1, Gi0/2 | SW3 | Gi0/1, Gi0/2 | Trunk |
| DHCP server | SW1 | Fa0/5 | DHCP server | Fa0 | Access VLAN 99 |
| PC1 | SW2 | Fa0/3 | PC1 | Fa0 | Access VLAN 10 |
| PC2 | SW2 | Fa0/4 | PC2 | Fa0 | Access VLAN 20 |
| PC3 | SW3 | Fa0/3 | PC3 | Fa0 | Access VLAN 10 |
| PC4 | SW3 | Fa0/4 | PC4 | Fa0 | Access VLAN 20 |

---

## Key configurations

### Step 1 — VLANs (all three switches, identical)

```
vlan 10
 name SALES
vlan 20
 name MARKETING
vlan 99
 name MGMT_DHCP
```

### Step 2 — EtherChannel on SW1 (3560-24PS)

```
! Toward SW2 — FastEthernet bundle
interface range FastEthernet0/1-2
 switchport trunk encapsulation dot1q
 channel-group 1 mode active
 channel-protocol lacp

! Toward SW3 — FastEthernet bundle
interface range FastEthernet0/3-4
 switchport trunk encapsulation dot1q
 channel-group 2 mode active
 channel-protocol lacp

interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99

interface Port-channel2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99

! Access port for DHCP server on VLAN 99
interface FastEthernet0/5
 switchport mode access
 switchport access vlan 99
```

### Step 3 — EtherChannel on SW2 (2960-24TT)

```
! Uplink toward SW1 — FastEthernet bundle
interface range FastEthernet0/1-2
 channel-group 1 mode active
 channel-protocol lacp

! Cross-link toward SW3 — GigabitEthernet bundle
interface range GigabitEthernet0/1-2
 channel-group 3 mode active
 channel-protocol lacp

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99

interface Port-channel3
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99

! Access ports toward PCs
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/4
 switchport mode access
 switchport access vlan 20
```

### Step 4 — EtherChannel on SW3 (2960-24TT)

```
! Uplink toward SW1 — FastEthernet bundle
interface range FastEthernet0/1-2
 channel-group 2 mode active
 channel-protocol lacp

! Cross-link toward SW2 — GigabitEthernet bundle
interface range GigabitEthernet0/1-2
 channel-group 3 mode active
 channel-protocol lacp

interface Port-channel2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99

interface Port-channel3
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99

! Access ports toward PCs
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/4
 switchport mode access
 switchport access vlan 20
```

### Step 5 — SW1 SVIs, routing, and DHCP relay

```
! Enable Layer 3 routing
ip routing

interface Vlan10
 ip address 10.10.10.1 255.255.255.0
 ip helper-address 10.99.99.10
 no shutdown

interface Vlan20
 ip address 10.20.20.1 255.255.255.0
 ip helper-address 10.99.99.10
 no shutdown

interface Vlan99
 ip address 10.99.99.1 255.255.255.0
 no shutdown
```

### Step 6 — Rapid PVST+ and root bridge election

```
! On SW1 — forced root for all VLANs
spanning-tree mode rapid-pvst
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 4096
spanning-tree vlan 99 priority 4096

! On SW2 and SW3 — mid-range priority, never accidentally elected root
spanning-tree mode rapid-pvst
spanning-tree vlan 10 priority 8192
spanning-tree vlan 20 priority 8192
spanning-tree vlan 99 priority 8192
```

### Step 7 — STP hardening on SW2 and SW3

```
! Root guard on the SW2-SW3 GigabitEthernet cross-link
! Prevents either access switch from claiming root relative to the other
interface Port-channel3
 spanning-tree guard root

! PortFast — access ports skip listening and learning, go straight to forwarding
! BPDU guard — port goes err-disabled immediately if a BPDU is received
! Protects against rogue switches being plugged into end-user ports
interface FastEthernet0/3
 spanning-tree portfast
 spanning-tree bpduguard enable

interface FastEthernet0/4
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### Step 8 — DHCP server pools

Configure in Services → DHCP on the Server-PT:

| Pool name | Gateway | Start IP | Mask | DNS |
|-----------|---------|----------|------|-----|
| VLAN10_SALES | 10.10.10.1 | 10.10.10.10 | 255.255.255.0 | 8.8.8.8 |
| VLAN20_MARKETING | 10.20.20.1 | 10.20.20.10 | 255.255.255.0 | 8.8.8.8 |

---

## Verification commands

```
! SW1 — confirm root bridge for all VLANs
! Look for 'This bridge is the root'
show spanning-tree vlan 10
show spanning-tree vlan 20

! All switches — confirm EtherChannel bundles
! Member ports should show 'P' (bundled), port-channels show 'SU'
show etherchannel summary

! Confirm LACP negotiation with neighbours
show lacp neighbor

! SW1 — confirm SVIs are up and routing table is populated
show ip interface brief
show ip route

! SW1 — confirm DHCP relay is active on client SVIs
show ip interface Vlan10 | include Helper
show ip interface Vlan20 | include Helper

! SW2 / SW3 — confirm root guard on Po3 and PortFast on access ports
show spanning-tree interface Port-channel3 detail
show spanning-tree interface FastEthernet0/3 detail

! PCs — confirm DHCP lease and correct pool
ipconfig /all

! Inter-VLAN test — PC1 (VLAN 10) to PC2 (VLAN 20)
ping 10.20.20.10

! Same-VLAN cross-switch test — PC1 to PC3 (both VLAN 10)
ping 10.10.10.11
```

### What to look for

| Check | Expected result |
|-------|----------------|
| `show spanning-tree vlan 10` on SW1 | "This bridge is the root" |
| `show etherchannel summary` | Po1, Po2, Po3 all show `SU`; members show `P` |
| `show ip route` on SW1 | Connected routes for all three subnets |
| PC1 and PC3 | DHCP lease from 10.10.10.0/24 pool |
| PC2 and PC4 | DHCP lease from 10.20.20.0/24 pool |
| PC1 → PC2 ping | Success — inter-VLAN routing via SW1 SVIs |
| PC1 → PC3 ping | Success — same VLAN across the triangle |
| BPDU on access port | Port goes `err-disabled` |

---

## Concepts demonstrated

| Concept | IOS command |
|---------|-------------|
| LACP EtherChannel | `channel-group X mode active`, `channel-protocol lacp` |
| Trunk encapsulation (3560) | `switchport trunk encapsulation dot1q` |
| Rapid PVST+ | `spanning-tree mode rapid-pvst` |
| Root bridge election | `spanning-tree vlan X priority 4096` |
| Root guard | `spanning-tree guard root` |
| PortFast | `spanning-tree portfast` |
| BPDU guard | `spanning-tree bpduguard enable` |
| Layer 3 switching | `ip routing`, `interface VlanX` |
| SVI inter-VLAN routing | `ip address` on Vlan interfaces |
| DHCP relay | `ip helper-address` |
| EtherChannel verification | `show etherchannel summary`, `show lacp neighbor` |

---

## Lessons learned

### Issue — `switchport mode trunk` rejected on 3560 port-channel interfaces

**Symptom:**
```
Command rejected: An interface whose trunk encapsulation is "Auto" can not be
configured to "trunk" mode.
```

**Root cause:** The 3560-24PS in Packet Tracer requires explicit trunk encapsulation to be set before the port mode can be changed. On a 2960, dot1q is the only encapsulation option so it is implicit. On the 3560, the encapsulation defaults to `Auto` and must be declared manually.

**Fix:** Add `switchport trunk encapsulation dot1q` before `switchport mode trunk` on all 3560 interfaces and port-channels:
```
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

**Key takeaway:** This requirement is specific to multilayer switches in Packet Tracer. It does not appear on Layer 2 only switches. Any time a trunk configuration is rejected on a 3560 or 3650 in PT, this is the first thing to check. The error message does not make the cause immediately obvious, which makes it a common lab stumbling block.

---

## Architecture note — collapsed core

This topology implements a collapsed core design — SW1 simultaneously acts as the distribution layer (inter-VLAN routing, DHCP relay) and the core layer (root bridge, EtherChannel hub). This is common in small-to-mid campus networks where separate core and distribution layers would add cost and complexity without meaningful benefit at the scale involved. In a larger network the roles would be separated across dedicated devices, with distribution switches handling per-VLAN STP root and inter-VLAN routing, and a separate core layer handling high-speed transit between distribution blocks.

---

## Files

| File | Description |
|------|-------------|
| `lab05.pkt` | Packet Tracer source file (PT 8.x) |
| `topology.png` | Network diagram |
| `configs/SW1.txt` | Final running configuration |
| `configs/SW2.txt` | Final running configuration |
| `configs/SW3.txt` | Final running configuration |

---

## Skills demonstrated

`LACP EtherChannel` · `Rapid PVST+` · `Root bridge election` · `Root guard` · `BPDU guard` · `PortFast` · `Layer 3 switching` · `SVI inter-VLAN routing` · `DHCP relay` · `ip helper-address` · `Collapsed core design` · `Trunk encapsulation` · `EtherChannel verification`
