# Lab 4.5 , ACL Defence Layers with DNS and Split-Horizon Resolution

A hands-on Cisco Packet Tracer lab implementing named extended ACLs across a three-zone network architecture. Builds directly on the NAT/DMZ topology from Lab 04, adding traffic filtering, DNS services, and split-horizon DNS resolution. Includes a detailed troubleshooting log documenting real issues encountered during the build.

---

## Overview

Lab 04 established the NAT/PAT and DMZ architecture. This lab asks the next logical question: now that zones exist, what traffic should actually be allowed to move between them?

ACLs are the answer at Layer 3/4. Three named extended ACLs enforce zone-to-zone policy at the perimeter router, each guarding a different traffic direction. DNS is added as a production-realistic service, which introduces its own filtering challenges , particularly around the stateless nature of UDP and the directional behaviour of the `eq` keyword in IOS ACL syntax.

---

## Objectives

- Implement named extended ACLs on all three perimeter router interfaces
- Understand the directional nature of ACL application (inbound vs outbound)
- Permit only explicitly required traffic, deny everything else
- Configure a DMZ DNS server with split-horizon A records for internal and external clients
- Understand why `established` works for TCP but not UDP, and handle DNS return traffic explicitly
- Recognise the limitations of Layer 3/4 ACLs against application-layer attacks such as DNS tunnelling
- Document and resolve real ACL troubleshooting scenarios encountered during the build

---

## Topology

This lab uses the same physical topology as Lab 04. Refer to that lab's README for device list, IP addressing, cabling, and NAT configuration. The additions in this lab are:

- Three named extended ACLs applied to the perimeter router
- DNS service enabled on the DMZ server at `192.168.2.20`
- Split-horizon DNS A records for `www.ryderkick.local`
- Static NAT entry for the DNS server: `203.0.113.4 → 192.168.2.20`

---

## Three zones, three ACLs

| ACL | Interface | Direction | Purpose |
|-----|-----------|-----------|---------|
| ACL-OUTSIDE | G0/0 | Inbound | Controls what the internet can send into the network |
| ACL-DMZ | G0/1 | Inbound | Prevents DMZ hosts initiating connections toward the LAN |
| ACL-LAN | G0/2 | Outbound | Controls what traffic is delivered to LAN hosts |

The placement of each ACL is deliberate. ACL-OUTSIDE is inbound on the outside interface , it evaluates every packet arriving from the internet before any routing decision. ACL-DMZ is inbound on the DMZ interface , it evaluates traffic leaving the DMZ toward the router. ACL-LAN is outbound on the LAN interface , it acts as a final gate on everything the router is about to deliver to internal hosts.

A critical point: **ACL-OUTSIDE only evaluates inbound traffic on G0/0.** Traffic exiting G0/0 toward the internet is not evaluated by this ACL. This matters for understanding which rules need to handle DNS responses and which do not.

---

## ACL configurations

### ACL-OUTSIDE

```
ip access-list extended ACL-OUTSIDE
 ! Inbound HTTP to the web server's public NAT IP
 permit tcp any host 203.0.113.3 eq www
 ! Inbound HTTPS to the web server
 permit tcp any host 203.0.113.3 eq 443
 ! Inbound DNS queries to the web server's public NAT IP
 permit udp any host 203.0.113.3 eq domain
 ! Inbound DNS queries to the DNS server's public NAT IP
 permit udp any host 203.0.113.4 eq domain
 ! Return traffic for sessions initiated from inside
 permit tcp any any established
 ! ICMP echo-reply so internal pings get responses
 permit icmp any any echo-reply
 deny ip any any

interface GigabitEthernet0/0
 ip access-group ACL-OUTSIDE in
```

### ACL-DMZ

```
ip access-list extended ACL-DMZ
 ! TCP return traffic from DMZ servers to anywhere
 permit tcp 192.168.2.0 0.0.0.255 any established
 ! Explicit permit for web server return traffic toward LAN
 ! Required because PT's established matching can be inconsistent post-NAT
 permit tcp host 192.168.2.10 192.168.1.0 0.0.0.255 established
 ! DNS responses FROM the DNS server (source port 53)
 permit udp host 192.168.2.20 eq domain any
 ! DNS server responses toward LAN specifically
 permit udp host 192.168.2.20 192.168.1.0 0.0.0.255 eq domain
 ! DNS queries from DMZ hosts to upstream resolvers (destination port 53)
 permit udp 192.168.2.0 0.0.0.255 any eq domain
 ! ICMP echo-reply from DMZ
 permit icmp 192.168.2.0 0.0.0.255 any echo-reply
 ! Block DMZ-initiated connections toward the trusted LAN
 ! This is the core DMZ security rule , a compromised server cannot
 ! initiate new connections to internal hosts
 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
 ! Permit remaining DMZ-to-internet traffic (updates, NTP etc)
 permit ip 192.168.2.0 0.0.0.255 any

interface GigabitEthernet0/1
 ip access-group ACL-DMZ in
```

### ACL-LAN

```
ip access-list extended ACL-LAN
 ! TCP return traffic delivered to LAN hosts
 permit tcp any 192.168.1.0 0.0.0.255 established
 ! DNS responses arriving at LAN hosts (source port 53)
 permit udp any eq domain 192.168.1.0 0.0.0.255
 ! ICMP echo-reply delivered to LAN hosts
 permit icmp any 192.168.1.0 0.0.0.255 echo-reply
 ! TCP return traffic from DMZ to LAN
 permit tcp 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 established
 ! Deny everything else inbound to the LAN
 deny ip any 192.168.1.0 0.0.0.255

interface GigabitEthernet0/2
 ip access-group ACL-LAN out
```

---

## DNS configuration , split-horizon

A DNS server at `192.168.2.20` (public NAT IP `203.0.113.4`) serves `www.ryderkick.local` with different answers depending on where the query originates. This is split-horizon DNS , a standard enterprise pattern.

**Why split-horizon is necessary:**

Internal clients getting back the public NAT IP `203.0.113.3` would send traffic to the perimeter router's outside interface. The router would need to hairpin NAT that traffic back inside , a configuration Packet Tracer does not support cleanly. Returning the private IP directly avoids the hairpin entirely and is more efficient.

External clients have no route to `192.168.2.10` and must use the public NAT IP. Returning the private IP to an external client would cause the connection to fail silently.

**DNS A records configured on the server:**

| Hostname | Address | Resolved by |
|----------|---------|-------------|
| www.ryderkick.local | 192.168.2.10 | Internal LAN clients |
| www.ryderkick.local | 203.0.113.3 | External clients |

**DHCP pool updated to point LAN clients at the DNS server:**

```
ip dhcp pool LAN
 dns-server 192.168.2.20
```

**Static NAT for the DNS server:**

```
ip nat inside source static 192.168.2.20 203.0.113.4
```

---

## Verification

```
! Confirm ACL application on each interface
show ip interface GigabitEthernet0/0 | include access list
show ip interface GigabitEthernet0/1 | include access list
show ip interface GigabitEthernet0/2 | include access list

! Check match counts , zero matches on a rule you expect to fire
! means traffic is being caught by something higher in the list
show ip access-lists

! Confirm NAT translations are correct
! Web server: 203.0.113.3 → 192.168.2.10
! DNS server:  203.0.113.4 → 192.168.2.20
show ip nat translations

! From PC1 , should resolve to 192.168.2.10 (internal IP)
nslookup www.ryderkick.local

! From PC1 browser , should load the web page
http://www.ryderkick.local

! From External PC , should resolve to 203.0.113.3 (public NAT IP)
nslookup www.ryderkick.local 203.0.113.4

! From External PC browser , should load the web page
http://www.ryderkick.local
```

### What to look for

| Check | Expected result |
|-------|----------------|
| ACL-OUTSIDE line 60 | Low or zero matches after setup |
| ACL-DMZ deny rule | Zero new matches during normal operation |
| NAT table | TCP port 80 entries only against 203.0.113.3 |
| PC1 nslookup | Returns 192.168.2.10 |
| External PC nslookup | Returns 203.0.113.3 |
| Both browsers | Web page loads successfully |

---

## Concepts demonstrated

| Concept | IOS command / detail |
|---------|---------------------|
| Named extended ACL | `ip access-list extended <name>` |
| Inbound ACL | `ip access-group <name> in` |
| Outbound ACL | `ip access-group <name> out` |
| TCP established matching | `permit tcp any any established` |
| UDP port direction | `eq domain` after source vs destination behaves differently |
| ACL match counters | `show ip access-lists` |
| Split-horizon DNS | Two A records for same hostname, different IPs per client zone |
| DNS static NAT | `ip nat inside source static 192.168.2.20 203.0.113.4` |

---

## Known limitations and real-world considerations

### ACLs alone cannot prevent DNS tunnelling

ACLs filter at Layer 3 and Layer 4 , they see IP addresses and port numbers only. They cannot inspect DNS payload content.

Tools such as dnscat2, iodine, and dns2tcp establish command-and-control channels or exfiltrate data by encoding arbitrary payloads inside valid DNS record types , particularly TXT, CNAME, and MX records. These packets are structurally legitimate DNS traffic. They arrive on UDP port 53, pass all ACL checks, and are forwarded without any indication that anything unusual is happening.

A compromised DMZ server could use DNS tunnelling to communicate with an external attacker even with ACL-DMZ fully in place. The tunnel traffic would match the existing `permit udp` rules and never trigger the deny.

In a production environment this would be mitigated by:
- A DNS security platform (Cisco Umbrella, Palo Alto DNS Security) performing payload inspection and query anomaly detection
- Query rate limiting at the resolver , DNS tunnelling generates unusually high query volumes
- Monitoring for anomalous TXT and NULL record queries , legitimate clients almost never issue these
- IDS/IPS rules (Snort, Suricata) watching for high-entropy DNS payloads and unusually large response sizes

The ACLs in this lab are a necessary foundation, not a complete defence. Layer 3/4 filtering should always be complemented by application-layer inspection for protocols as commonly abused as DNS.

---

## Lessons learned , troubleshooting log

This section documents every real issue encountered during the build and the reasoning used to resolve each one. The troubleshooting process is as instructive as the configuration itself.

---

### Issue 1 , DNS return traffic blocked by ACL-LAN

**Symptom:** LAN PCs could ping the web server but could not load the web page by hostname. Direct IP access worked fine.

**Root cause:** ACL-LAN contained this rule:
```
permit udp any 192.168.1.0 0.0.0.255 eq domain
```
The `eq domain` positioned after the destination matches UDP traffic headed **to** port 53 on LAN hosts , i.e. DNS queries arriving at LAN hosts. LAN PCs are DNS clients, not servers. DNS **responses** arriving from port 53 were never matched by any permit rule and fell through to the deny.

**Fix:** Reposition `eq domain` to follow the source address:
```
permit udp any eq domain 192.168.1.0 0.0.0.255
```
This matches UDP with **source** port 53 , DNS response traffic , destined for LAN hosts.

**Key takeaway:** The `eq` keyword in IOS ACL syntax matches the port of whichever address it follows. `permit udp any 192.168.1.0 eq domain` means destination port 53. `permit udp any eq domain 192.168.1.0` means source port 53. Getting these reversed produces a rule that compiles without error but matches completely the wrong traffic. There is no TCP `established` equivalent for UDP , every direction of a UDP exchange must be explicitly permitted.

---

### Issue 2 , ACL-DMZ blocking web server return traffic to LAN

**Symptom:** External access to the web server worked. LAN PC access by hostname was blocked. `show ip access-lists` showed the deny rule in ACL-DMZ accumulating matches during browsing attempts.

**Root cause:** ACL-DMZ relied on `permit tcp established` to allow web server responses back to LAN clients:
```
permit tcp 192.168.2.0 0.0.0.255 any established
```
Packet Tracer's implementation of `established` is inconsistent for traffic that has been through NAT translation. The return flow was not being recognised as belonging to an established session, causing it to fall through to the `deny ip DMZ → LAN` rule.

**Fix:** Add an explicit permit for web server TCP return traffic before the deny rule:
```
permit tcp host 192.168.2.10 192.168.1.0 0.0.0.255 established
```
This preserves the security intent , DMZ cannot initiate new connections toward the LAN , while reliably permitting return traffic from the specific web server host.

**Key takeaway:** `established` matching can behave inconsistently in Packet Tracer for NAT-translated flows. When a rule you expect to fire shows zero matches despite traffic clearly flowing, check whether NAT is rewriting addresses in a way that breaks session state tracking. Explicit host-specific permits are more reliable in PT for return traffic through a NAT boundary.

---

### Issue 3 , DNS ACL rules in ACL-DMZ matching wrong direction

**Symptom:** DNS resolution was working within the DMZ but DNS responses were not reaching LAN clients.

**Root cause:** ACL-DMZ contained these rules:
```
permit udp host 192.168.2.20 any eq domain
permit udp 192.168.2.0 0.0.0.255 any eq domain
```
Both had `eq domain` after the destination, meaning they permitted UDP headed **to** port 53 , DNS queries leaving the DMZ toward upstream resolvers. DNS **responses** from the DNS server have **source** port 53, so neither rule matched them.

**Fix:** Separate the two use cases explicitly:
```
! DNS responses FROM the DNS server , source port 53
permit udp host 192.168.2.20 eq domain any
! DNS queries TO upstream resolvers , destination port 53
permit udp 192.168.2.0 0.0.0.255 any eq domain
```

**Key takeaway:** For any UDP service you need to think about both directions independently and write a rule for each. A DNS server needs two rules: one for responses it sends (source port 53) and one for queries it makes upstream (destination port 53). Writing them identically silently drops one direction.

---

### Issue 4 , ACL-OUTSIDE not evaluating DNS responses correctly

**Symptom:** External PC DNS queries were reaching the DNS server but responses were being dropped.

**Root cause:** A rule added to ACL-OUTSIDE to permit DNS responses:
```
permit udp any eq domain any
```
Was applied **inbound** on G0/0. ACL-OUTSIDE only evaluates traffic arriving **from** the internet. DNS responses leaving toward the External PC exit G0/0 **outbound** and are not evaluated by this ACL at all , making the rule both unnecessary and ineffective.

**Fix:** Remove the redundant rule. Since there is no outbound ACL on G0/0, response traffic exits freely. Add a specific inbound permit for DNS queries to the DNS server's public NAT IP instead:
```
permit udp any host 203.0.113.4 eq domain
```

**Key takeaway:** Always reason about which direction a packet is travelling relative to the interface an ACL is applied to before writing a rule for it. Drawing the packet flow on paper and marking the ACL evaluation point explicitly prevents this category of mistake.

---

### Issue 5 , `ip host` entry on ISP router overriding DNS resolution

**Symptom:** External PC DNS queries were returning the wrong IP address for `www.ryderkick.local` despite the DNS server A record appearing correct.

**Root cause:** The ISP router had a static host table entry:
```
ip host www.ryderkick.local 203.0.113.4
```
IOS `ip host` entries act as a local DNS cache that takes priority over external DNS resolution. The ISP router was intercepting the query and returning `.4` (the DNS server's IP) instead of forwarding the query to the DNS server and returning `.3` (the web server's IP). The client had no indication the answer came from the wrong source.

**Fix:**
```
no ip host www.ryderkick.local 203.0.113.4
```

**Key takeaway:** `ip host` entries are silently authoritative , they answer DNS queries without any indication to the client that a local override is in effect. In a production environment, a misconfigured or maliciously inserted `ip host` entry on a transit router would constitute DNS hijacking. Baseline audits of router configurations should always include `show hosts` to check for unexpected static entries.

---

### Issue 6 , Split-horizon DNS required for internal clients

**Symptom:** After all ACL and NAT issues were resolved, internal LAN clients accessing `www.ryderkick.local` by name were triggering NAT hairpin behaviour that Packet Tracer does not support.

**Root cause:** The DNS server was returning `203.0.113.3` (the public NAT IP) to all clients regardless of origin. Internal LAN clients sending traffic to `203.0.113.3` were routing toward the perimeter router's outside interface, which would need to translate the destination back to `192.168.2.10` and forward it internally. This NAT hairpin configuration is not supported in Packet Tracer.

**Fix:** Configure two separate A records for the same hostname:

| Hostname | Address | For |
|----------|---------|-----|
| www.ryderkick.local | 192.168.2.10 | Internal clients |
| www.ryderkick.local | 203.0.113.3 | External clients |

Internal clients receive the private IP directly routable within the network. External clients receive the public NAT IP. This is split-horizon DNS , a standard enterprise pattern implemented in production by running separate internal and external DNS zones that return different answers for the same hostname.

**Key takeaway:** DNS-to-NAT interaction is a common source of subtle connectivity issues. When internal clients use public DNS records to reach internal services, NAT hairpinning is required. Split-horizon DNS avoids the hairpin entirely and is the cleaner, more scalable solution.

---

## Files

| File | Description |
|------|-------------|
| `lab03.pkt` | Packet Tracer source file (PT 8.x) |
| `topology.png` | Network diagram |
| `configs/Perimeter.txt` | Final running configuration including all ACLs |
| `configs/ISP.txt` | Final running configuration |

More screenshots found in the `screenshots` folder. 

---

## Skills demonstrated

`Named extended ACLs` · `Inbound and outbound ACL placement` · `TCP established matching` · `UDP directional filtering` · `ACL troubleshooting` · `Split-horizon DNS` · `DNS/NAT interaction` · `DMZ security policy` · `Layer 3/4 vs Layer 7 defence` · `DNS tunnelling awareness` · `IOS host table` · `Simulation mode packet tracing`
