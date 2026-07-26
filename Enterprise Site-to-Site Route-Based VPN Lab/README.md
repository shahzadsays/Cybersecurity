# Enterprise Site-to-Site Route-Based VPN Lab

A practical Palo Alto VM-Series lab that demonstrates a route-based site-to-site IPsec VPN between two branch sites in EVE-NG. The design validates WAN connectivity, tunnel buildout, static routing, security policy enforcement, and end-to-end ICMP reachability across an encrypted overlay.

This lab is built for hands-on firewall engineering practice and portfolio use.

---

## Lab Objective

The goal of this lab is to establish bidirectional communication between two simulated enterprise sites:

- **London** using `PA-LDN`
- **Manchester** using `PA-MAN`

Traffic is carried over a route-based IPsec tunnel using PAN-OS 11.2, with CLI-driven configuration and troubleshooting.

Palo Alto’s route-based VPN guidance confirms that traffic is routed based on destination IP and that a tunnel interface plus static routes are the standard approach for this design. [web:98][web:99][web:103]

---

## Topology Overview

```text
London LAN (10.1.1.0/24) ---- PA-LDN ---- 172.16.1.0/30 ---- PA-MAN ---- Manchester LAN (10.2.2.0/24)
```

### Site A: London

| Component | Value |
|---|---:|
| Firewall | PA-LDN |
| WAN interface | `ethernet1/2` |
| WAN IP | `172.16.1.1/30` |
| LAN interface | `ethernet1/1` |
| LAN IP | `10.1.1.1/24` |
| Tunnel interface | `tunnel.1` |
| End host | `10.1.1.10/24` |

### Site B: Manchester

| Component | Value |
|---|---:|
| Firewall | PA-MAN |
| WAN interface | `ethernet1/2` |
| WAN IP | `172.16.1.2/30` |
| LAN interface | `ethernet1/1` |
| LAN IP | `10.2.2.1/24` |
| Tunnel interface | `tunnel.1` |
| End host | `10.2.2.10/24` |

---

## Configuration Method

This lab uses a CLI-first approach to validate syntax, routing behavior, and tunnel buildout in a fresh PAN-OS 11.2 environment.

### Core Build Elements

- Layer 3 interface setup.
- Interface management profile for ping validation.
- Security zones for WAN, LAN, and VPN traffic.
- Virtual router with static routes.
- IKEv2 Phase 1 configuration.
- IPsec Phase 2 configuration.
- Route-based tunnel interface.
- Explicit security policies for VPN traffic.

Palo Alto documentation and KB examples show that a route-based VPN uses a tunnel interface, static route placement, and security rules to allow the traffic flow. [web:98][web:99][web:103]

---

## Troubleshooting Highlights

### CLI Syntax Differences

Some settings behave differently in the CLI than in the GUI, especially under crypto and gateway configuration.

### Static Route Behavior

For tunnel routes, the next-hop is not always required in the CLI when the route points directly to a tunnel interface.

### End-Host Routing

The simulated LAN devices were Cisco IOS nodes acting as hosts, so they needed a default route back to the firewall to reach remote networks.

### Interface Management Profiles

The `Allow-Ping` profile was used to simplify step-by-step validation during build and troubleshooting.

These are common operational lessons when building route-based VPNs in a lab, especially when using router-based end hosts rather than true PCs. [web:100][web:103][web:104]

---

## Verification Results

### ICMP Validation

After correcting routing and syntax issues, end-to-end ping was successful between:

- `10.1.1.10` in London
- `10.2.2.10` in Manchester

### Packet Capture Validation

Traffic on the WAN side showed encrypted IPsec behavior, while LAN-side captures showed normal ICMP traffic before encryption and after decryption.

This is the expected behavior for a working route-based site-to-site IPsec tunnel. [web:98][web:99][web:106]

---

## Firewall Configuration

The following configuration was applied on both firewalls with site-specific IP addressing and peer references.

### PA-LDN Configuration

```bash
configure

set network profiles interface-management-profile Allow-Ping ping yes

set network interface ethernet ethernet1/2 layer3 ip 172.16.1.1/30
set network interface ethernet ethernet1/1 layer3 ip 10.1.1.1/24
set network interface ethernet ethernet1/1 layer3 interface-management-profile Allow-Ping
set network interface ethernet ethernet1/2 layer3 interface-management-profile Allow-Ping

set network interface tunnel units tunnel.1
set network interface tunnel units tunnel.1 interface-management-profile Allow-Ping

set zone OUTSIDE network layer3 ethernet1/2
set zone LAN network layer3 ethernet1/1
set zone VPN-Zone network layer3 tunnel.1

set network virtual-router default interface [ ethernet1/1 ethernet1/2 tunnel.1 ]

set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile encryption aes-256-cbc
set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile hash sha256
set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile dh-group group14
set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile lifetime hours 8

set network ike gateway GW-To-Manchester authentication pre-shared-key key PaloAltoSecret123!
set network ike gateway GW-To-Manchester local-address interface ethernet1/2 ip 172.16.1.1/30
set network ike gateway GW-To-Manchester peer-address ip 172.16.1.2
set network ike gateway GW-To-Manchester protocol version ikev2
set network ike gateway GW-To-Manchester protocol ikev2 ike-crypto-profile IKE-Crypto-Profile

set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile esp encryption aes-256-cbc
set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile esp authentication sha256
set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile dh-group group14
set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile lifetime hours 1

set network tunnel ipsec Tunnel-To-Manchester tunnel-interface tunnel.1
set network tunnel ipsec Tunnel-To-Manchester auto-key ike-gateway GW-To-Manchester
set network tunnel ipsec Tunnel-To-Manchester auto-key ipsec-crypto-profile IPsec-Crypto-Profile

set network virtual-router default routing-table ip static-route Route-To-Manchester destination 10.2.2.0/24 interface tunnel.1

set rulebase security rules Outbound-To-VPN from LAN to VPN-Zone source any destination any application any service any action allow
set rulebase security rules Inbound-From-VPN from VPN-Zone to LAN source any destination any application any service any action allow

commit
exit
```

### PA-MAN Configuration

```bash
configure

set network profiles interface-management-profile Allow-Ping ping yes

set network interface ethernet ethernet1/2 layer3 ip 172.16.1.2/30
set network interface ethernet ethernet1/1 layer3 ip 10.2.2.1/24
set network interface ethernet ethernet1/1 layer3 interface-management-profile Allow-Ping
set network interface ethernet ethernet1/2 layer3 interface-management-profile Allow-Ping

set network interface tunnel units tunnel.1
set network interface tunnel units tunnel.1 interface-management-profile Allow-Ping

set zone OUTSIDE network layer3 ethernet1/2
set zone LAN network layer3 ethernet1/1
set zone VPN-Zone network layer3 tunnel.1

set network virtual-router default interface [ ethernet1/1 ethernet1/2 tunnel.1 ]

set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile encryption aes-256-cbc
set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile hash sha256
set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile dh-group group14
set network ike crypto-profiles ike-crypto-profiles IKE-Crypto-Profile lifetime hours 8

set network ike gateway GW-To-London authentication pre-shared-key key PaloAltoSecret123!
set network ike gateway GW-To-London local-address interface ethernet1/2 ip 172.16.1.2/30
set network ike gateway GW-To-London peer-address ip 172.16.1.1
set network ike gateway GW-To-London protocol version ikev2
set network ike gateway GW-To-London protocol ikev2 ike-crypto-profile IKE-Crypto-Profile

set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile esp encryption aes-256-cbc
set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile esp authentication sha256
set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile dh-group group14
set network ike crypto-profiles ipsec-crypto-profiles IPsec-Crypto-Profile lifetime hours 1

set network tunnel ipsec Tunnel-To-London tunnel-interface tunnel.1
set network tunnel ipsec Tunnel-To-London auto-key ike-gateway GW-To-London
set network tunnel ipsec Tunnel-To-London auto-key ipsec-crypto-profile IPsec-Crypto-Profile

set network virtual-router default routing-table ip static-route Route-To-London destination 10.1.1.0/24 interface tunnel.1

set rulebase security rules Outbound-To-VPN from LAN to VPN-Zone source any destination any application any service any action allow
set rulebase security rules Inbound-From-VPN from VPN-Zone to LAN source any destination any application any service any action allow

commit
exit
```

---

## Test Plan

### End-to-End Checks

- Ping from London host `10.1.1.10` to Manchester host `10.2.2.10`.
- Ping from Manchester host to London host.
- Verify IKE SA establishment.
- Verify IPsec SA establishment.
- Capture WAN-side ESP traffic and LAN-side ICMP traffic.

### Useful Commands

```bash
show vpn ike-sa
show vpn ipsec-sa
show routing route
test vpn ike-sa gateway GW-To-Manchester
test vpn ipsec-sa tunnel Tunnel-To-Manchester
```

---

## What This Lab Demonstrates

- Route-based IPsec VPN design
- PAN-OS CLI configuration
- Tunnel interface routing
- IKEv2 and IPsec crypto setup
- Security policy controls
- Lab troubleshooting and validation
- Packet capture verification

---

## Repository Structure

```text
paloalto-route-vpn-lab/
├── configs/
├── captures/
├── screenshots/
├── diagrams/
└── README.md
```

---

## Notes

- This is a real routed VPN lab, not a policy-based shortcut.
- The tunnel uses route-based forwarding with static routes.
- End hosts behind Cisco IOS nodes require proper default routing.
- Always commit before testing VPN state.

---

## License

For educational and portfolio use.
