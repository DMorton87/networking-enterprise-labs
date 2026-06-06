# Lab 02 - VLAN Security and Switch Hardening

## Scenario

A growing organization requires stronger access-layer security controls to prevent unauthorized devices from joining the network and to reduce the risk of common Layer 2 attacks. The network must continue to provide connectivity between departments while implementing security-focused switch hardening measures.

## Objectives

* Implement VLAN segmentation for multiple departments
* Configure a dedicated management VLAN
* Configure 802.1Q trunking
* Change the native VLAN from the default configuration
* Implement port security using sticky MAC addresses
* Enable BPDU Guard on access ports
* Create a parking lot VLAN for unused switch ports
* Validate security controls through testing

## Device Inventory

### Network Devices

* Cisco 2911 Router (R1)
* Cisco 2960 Switch (SW1)

### Endpoints

* Sales-PC
* HR-PC
* IT-PC
* Rogue-Laptop (Security Testing)

## VLAN Design

| VLAN | Purpose          | Network         |
| ---- | ---------------- | --------------- |
| 10   | Sales            | 192.168.10.0/24 |
| 20   | HR               | 192.168.20.0/24 |
| 30   | IT               | 192.168.30.0/24 |
| 40   | Management       | 192.168.40.0/24 |
| 999  | Native VLAN      | N/A             |
| 666  | Parking Lot VLAN | N/A             |

## Technologies Implemented

* VLAN Segmentation
* Router-on-a-Stick Inter-VLAN Routing
* DHCP Services
* 802.1Q Trunking
* Port Security
* Sticky MAC Address Learning
* BPDU Guard
* Management VLAN Separation
* Native VLAN Hardening
* Parking Lot VLAN Strategy

## Security Controls Implemented

### Dedicated Management VLAN

Infrastructure management traffic was separated from user traffic by moving switch management services to VLAN 40. This reduces exposure of management interfaces and reflects common enterprise design practices.

### Native VLAN Hardening

The trunk native VLAN was changed from the default VLAN 1 to VLAN 999. This helps reduce exposure to common VLAN hopping techniques and demonstrates secure trunk configuration practices.

### Port Security

Port security was configured on all user-facing access ports using sticky MAC address learning.

Configuration included:

* Maximum of one MAC address per port
* Sticky MAC learning
* Shutdown action on violation

This prevents unauthorized devices from replacing approved endpoints on the network.

### BPDU Guard

BPDU Guard was enabled on PortFast-enabled access ports.

If a device begins transmitting BPDUs on an access port, the port is automatically disabled to prevent accidental or malicious switching infrastructure from being introduced into the network.

### Parking Lot VLAN

Unused switch ports were assigned to VLAN 666 and administratively disabled.

This provides an additional layer of protection by ensuring that unused ports cannot immediately gain access to production networks if accidentally enabled.

## Verification

The following verification procedures were completed:

* VLAN creation and assignment validation
* Trunk configuration validation
* Inter-VLAN routing verification
* DHCP address assignment verification
* Port security verification
* BPDU Guard verification
* Management VLAN connectivity testing

## Security Validation

A Rogue-Laptop was introduced during testing to validate port security controls.

After disconnecting the authorized endpoint and connecting the unauthorized device to a secured access port, the switch detected a MAC address violation and triggered the configured port security policy.

This demonstrated that unauthorized devices could not be introduced without detection and enforcement.

## Challenges Encountered

One challenge involved understanding the distinction between switch management interfaces and router subinterfaces when configuring management VLANs. Troubleshooting this issue reinforced the importance of understanding Layer 2 versus Layer 3 device management and IP addressing design.

Additionally, verification of security features required careful selection of operational commands to confirm that configured controls were functioning as expected.

## Lessons Learned

This lab demonstrated how Layer 2 security controls can significantly improve the security posture of an enterprise network. While VLAN segmentation provides logical separation between departments, additional controls such as Port Security, BPDU Guard, native VLAN hardening, and management VLAN separation provide defense-in-depth against common access-layer threats.

The exercise reinforced the importance of validating security controls through testing rather than relying solely on configuration review.
