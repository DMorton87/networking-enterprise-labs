# Lab 02 — OSPF Multi-Area with Centralised DHCP Relay

A hands-on Cisco Packet Tracer lab practising multi-area OSPFv2 design, ABR route summarisation, external route redistribution, and centralised DHCP relay across routed segments.

---

## Objectives

- Configure OSPFv2 across three areas: Area 0 (backbone), Area 1, and Area 2
- Assign router IDs manually and verify full OSPF adjacency formation
- Apply passive interfaces on all LAN-facing ports
- Summarise each area's routes at the ABR using `area range`
- Deploy a centralised DHCP server and configure `ip helper-address` relay on R2 and R3
- Verify end-to-end reachability across all areas using ping

---

## Topology

```
                        [ Area 1 ]                  [ Area 0 ]              [ Area 2 ]

  DHCP Server ──┐
  PC1, PC2 ─────┤                         10.0.0.0/30
                SW1 ── R2 ── ABR1 ──────────────────── ABR2 ── R3 ── SW2 ── PC3, PC4
                       |      |                          |      |
                  G0/1 |  G0/2|                      G0/1|  G0/1|
                       |      |                          |      |
                  R2–ABR1     R1–ABR1             ABR2–R3     LAN 2
                 10.1.2.0/30  10.1.1.0/30        10.2.1.0/30  10.2.0.0/24
```


## IP Addressing

| Device | Interface | IP Address | Subnet | Purpose |
|--------|-----------|------------|--------|---------|
| ABR1 | G0/0 | 10.0.0.1 | /30 | Backbone link to ABR2 |
| ABR1 | G0/1 | 10.1.1.2 | /30 | Link to R1 |
| ABR1 | G0/2 | 10.1.2.2 | /30 | Link to R2 |
| ABR1 | Lo0 | 1.1.1.1 | /32 | Router ID |
| ABR2 | G0/0 | 10.0.0.2 | /30 | Backbone link to ABR1 |
| ABR2 | G0/1 | 10.2.1.2 | /30 | Link to R3 |
| ABR2 | Lo0 | 4.4.4.4 | /32 | Router ID |
| ABR2 | Lo1 | 172.16.1.1 | /16 | Simulated external network |
| R1 | G0/0 | 10.1.1.1 | /30 | Link to ABR1 |
| R1 | Lo0 | 2.2.2.2 | /32 | Router ID |
| R2 | G0/0 | 10.1.2.1 | /30 | Link to ABR1 |
| R2 | G0/1 | 10.1.0.1 | /24 | LAN gateway — Area 1 |
| R2 | Lo0 | 3.3.3.3 | /32 | Router ID |
| R3 | G0/0 | 10.2.1.1 | /30 | Link to ABR2 |
| R3 | G0/1 | 10.2.0.1 | /24 | LAN gateway — Area 2 |
| R3 | Lo0 | 5.5.5.5 | /32 | Router ID |
| DHCP Server | Fa0 | 10.1.0.2 | /24 | Static — Area 1 LAN |
| PC1 | Fa0 | DHCP | /24 | Area 1 client |
| PC2 | Fa0 | DHCP | /24 | Area 1 client |
| PC3 | Fa0 | DHCP | /24 | Area 2 client |
| PC4 | Fa0 | DHCP | /24 | Area 2 client |

---

## Key Configurations

### OSPF — multi-area with summarisation (ABR1 example)

```
router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.3 area 0
 network 1.1.1.1 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.3 area 1
 network 10.1.2.0 0.0.0.3 area 1
 area 1 range 10.1.0.0 255.255.0.0
```

`area 1 range` summarises all Area 1 prefixes into a single `10.1.0.0/16` advertisement injected into Area 0, reducing LSA flooding across the backbone.

### Passive interface — LAN-facing ports (R2 example)

```
router ospf 1
 passive-interface GigabitEthernet0/1
```

Prevents OSPF Hello packets being sent toward end hosts. End hosts cannot form OSPF adjacencies, so sending Hellos is both wasteful and a minor security exposure.


### DHCP — centralised relay

A single DHCP server at `10.1.0.2` serves both Area 1 and Area 2 LANs. R2 and R3 act as relay agents, converting client broadcasts into unicast packets forwarded across the OSPF network to the server.

**Server pools (configured via Services > DHCP in Packet Tracer):**

| Pool Name | Default Gateway | Start IP | Subnet Mask |
|-----------|----------------|----------|-------------|
| AREA1_LAN | 10.1.0.1 | 10.1.0.10 | 255.255.255.0 |
| AREA2_LAN | 10.2.0.1 | 10.2.0.10 | 255.255.255.0 |

**Relay configuration on R2 and R3:**

```
interface GigabitEthernet0/1
 ip helper-address 10.1.0.2
```

---

## Verification Commands

```
! Confirm all adjacencies are in FULL state
show ip ospf neighbor

! Confirm ABR1 and ABR2 participate in two areas each
show ip ospf

! On R1 — expect O IA routes for Area 2 and backbone links, O E2 for 172.16.0.0/16
show ip route ospf


! Confirm passive interface and relay are configured correctly
show ip interface GigabitEthernet0/1

! Confirm LSA types in the database (Type 1, 2, 3, 5)
show ip ospf database

! Confirm DHCP relay is active on LAN interfaces
show ip interface GigabitEthernet0/1 | include Helper

! End-to-end reachability — run from PC1 command prompt
ping 10.2.0.10
ping 10.2.0.11
```

### What to look for

| Check | Expected result |
|-------|----------------|
| `show ip ospf neighbor` | All adjacencies show FULL |
| `show ip route ospf` on R1 | `O IA 10.2.0.0/16`, `O E2 172.16.0.0/16` present |
| `show ip route ospf` on R3 | `O IA 10.1.0.0/16` present |
| `show ip ospf database` | Type 3 Summary LSAs visible at ABRs, Type 5 External LSA for 172.16.0.0 |
| DHCP bindings on server | Four MAC-to-IP bindings — one per PC |
| PC1 `ping 10.2.0.10` | Success — cross-area reachability confirmed |

---

## Concepts Demonstrated

| Concept | IOS Command |
|---------|-------------|
| Multi-area OSPF | `router ospf`, `network ... area` |
| Manual router ID | `router-id` |
| Passive interface | `passive-interface` |
| ABR summarisation | `area X range` |
| DHCP relay agent | `ip helper-address` |

---


## Skills Demonstrated

`OSPFv2` · `Multi-area design` · `ABR / ASBR roles` · `LSA types 1 2 3 5` · `Route summarisation` · `DHCP relay` · `ip helper-address` · `Passive interface` · `IOS verification`
