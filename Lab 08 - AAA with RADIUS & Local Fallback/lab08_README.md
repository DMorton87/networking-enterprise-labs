# Lab 08 — AAA with RADIUS & Local Fallback

A hands-on Cisco Packet Tracer lab implementing centralised authentication across multiple network devices using a RADIUS server, with a local database fallback for resilience. Covers AAA model configuration, RADIUS server setup, SSH-only access hardening, enable authentication via RADIUS, and the security implications of centralised credential management.

---

## Overview

Before AAA, each network device manages its own authentication independently — separate passwords on every router, no central audit trail, and a password change requiring manual updates on every device individually. With 50 devices that means 50 separate changes every time an engineer leaves or a password rotation is required.

RADIUS solves this by centralising authentication. One server holds the user database. Every device consults it on every login attempt. Add a user once and they can authenticate to every device. Revoke a user once and they lose access everywhere simultaneously.

The local fallback exists as a break-glass mechanism — if the RADIUS server is unreachable, locally configured credentials allow continued access to network devices. This is not a backdoor; it is a deliberate resilience design that prevents a RADIUS outage from locking administrators out of the network entirely.

---

## Objectives

- Enable AAA globally on three routers using `aaa new-model`
- Configure RADIUS server clients and user accounts in Packet Tracer
- Point each router at the RADIUS server with a shared secret key
- Define an authentication method list with RADIUS primary and local fallback
- Apply the method list to VTY lines with SSH-only transport
- Configure enable authentication through RADIUS for privileged access
- Test RADIUS authentication, local fallback, and SSH-only access
- Understand the security implications of centralised credential management

---

## Topology

```
              [ RADIUS server ]
               10.0.0.100/24
                    |
               [ SW-CORE ]
              /     |      \
            R1      R2      R3
        10.0.0.1  10.0.0.2  10.0.0.3
                    |
               [ Admin PC ]
               10.0.0.200/24
               SSH test client
```

---

## Device list

| Device | PT model | Role |
|--------|----------|------|
| R1, R2, R3 | ISR4331 | Network devices secured with AAA |
| SW-CORE | 2960-24TT | Core access switch |
| RADIUS server | Server-PT | Centralised authentication server |
| Admin PC | PC-PT | SSH test client |

---

## IP addressing

| Device | Interface | Address |
|--------|-----------|---------|
| R1 | G0/0/0 | 10.0.0.1/24 |
| R2 | G0/0/0 | 10.0.0.2/24 |
| R3 | G0/0/0 | 10.0.0.3/24 |
| RADIUS server | Fa0 | 10.0.0.100/24 |
| Admin PC | Fa0 | 10.0.0.200/24 |

---

## Cabling

| Cable | From | Interface | To | Interface |
|-------|------|-----------|----|-----------|
| 1 | SW-CORE | Fa0/1 | R1 | G0/0/0 |
| 2 | SW-CORE | Fa0/2 | R2 | G0/0/0 |
| 3 | SW-CORE | Fa0/3 | R3 | G0/0/0 |
| 4 | SW-CORE | Fa0/4 | RADIUS server | Fa0 |
| 5 | SW-CORE | Fa0/5 | Admin PC | Fa0 |

---

## Key configurations

### Step 1 — Basic interfaces on R1, R2, R3

```
interface GigabitEthernet0/0/0
 ip address 10.0.0.x 255.255.255.0
 no shutdown
```

### Step 2 — RADIUS server setup

Services → AAA on the Server-PT:

**Network configuration — router clients:**

| Client name | Client IP | Secret key |
|-------------|-----------|------------|
| R1 | 10.0.0.1 | RADIUSKEY |
| R2 | 10.0.0.2 | RADIUSKEY |
| R3 | 10.0.0.3 | RADIUSKEY |

**User setup — RADIUS accounts:**

| Username | Password |
|----------|----------|
| radiusadmin | Admin123 |
| netops | Netops456 |

### Step 3 — AAA configuration on all three routers (identical)

```
! Enable AAA globally — must come before all other AAA commands
aaa new-model

! Point the router at the RADIUS server
! auth-port 1645 is the Packet Tracer default
! key must match what was configured on the server
radius-server host 10.0.0.100 auth-port 1645 key RADIUSKEY

! Local fallback user — break-glass account for RADIUS outages
! privilege 15 grants full access without needing a separate enable step
username localadmin privilege 15 secret Local789

! Authentication method list for login
! RADIUS is tried first — local database only if RADIUS is unreachable
! Note: if RADIUS rejects credentials, local is NOT tried — fallback
! only triggers on timeout/unreachable, not on explicit rejection
aaa authentication login default group radius local

! Enable authentication through RADIUS
! Allows the same RADIUS credentials to authenticate privileged access
aaa authentication enable default group radius enable

! Enable secret for local fallback during enable authentication
enable secret Cisco123

! VTY lines — SSH only, no Telnet
! AAA method list applied explicitly
line vty 0 4
 login authentication default
 transport input ssh
```

### Step 4 — SSH hardening on all three routers

```
hostname R1
ip domain-name lab.local
crypto key generate rsa modulus 1024
ip ssh version 2
```

---

## Verification

```
! Confirm AAA is enabled and method lists are correct
show aaa method-lists authentication

! Confirm RADIUS server configuration
show radius-server

! Confirm SSH is enabled and version
show ip ssh

! Confirm local user database
show running-config | section username

! Confirm VTY line configuration
show running-config | section line vty

! View active sessions after login
show users
```

### Testing sequence

**Test 1 — RADIUS authentication (normal operation):**
```
! From Admin PC command prompt
ssh -l radiusadmin 10.0.0.1
! Password: Admin123
! Then at R1> prompt:
enable
! Username: radiusadmin
! Password: Admin123
! Expected: R1#
show privilege
! Expected: Current privilege level is 15
```

**Test 2 — Second RADIUS user:**
```
ssh -l netops 10.0.0.1
! Password: Netops456
```

**Test 3 — Local fallback (RADIUS offline):**
```
! On RADIUS server: Services → AAA → toggle Off
! Then from Admin PC:
ssh -l localadmin 10.0.0.1
! Password: Local789
! Expected: login succeeds via local database
! Turn RADIUS back on after testing
```

**Test 4 — Wrong credentials (RADIUS online):**
```
ssh -l localadmin 10.0.0.1
! Expected: login fails — RADIUS doesn't know this user
! Local database is NOT tried while RADIUS is reachable
```

### What to look for

| Check | Expected result |
|-------|----------------|
| SSH with RADIUS credentials | Login succeeds |
| Enable with RADIUS credentials | Privilege 15 granted |
| `show privilege` | Returns 15 |
| SSH with local creds (RADIUS up) | Login fails — local only for fallback |
| SSH with local creds (RADIUS down) | Login succeeds — fallback working |
| Telnet attempt | Connection refused — SSH only |
| Same RADIUS creds on R2 and R3 | Login succeeds — centralised auth confirmed |

---

## Concepts demonstrated

| Concept | IOS command |
|---------|-------------|
| Enable AAA | `aaa new-model` |
| RADIUS server | `radius-server host` |
| Login method list | `aaa authentication login default group radius local` |
| Enable method list | `aaa authentication enable default group radius enable` |
| Local fallback user | `username X privilege 15 secret Y` |
| Apply to VTY | `login authentication default` |
| SSH only | `transport input ssh` |
| SSH version 2 | `ip ssh version 2` |
| RSA key generation | `crypto key generate rsa modulus 1024` |
| Verify AAA | `show aaa method-lists authentication` |

---

## How the fallback chain works

The method list `group radius local` has specific behaviour worth understanding precisely:

```
aaa authentication login default group radius local
```

| Scenario | Result |
|----------|--------|
| RADIUS reachable, credentials valid | Permit — RADIUS accept |
| RADIUS reachable, credentials invalid | Deny — RADIUS reject, local NOT tried |
| RADIUS unreachable (timeout) | Try local database |
| RADIUS unreachable, local credentials valid | Permit — local accept |
| RADIUS unreachable, local credentials invalid | Deny |
| Both unavailable | Deny |

The local database is only consulted when RADIUS cannot be reached. An explicit RADIUS rejection does not fall through to local — this is intentional and prevents an attacker from bypassing RADIUS by somehow causing it to reject valid credentials and relying on a weaker local password.

---

## Security considerations — the double-edged sword of centralisation

Centralising authentication through RADIUS is simultaneously the most powerful security improvement and the most significant risk concentration in this lab.

### The benefits

A single user database means a single place to enforce password policy, a single place to revoke access, and a single audit trail showing who authenticated to which device and when. When an engineer leaves, one account deletion removes their access to every device on the network instantly.

### The risk

Those same properties mean that a single compromised set of RADIUS credentials grants access to every device simultaneously. In this lab, `radiusadmin` with password `Admin123` can SSH into R1, R2, and R3 and reach privilege 15 on all of them with identical credentials. The centralisation that makes management efficient makes credential compromise catastrophic.

In a production environment this is mitigated by:

**Per-user privilege levels** — RADIUS can return a `Service-Type` or `Cisco-AVPair` attribute specifying the privilege level granted to each user individually. Helpdesk accounts get privilege 1 (show commands only), engineers get privilege 7, senior engineers and break-glass accounts get privilege 15. No single compromised account automatically equals full network access.

**Per-device access control** — RADIUS policies can restrict which devices a user is permitted to authenticate to. A WAN engineer's account should not be able to log into a core switch.

**Multi-factor authentication** — enterprise RADIUS deployments (Cisco ISE, FreeRADIUS with Google Authenticator, Microsoft NPS with Azure MFA) require a second factor in addition to credentials. A leaked password alone is not enough to authenticate.

**AAA accounting** — the third A in AAA, not implemented in this lab but critical in production. Every login, every `enable`, and optionally every command typed is logged to a central server with a timestamp. Even if an attacker authenticates successfully, there is a complete audit trail.

**Short session timeouts** — `exec-timeout 10 0` on VTY lines terminates idle sessions after 10 minutes, limiting the window for session hijacking.

**Strong shared secrets** — the RADIUS shared secret (`RADIUSKEY` in this lab) should be a long random string in production. It protects the RADIUS exchange between the router and server — a weak secret makes the authentication protocol itself vulnerable.

---

## Design note — Telnet excluded

This lab implements SSH-only access on all VTY lines. Telnet is excluded entirely. Telnet transmits all data including usernames and passwords in cleartext — any device on the path between the admin workstation and the router can capture credentials trivially. In a network where RADIUS credentials grant access to every device simultaneously, a single Telnet session sniffed in transit could compromise the entire network. SSH with version 2 enforced provides encrypted transport for all management traffic.

---

## Files

| File | Description |
|------|-------------|
| `lab08.pkt` | Packet Tracer source file (PT 9.x) |
| `topology.png` | Network diagram |
| `configs/R1.txt` | Final running configuration |
| `configs/R2.txt` | Final running configuration |
| `configs/R3.txt` | Final running configuration |

---

## Skills demonstrated

`AAA` · `RADIUS` · `aaa new-model` · `Authentication method lists` · `Local fallback` · `Enable authentication` · `SSH v2` · `RSA key generation` · `VTY hardening` · `Centralised credential management` · `Privilege level control` · `Break-glass accounts` · `Management plane security`
