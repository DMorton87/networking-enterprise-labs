# Lab 04 — NAT/PAT with DMZ Architecture


## Objectives

- Design and implement a three-zone network: Internet, DMZ, and trusted LAN
- Configure PAT overload to allow LAN clients to share a single public IP for outbound internet access
- Configure static NAT to expose a DMZ web server to the internet via a dedicated public IP
- Set up a DHCP pool on the perimeter router serving LAN clients
- Verify NAT translations using `show ip nat translations`
- Test inbound and outbound connectivity from both sides of the NAT boundary
- Understand why the DMZ exists as a security boundary between untrusted and trusted zones

---

## The design problem this lab solves

A company has a small public IP block from their ISP. They need to:
- Let internal LAN users browse the internet — many private IPs sharing one public IP
- Host a public-facing web server reachable directly from the internet
- Keep the web server isolated from the trusted LAN so that a compromise does not expose internal hosts

This is the architecture behind almost every small-to-mid enterprise network ever built. The DMZ (demilitarised zone) sits between the internet and the internal LAN, hosting services that need to be publicly reachable without being trusted with access to internal systems.

---

## Topology

```
                        [ Internet / ISP ]
                         ISP router
                         203.0.113.1/29
                         External PC
                         203.0.113.5/29
                               |
                         203.0.113.0/29 (WAN)
                               |
                    [ Perimeter router ]
                    Outside: 203.0.113.2/29
                    DMZ:     192.168.2.1/24
                    LAN:     192.168.1.1/24
                        /              \
              [ DMZ — semi-trusted ]   [ LAN — trusted ]
              Web server 192.168.2.10  PC1–PC4 (DHCP)
              DNS server 192.168.2.20  192.168.1.10–.13
```

---

## Three zones, three security levels

| Zone | Subnet | Trust level | NAT type |
|------|--------|-------------|----------|
| Internet / ISP | 203.0.113.0/29 | Untrusted | — |
| DMZ | 192.168.2.0/24 | Semi-trusted | Static NAT (1-to-1) |
| LAN | 192.168.1.0/24 | Trusted | PAT overload (many-to-1) |

All inter-zone traffic must pass through the perimeter router. In a production environment, ACLs or a stateful firewall would enforce zone-to-zone policy — for example, blocking DMZ-initiated connections toward the LAN. That ACL work is the focus of Lab 03.

---

## Device list

| Device | PT model | Role |
|--------|----------|------|
| Perimeter | ISR4331 | NAT/PAT gateway, DHCP server, default route |
| ISP | ISR4221 | Simulates ISP and upstream internet |
| SW-DMZ | 2960-24TT | DMZ access switch |
| SW-LAN | 2960-24TT | LAN access switch |
| Web server | Server-PT | Public-facing HTTP server in DMZ |
| DNS server | Server-PT | DNS server in DMZ |
| PC1–PC4 | PC-PT | LAN clients, DHCP |
| External PC | PC-PT | Simulates internet user testing inbound NAT |

---

## IP addressing

| Device | Interface | Address | Purpose |
|--------|-----------|---------|---------|
| ISP | G0/0 | 203.0.113.1/29 | WAN link toward perimeter |
| ISP | G0/1 | 203.0.113.5/29 | Link to external PC |
| Perimeter | G0/0 | 203.0.113.2/29 | Outside — NAT outside |
| Perimeter | G0/1 | 192.168.2.1/24 | DMZ gateway — NAT inside |
| Perimeter | G0/2 | 192.168.1.1/24 | LAN gateway — NAT inside |
| Web server | Fa0 | 192.168.2.10/24 | Static NAT inside address |
| DNS server | Fa0 | 192.168.2.20/24 | DMZ host |
| PC1–PC4 | Fa0 | DHCP .10–.13 | LAN clients |
| External PC | Fa0 | 203.0.113.5/29 | Test machine on simulated internet |
| Public web server IP | — | 203.0.113.3 | Static NAT outside address |

> 203.0.113.0/24 is RFC 5737 documentation-reserved space — safe to use for simulating public IPs in labs without risk of conflicting with real internet addresses.

---

## Cabling

| Cable | From | Interface | To | Interface |
|-------|------|-----------|----|-----------|
| 1 | ISP | G0/0 | Perimeter | G0/0 |
| 2 | ISP | G0/1 | External PC | Fa0 |
| 3 | Perimeter | G0/1 | SW-DMZ | Fa0/1 |
| 4 | Perimeter | G0/2 | SW-LAN | Fa0/1 |
| 5 | SW-DMZ | Fa0/2 | Web server | Fa0 |
| 6 | SW-DMZ | Fa0/3 | DNS server | Fa0 |
| 7–10 | SW-LAN | Fa0/2–Fa0/5 | PC1–PC4 | Fa0 |

---

## Key configurations

### ISP router

```
hostname ISP

interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.248
 description TO_PERIMETER
 no shutdown

interface GigabitEthernet0/1
 ip address 203.0.113.5 255.255.255.248
 description TO_EXTERNAL_PC
 no shutdown

! Route so ISP can forward traffic destined for the public web server IP
ip route 203.0.113.3 255.255.255.255 203.0.113.2
```

### Perimeter router

```
hostname Perimeter

! WAN interface — must be marked NAT outside
interface GigabitEthernet0/0
 ip address 203.0.113.2 255.255.255.248
 description WAN_OUTSIDE
 ip nat outside
 no shutdown

! DMZ interface — NAT inside
interface GigabitEthernet0/1
 ip address 192.168.2.1 255.255.255.0
 description DMZ
 ip nat inside
 no shutdown

! LAN interface — NAT inside
interface GigabitEthernet0/2
 ip address 192.168.1.1 255.255.255.0
 description LAN_TRUSTED
 ip nat inside
 no shutdown

! Default route — all unmatched traffic exits toward ISP
ip route 0.0.0.0 0.0.0.0 203.0.113.1

! DHCP pool for LAN clients
ip dhcp excluded-address 192.168.1.1 192.168.1.9
ip dhcp pool LAN
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.1
 dns-server 192.168.2.20

! Static NAT — permanent 1-to-1 mapping for the web server
! Internet traffic arriving at 203.0.113.3 is always forwarded to 192.168.2.10
ip nat inside source static 192.168.2.10 203.0.113.3

! PAT overload — LAN clients share the outside interface IP
! ACL defines which source addresses are eligible for translation
ip access-list standard LAN_HOSTS
 permit 192.168.1.0 0.0.0.255

ip nat inside source list LAN_HOSTS interface GigabitEthernet0/0 overload
```

**Why two different NAT types?** The web server needs a predictable, permanent public address so DNS records can point to it — that is static NAT. LAN users only need outbound internet access and do not need to be individually reachable from outside — that is PAT overload, which conserves public IPs by multiplexing thousands of sessions onto one address, using port numbers to track each individual flow.

### Web server

- IP: `192.168.2.10` — Subnet: `255.255.255.0` — Gateway: `192.168.2.1`
- Services tab → HTTP → **On**

### External PC

- IP: `203.0.113.5` — Subnet: `255.255.255.248` — Gateway: `203.0.113.1`

---

## Verification

```
! Confirm NAT inside/outside assignments are correct
show ip interface brief

! View all active NAT translations
! Static NAT entry for web server should always be present
! PAT entries appear after LAN clients generate traffic
show ip nat translations

! Confirm hit counts on NAT rules — useful for proving traffic is flowing
show ip nat statistics

! From PC1 command prompt — tests PAT outbound
ping 203.0.113.1

! From External PC web browser — tests static NAT inbound
! Type the full URL including protocol prefix
http://203.0.113.3
```

### What to look for in `show ip nat translations`

Static NAT entry (always present):
```
---  203.0.113.3      192.168.2.10    ---    ---
```

PAT entry (appears after a LAN client generates traffic):
```
icmp 203.0.113.2:1024  192.168.1.10:1024  203.0.113.1:1024  203.0.113.1:1024
```

TCP entry (appears when External PC loads the web page):
```
tcp  203.0.113.3:80    192.168.2.10:80    203.0.113.5:xxxx  203.0.113.5:xxxx
```

---

## Concepts demonstrated

| Concept | IOS command |
|---------|-------------|
| NAT outside interface | `ip nat outside` |
| NAT inside interface | `ip nat inside` |
| Static NAT | `ip nat inside source static <inside> <outside>` |
| PAT overload | `ip nat inside source list <acl> interface <int> overload` |
| NAT verification | `show ip nat translations` |
| NAT statistics | `show ip nat statistics` |
| Default route | `ip route 0.0.0.0 0.0.0.0 <next-hop>` |
| DHCP pool | `ip dhcp pool`, `ip dhcp excluded-address` |

---

## Lessons learned — troubleshooting log

This section documents a real issue encountered during the lab build and the diagnostic process used to resolve it. This kind of systematic troubleshooting is as valuable to demonstrate as the configuration itself.

### Issue: External PC could ping the web server but could not load the web page

**Symptom:** Ping from External PC to `203.0.113.3` succeeded. Opening `http://203.0.113.3` in the PT web browser failed silently.

**Initial suspicion:** DNS misconfiguration.

**Why DNS was ruled out:** The browser was using a direct IP address, not a hostname, so DNS is never consulted. A failure at that point has to be happening at the transport or network layer.

**Diagnostic step 1 — check NAT translations:**
```
show ip nat translations
```
Only the static NAT entry was present. No TCP port 80 translation appeared even while the browser was attempting to connect. This indicated the TCP SYN was not completing the NAT translation, meaning the packet was being dropped somewhere before NAT could process the return.

**Diagnostic step 2 — simulation mode:**
Ran the HTTP attempt in Packet Tracer simulation mode and traced the packet hop by hop. The TCP packet arrived at the ISP router and was dropped there. The ISP router reported:

```
The packet is a non-ICMP packet to the outgoing port's subnet ID or broadcast.
The device drops the packet.
```

**Root cause — subnet boundary collision:**
The WAN link was originally configured as a `/30`, giving only two usable host addresses:
- `203.0.113.1` — ISP router
- `203.0.113.2` — Perimeter outside interface
- `203.0.113.3` — **broadcast address of the /30 subnet**

The static NAT public IP `203.0.113.3` had been assigned without accounting for the fact that it was the broadcast address of the WAN subnet. IOS correctly dropped non-ICMP packets destined for a subnet broadcast address. ICMP (ping) was handled more permissively, which is why ping succeeded while TCP failed — a misleading symptom that could easily send troubleshooting in the wrong direction.

**Why ping worked but TCP did not:**
Cisco IOS handles ICMP differently from TCP at the subnet boundary check. This is a known behavioural difference and a useful reminder that a successful ping does not guarantee TCP connectivity — they are different protocols and can be treated differently at each hop.

**Resolution — resize the WAN subnet from /30 to /29:**
A `/29` provides six usable host addresses, giving enough room to assign a dedicated public IP for static NAT while keeping the ISP and perimeter router addresses in the same subnet:

| Address | Role |
|---------|------|
| 203.0.113.1 | ISP router |
| 203.0.113.2 | Perimeter outside interface |
| 203.0.113.3 | Static NAT public IP for web server |
| 203.0.113.4 | Reserved for future use |
| 203.0.113.5 | External PC |
| 203.0.113.6 | Spare |

Updated subnet mask `255.255.255.248` on ISP G0/0, ISP G0/1, Perimeter G0/0, and External PC. HTTP access confirmed working immediately after.

**Key takeaway:** Always verify that your NAT outside addresses are valid host addresses within the WAN subnet — not the network address, not the broadcast address. When ping succeeds but TCP fails, trace at the packet level rather than assuming the path is clear. Packet Tracer simulation mode is invaluable for isolating exactly which hop is dropping a packet and why.

---

## Files

| File | Description |
|------|-------------|
| `lab04.pkt` | Packet Tracer source file (PT 8.x) |
| `topology.png` | Network diagram |
| `configs/Perimeter.txt` | Final running configuration |
| `configs/ISP.txt` | Final running configuration |

---

## Skills demonstrated

`Static NAT` · `PAT overload` · `DMZ architecture` · `Three-zone security model` · `NAT inside/outside` · `Default routing` · `DHCP` · `Subnet design` · `IOS NAT verification` · `Simulation mode troubleshooting`
