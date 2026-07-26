# Enterprise Branch Lab (App-ID Validated)

A production-style enterprise branch lab built in EVE-NG to demonstrate Palo Alto firewall segmentation, VLAN-based routing, NAT, App-ID policy control, and Cisco IOS-based access, server, and edge services.

This lab validates end-to-end traffic flow across multiple internal zones, internet breakout, and Layer 7 policy enforcement.

---

## Lab Objective

The goal of this lab was to build a branch office design with:

- One Palo Alto VM-Series firewall.
- One Cisco core switch providing VLANs, trunking, and DHCP.
- One Cisco ISP router simulating the internet and a web server.
- Multiple user and service VLANs.
- App-ID-based security policy enforcement.
- Verified web access, ICMP routing, and NAT.

Palo Alto documentation confirms that App-ID is used to classify and control applications, while URL filtering is applied through security policy rules. [web:117][web:111]

---

## Topology Overview

```text
Internet / Home Network
        |
     Cloud0
        |
   PA-VM Management
        |
   PA-VM Data Plane
        |
   ISP-RTR (Untrust / Simulated Internet / Web)
        |
   Cisco Core Switch (sw2)
     |      |       |
   VLAN10  VLAN20  VLAN30
   Users   Servers   Voice
```

### Addressing Summary

| Device | Interface | IP |
|---|---|---:|
| PA-VM | Management | `192.168.1.131/24` |
| PA-VM | ethernet1/2 | `192.168.100.2/30` |
| PA-VM | ethernet1/3.10 | `10.1.10.1/24` |
| PA-VM | ethernet1/3.20 | `10.1.20.1/24` |
| PA-VM | ethernet1/3.30 | `10.1.30.1/24` |
| ISP-RTR | FastEthernet0/0 | `192.168.100.1/30` |
| ISP-RTR | Loopback0 | `8.8.8.8/32` |
| ISP-RTR | Loopback1 | `13.13.13.13/32` |
| sw2 | VLAN10 SVI | `10.1.10.2/24` |
| sw2 | VLAN30 SVI | `10.1.30.2/24` |

---

## Design Highlights

- VLAN-based segmentation for users, servers, and voice.
- Firewall subinterfaces used as default gateways.
- NAT for outbound internet access.
- App-ID policy update to avoid unwanted TCP resets for the Cisco IOS web payload.
- Cisco IOS HTTP service enabled for custom web validation.

Palo Alto’s App-ID documentation explains that App-ID identifies and controls application traffic, which is the basis for the policy adjustment used in this lab. [web:117]

---

## Palo Alto Firewall Configuration

### Management Access

```bash
configure
set deviceconfig system type static
set deviceconfig system ip-address 192.168.1.131 netmask 255.255.255.0 default-gateway 192.168.1.1 dns-setting servers primary 8.8.8.8
commit
```

### Data Plane

```bash
configure

set network profiles interface-management-profile Allow_Ping ping yes

set network interface ethernet ethernet1/2 layer3 ip 192.168.100.2/30
set network interface ethernet ethernet1/2 interface-management-profile Allow_Ping
set network interface ethernet ethernet1/3 layer3

set network interface ethernet ethernet1/3 layer3 units ethernet1/3.10 tag 10 ip 10.1.10.1/24
set network interface ethernet ethernet1/3 layer3 units ethernet1/3.10 interface-management-profile Allow_Ping
set network interface ethernet ethernet1/3 layer3 units ethernet1/3.20 tag 20 ip 10.1.20.1/24
set network interface ethernet ethernet1/3 layer3 units ethernet1/3.20 interface-management-profile Allow_Ping
set network interface ethernet ethernet1/3 layer3 units ethernet1/3.30 tag 30 ip 10.1.30.1/24
set network interface ethernet ethernet1/3 layer3 units ethernet1/3.30 interface-management-profile Allow_Ping

set zone Untrust-Zone network layer3 [ ethernet1/2 ]
set zone Users-Zone network layer3 [ ethernet1/3.10 ]
set zone Servers-Zone network layer3 [ ethernet1/3.20 ]
set zone Voice-Zone network layer3 [ ethernet1/3.30 ]

set network virtual-router default interface [ ethernet1/2 ethernet1/3.10 ethernet1/3.20 ethernet1/3.30 ]
set network virtual-router default routing-table ip static-route To_Internet destination 0.0.0.0/0 nexthop ip-address 192.168.100.1

set profiles url-filtering Block-Time-Wasters block [ social-networking games ]
set profiles url-filtering Block-Time-Wasters credential-enforcement mode disabled
set profiles url-filtering Block-Time-Wasters credential-enforcement log-severity medium

set rulebase nat rules Branch_NAT from [ Users-Zone Servers-Zone Voice-Zone ]
set rulebase nat rules Branch_NAT to Untrust-Zone
set rulebase nat rules Branch_NAT source any
set rulebase nat rules Branch_NAT destination any
set rulebase nat rules Branch_NAT service any
set rulebase nat rules Branch_NAT to-interface ethernet1/2
set rulebase nat rules Branch_NAT source-translation dynamic-ip-and-port interface-address interface ethernet1/2

set rulebase security rules Users_To_Internet from Users-Zone
set rulebase security rules Users_To_Internet to Untrust-Zone
set rulebase security rules Users_To_Internet source any
set rulebase security rules Users_To_Internet destination any
set rulebase security rules Users_To_Internet application [ web-browsing ssl dns ping cisco-ios-ui ]
set rulebase security rules Users_To_Internet service application-default
set rulebase security rules Users_To_Internet action allow
set rulebase security rules Users_To_Internet profile-setting profiles url-filtering Block-Time-Wasters

set rulebase security rules Users_To_Servers from Users-Zone
set rulebase security rules Users_To_Servers to Servers-Zone
set rulebase security rules Users_To_Servers source any
set rulebase security rules Users_To_Servers destination 10.1.20.0/24
set rulebase security rules Users_To_Servers application any
set rulebase security rules Users_To_Servers service application-default
set rulebase security rules Users_To_Servers action allow

set rulebase security rules Voice_Outbound from Voice-Zone
set rulebase security rules Voice_Outbound to Untrust-Zone
set rulebase security rules Voice_Outbound source any
set rulebase security rules Voice_Outbound destination any
set rulebase security rules Voice_Outbound application [ sip rtp ]
set rulebase security rules Voice_Outbound service application-default
set rulebase security rules Voice_Outbound action allow

commit
```

App-ID and security policy behavior in this lab reflects Palo Alto best-practice policy control, while URL filtering is attached through security policy enforcement. [web:115][web:111][web:117]

---

## Cisco Core Switch

### VLAN and Trunk Configuration

```bash
enable
configure terminal
hostname sw2

vlan 10
 name Users
vlan 20
 name Servers
vlan 30
 name Voice
exit

interface FastEthernet1/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 spanning-tree portfast trunk
 no shutdown
exit

interface FastEthernet1/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit

interface FastEthernet1/1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit

interface FastEthernet1/0
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
exit
```

### DHCP Services

```bash
ip dhcp excluded-address 10.1.10.1 10.1.10.10
ip dhcp excluded-address 10.1.30.1 10.1.30.10

ip dhcp pool VLAN10_USERS
 network 10.1.10.0 255.255.255.0
 default-router 10.1.10.1
 dns-server 8.8.8.8
exit

ip dhcp pool VLAN30_VOICE
 network 10.1.30.0 255.255.255.0
 default-router 10.1.30.1
 dns-server 8.8.8.8
exit

interface Vlan10
 ip address 10.1.10.2 255.255.255.0
 no shutdown
exit

interface Vlan30
 ip address 10.1.30.2 255.255.255.0
 no shutdown
exit

end
write memory
```

---

## ISP Router

### Interface and HTTP Configuration

```bash
enable
configure terminal
hostname ISP-RTR

interface FastEthernet0/0
 description Link to PAFW Untrust
 ip address 192.168.100.1 255.255.255.252
 no shutdown
exit

interface Loopback0
 description Simulated Internet
 ip address 8.8.8.8 255.255.255.255
exit

interface Loopback1
 ip address 13.13.13.13 255.255.255.255
 no shutdown
exit

ip http server
ip http secure-server
username admin privilege 15 secret cisco123
ip http authentication local
ip http active-session-modules none
ip route 0.0.0.0 0.0.0.0 Null0
end
write memory
```

### Custom HTML Page

```bash
tclsh
set html [open "flash:hello.html" w+]
puts $html "<h1>hello</h1>"
close $html
tclquit
```

```bash
configure terminal
ip http path flash:
end
write memory
```

---

## Validation Tests

### Test 1: ICMP / Routing / NAT

From `VPC10`, ping `8.8.8.8`.

Expected result:
- Successful replies.
- Confirms routing, VLAN trunking, and NAT are working.

### Test 2: App-ID Web Verification

From the Windows host in the Users zone, browse to:

```text
http://13.13.13.13/hello.html
```

Expected result:
- A page displaying a bold `hello`.

### Log Validation

Check **Monitor > Logs > Traffic** on the Palo Alto firewall.

Expected result:
- Traffic matches the `Users_To_Internet` rule.
- The added `cisco-ios-ui` application entry prevents TCP reset behavior and allows the page to load.

---

## Troubleshooting Lessons

- Firewall subinterfaces make the firewall the authoritative default gateway for VLANs.
- Cisco IOS router-based hosts require the right gateway or routing for return traffic.
- App-ID can affect how web payloads are handled if a service is identified too narrowly.
- DHCP on the switch works best when gateway IPs are aligned with the firewall subinterfaces.

---

## What This Lab Demonstrates

- Palo Alto App-ID policy design.
- VLAN trunking and subinterface routing.
- NAT for branch internet breakout.
- Cisco IOS switch DHCP and segmentation.
- Cisco IOS HTTP service validation.
- Real-world troubleshooting in a multi-vendor branch design.

---

## Repository Structure

```text
enterprise-branch-lab/
├── paloalto/
├── cisco-switch/
├── isp-router/
├── screenshots/
├── captures/
└── README.md
```

---

## Notes

- This lab is intentionally built for hands-on validation, not just diagramming.
- The App-ID fix is part of the final validated policy set.
- The configuration assumes EVE-NG with Cisco IOS nodes acting as end hosts and services.
- Use screenshots and packet captures to prove the workflow in your portfolio.

---

## License

For educational and portfolio use.
