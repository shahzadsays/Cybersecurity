# Palo Alto Active/Passive HA & Layer 2 Failover Lab

A hands-on Palo Alto Networks lab that demonstrates Active/Passive HA, shared data-plane synchronization, path monitoring-based failover, and Layer 2 upstream switch failover behavior.

This lab is designed to show how a firewall pair can maintain stateful service continuity while failing over between two devices and two upstream LAN switch paths.

---

## Lab Objectives

- Build a Palo Alto Active/Passive HA pair.
- Synchronize firewall configuration between peers.
- Configure path monitoring for WAN and LAN failover detection.
- Validate firewall failover behavior in a Layer 2 switched environment.
- Demonstrate practical HA operations for portfolio and interview use.

Palo Alto documentation confirms that active/passive HA uses synchronized configuration and supports link and path monitoring for health-based failover. [web:78][web:79][web:83]

---

## Topology Overview

```text
Internet
   |
   v
PA-118 (Active) ---- HA1 / HA2 ---- PA-119 (Passive)
   |                                  |
   |                                  |
sw118                               sw119
   |                                  |
Trust VLAN                          Trust VLAN
   \___________ ISL / trunk __________/
```

### Addressing

| Component | IP / VLAN |
|---|---:|
| WAN / Untrust | `172.16.100.0/24` |
| Trust / LAN | `10.1.1.0/24` |
| PA-118 HA1 | `169.254.1.1/30` |
| PA-119 HA1 | `169.254.1.2/30` |
| PA-118 HA2 | `169.254.2.1/30` |
| PA-119 HA2 | `169.254.2.2/30` |

---

## Lab Design Notes

- Data-plane configuration is created on the active firewall and synchronized to the passive peer.
- HA1 is used for control/config synchronization.
- HA2 is used for state/session synchronization.
- Path monitoring is configured to detect failures that do not drop the physical link.
- The upstream switches must support the Layer 2 domain required for failover traffic continuity.

Palo Alto notes that only committed configuration synchronizes between HA peers, and that path monitoring is supported in active/passive deployments. [web:83][web:78]

---

## Firewall Configuration

### Shared Data-Plane Configuration

Configure the following once on the active firewall:

```bash
configure

set network interface ethernet ethernet1/1 layer3 ip 172.16.100.10/24
set zone UNTRUST network layer3 ethernet1/1

set network interface ethernet ethernet1/2 layer3 ip 10.1.1.254/24
set zone TRUST network layer3 ethernet1/2

set network virtual-router default interface [ ethernet1/1 ethernet1/2 ]

set network virtual-router default routing-table ip static-route default-route destination 0.0.0.0/0 nexthop ip-address 172.16.100.1

set rulebase nat rules VPC-INTRNET-NAT source-translation dynamic-ip-and-port interface-address interface ethernet1/1
set rulebase nat rules VPC-INTRNET-NAT to UNTRUST from TRUST source any destination any
```

### Notes

- The interface, zone, virtual router, route, and NAT settings are shared in the HA pair after synchronization.
- Make sure the configuration is committed on the active firewall before syncing to the peer.

Palo Alto documents that synchronized settings are committed and then propagated to the HA peer. [web:83]

---

## HA Base Configuration

### Primary Firewall: PA-118

```bash
configure

set network interface ethernet ethernet1/3 ha
set network interface ethernet ethernet1/4 ha

set deviceconfig high-availability enabled yes
set deviceconfig high-availability interface ha1 port ethernet1/3 ip-address 169.254.1.1 netmask 255.255.255.252
set deviceconfig high-availability interface ha2 port ethernet1/4 ip-address 169.254.2.1 netmask 255.255.255.252

set deviceconfig high-availability group group-id 1 peer-ip 169.254.1.2
set deviceconfig high-availability group group-id 1 state-synchronization enabled yes transport ethernet
set deviceconfig high-availability group group-id 1 mode active-passive

set deviceconfig high-availability group group-id 1 election-option device-priority 100 preemptive yes timers recommended
```

### Secondary Firewall: PA-119

```bash
configure

set network interface ethernet ethernet1/3 ha
set network interface ethernet ethernet1/4 ha

set deviceconfig high-availability enabled yes
set deviceconfig high-availability interface ha1 port ethernet1/3 ip-address 169.254.1.2 netmask 255.255.255.252
set deviceconfig high-availability interface ha2 port ethernet1/4 ip-address 169.254.2.2 netmask 255.255.255.252

set deviceconfig high-availability group group-id 1 peer-ip 169.254.1.1
set deviceconfig high-availability group group-id 1 state-synchronization enabled yes transport ethernet
set deviceconfig high-availability group group-id 1 mode active-passive

set deviceconfig high-availability group group-id 1 election-option device-priority 200 preemptive yes timers recommended
```

Palo Alto documentation confirms active/passive HA uses a peer relationship, configuration synchronization, and optional preemption based on device priority. [web:78][web:83]

---

## Path Monitoring

Path monitoring is required in this lab so the firewall can fail over when a virtual or upstream switch black-holes traffic without dropping the link.

### Monitoring Targets

- WAN gateway: `172.16.100.1`
- LAN core switch/router: `10.1.1.2`

### Configuration

Configure this on the active firewall using the GUI:

- Go to **Device > High Availability > Link and Path Monitoring**
- Path Group Name: `default`
- Failure Condition: `any`
- Add WAN destination: `172.16.100.1`
- Add LAN destination: `10.1.1.2`

### Commit and Sync

After configuration:

```bash
request high-availability sync-to-remote running-config
```

Palo Alto documents path monitoring as a supported HA failover condition and provides operational commands for syncing running configuration to the peer. [web:79][web:80][web:83]

---

## Layer 2 Switch Design

The lab uses two upstream Cisco Layer 2 switches to demonstrate failover across the trust side.

### Primary LAN Switch: sw118

```plaintext
interface Ethernet1/2
 description UPLINK-TO-PA118
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
```

### Secondary LAN Switch: sw119

```plaintext
interface Ethernet1/2
 description UPLINK-TO-PA119
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 no shutdown
```

### Inter-Switch Link

- Ensure the inter-switch link permits the VLANs used for the firewall trust uplinks.
- Confirm GARP and data-plane traffic can traverse during failover.

---

## Verification Checklist

- HA state shows one firewall as active and the other as passive.
- Configuration sync is successful.
- HA1 and HA2 links are up.
- Path monitoring destinations are reachable.
- NAT works from Trust to Untrust.
- Failover occurs when a monitored path fails.
- Traffic resumes on the standby firewall after failover.

---

## Test Scenarios

### Scenario 1: Active Firewall Failure
- Shut down or disconnect the active firewall.
- Confirm the passive firewall becomes active.
- Verify traffic resumes.

### Scenario 2: WAN Path Failure
- Simulate loss of the upstream gateway.
- Confirm path monitoring triggers failover.

### Scenario 3: LAN Path Failure
- Simulate loss of the core LAN switch path.
- Confirm failover occurs as expected.

---

## Skills Demonstrated

- Palo Alto Active/Passive HA
- Configuration synchronization
- HA1 and HA2 setup
- Path monitoring failover
- Layer 2 switch redundancy
- Stateful firewall operations
- Enterprise network resilience

---

## Repository Structure

```text
paloalto-ha-lab/
├── firewall/
├── switching/
├── monitoring/
└── README.md
```

---

## Notes

- Make configuration changes on the active firewall first.
- Commit before syncing to the passive peer.
- Path monitoring is especially important in virtualized or switch-based labs.
- Verify VLAN and trunk consistency between switches.

---

## License

For educational and portfolio use.
