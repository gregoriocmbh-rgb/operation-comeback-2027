# Network Implementation & Configuration Guide

**Project:** Operation Comeback 2027
**Lab:** TTP 7.6 — Default Routing & Simulated ISP
**Related change:** CHG-NET-001 — Implement Default Routing to Simulated ISP
**Environment:** Cisco Modeling Labs (CML), Cisco IOSv routers, Alpine Linux hosts
**Document status:** As-built guide based on lab notes and verified test results
**Version:** 1.0

## 1. Purpose and Scope

This document describes the addressing, router configurations, host settings, and verification procedures used in the simulated HQ–Branch network with an ISP router. It complements the CHG-NET-001 change record; it does not replace the change's risk assessment, approval, rollback, or closure sections.

The source was the author's `Initial Configuration.txt` working notes. Where commands were missing from those notes, the additional commands below are based on the configurations and routing tables verified during CHG-NET-001. The commands shown are a **reconstructed configuration guide**, not a verbatim running-config export.

## 2. Logical Topology

```text
PC-HQ                  R1-HQ                  ISP-R1
10.10.10.10/24         Gi0/0 10.10.10.1       Gi0/0 203.0.113.2/30
   |                   Gi0/1 10.10.12.1/30     Lo0   198.51.100.1/32
   +--- Gi0/0 R1-HQ ---+  |              Gi0/3 203.0.113.1/30 ---+
                          |
                      10.10.12.0/30
                          |
                      Gi0/0 R2-BRANCH
                      10.10.12.2/30
                      Gi0/1 10.10.20.1/24
                          |
                       PC-BRANCH
                       10.10.20.10/24
```

> For the authoritative visual topology, refer to [final topology](../evidence/CHG-NET-001-Final-Topology.png.png) in the change record.

## 3. IP Addressing Plan

| Device | Interface | IP address / prefix | Purpose |
|---|---|---|---|
| PC-HQ | eth0 | 10.10.10.10/24 | HQ workstation |
| R1-HQ | Gi0/0 | 10.10.10.1/24 | HQ default gateway |
| R1-HQ | Gi0/1 | 10.10.12.1/30 | Link to R2-BRANCH |
| R2-BRANCH | Gi0/0 | 10.10.12.2/30 | Link to R1-HQ |
| R2-BRANCH | Gi0/1 | 10.10.20.1/24 | Branch default gateway |
| PC-BRANCH | eth0 | 10.10.20.10/24 | Branch workstation |
| R1-HQ | Gi0/3 | 203.0.113.1/30 | Link to simulated ISP |
| ISP-R1 | Gi0/0 | 203.0.113.2/30 | Link to R1-HQ |
| ISP-R1 | Loopback0 | 198.51.100.1/32 | Simulated external destination |

**Host gateways:** PC-HQ → `10.10.10.1`; PC-BRANCH → `10.10.20.1`.

> The 203.0.113.0/24 and 198.51.100.0/24 address blocks are documentation ranges used here for simulation, not public Internet connectivity.

## 4. Router Configuration

### 4.1 R1-HQ

```cisco
configure terminal
hostname R1-HQ
!
interface GigabitEthernet0/0
 ip address 10.10.10.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.10.12.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/3
 description LINK_TO_ISP-R1_Gi0/0
 ip address 203.0.113.1 255.255.255.252
 no shutdown
!
ip route 10.10.20.0 255.255.255.0 10.10.12.2
ip route 0.0.0.0 0.0.0.0 203.0.113.2
end
copy running-config startup-config
```

**Routing intent:** the specific route reaches Branch via R2-BRANCH; the default route forwards otherwise unknown destinations to ISP-R1.

### 4.2 R2-BRANCH

```cisco
configure terminal
hostname R2-BRANCH
!
interface GigabitEthernet0/0
 ip address 10.10.12.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.10.20.1 255.255.255.0
 no shutdown
!
ip route 10.10.10.0 255.255.255.0 10.10.12.1
ip route 0.0.0.0 0.0.0.0 10.10.12.1
end
copy running-config startup-config
```

**Routing intent:** HQ traffic follows its specific static route; external traffic follows the default route toward R1-HQ.

### 4.3 ISP-R1

```cisco
configure terminal
hostname ISP-R1
!
interface GigabitEthernet0/0
 description LINK_TO_R1-HQ_Gi0/3
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface Loopback0
 description SIMULATED_EXTERNAL_NETWORK
 ip address 198.51.100.1 255.255.255.255
!
ip route 10.10.10.0 255.255.255.0 203.0.113.1
ip route 10.10.20.0 255.255.255.0 203.0.113.1
end
copy running-config startup-config
```

**Routing intent:** the two static routes provide the return path to HQ and Branch. A default route is not required for the destinations in this lab.

## 5. Alpine Linux Host Configuration

The original notes included both temporary `ip` commands and an `/etc/network/interfaces` alternative. **These are alternative configuration methods, not steps that must both be applied.** Confirm the host interface name before use; this lab used `eth0`.

### 5.1 PC-HQ — temporary configuration

```sh
hostname PC-HQ
ip addr add 10.10.10.10/24 dev eth0
ip route add default via 10.10.10.1 dev eth0
ip addr show dev eth0
ip route
ping -c 3 10.10.10.1
```

### 5.2 PC-BRANCH — temporary configuration

```sh
hostname PC-BRANCH
ip addr add 10.10.20.10/24 dev eth0
ip route add default via 10.10.20.1 dev eth0
ip addr show dev eth0
ip route
ping -c 3 10.10.20.1
```

> Run commands with sufficient privileges. If an address or default route already exists, inspect the current state before adding another. The temporary commands do not establish persistence across a restart.

### 5.3 Persistent host configuration — alternative

The source notes proposed editing `/etc/network/interfaces` and restarting the networking service. The following illustrates that approach; **persistent host configuration was not separately verified in the CHG-NET-001 final validation**.

**PC-HQ:**

```text
auto eth0
iface eth0 inet static
    address 10.10.10.10/24
    gateway 10.10.10.1
```

**PC-BRANCH:**

```text
auto eth0
iface eth0 inet static
    address 10.10.20.10/24
    gateway 10.10.20.1
```

**Suggested verification after applying a compatible persistent configuration:**

```sh
rc-service networking restart
ip addr show dev eth0
ip route
```

## 6. Verification Commands

Run on the applicable Cisco router:

```cisco
show ip interface brief
show ip route
show ip arp
show interfaces GigabitEthernet0/3
```

Use `show interfaces GigabitEthernet0/3` specifically on R1-HQ for the simulated ISP link. On other devices, substitute the relevant interface.

**Expected routing-table highlights:**

| Device | Route | Next hop / interface |
|---|---|---|
| R1-HQ | `S* 0.0.0.0/0` | 203.0.113.2 |
| R1-HQ | `S 10.10.20.0/24` | 10.10.12.2 |
| R2-BRANCH | `S* 0.0.0.0/0` | 10.10.12.1 |
| R2-BRANCH | `S 10.10.10.0/24` | 10.10.12.1 |
| ISP-R1 | `S 10.10.10.0/24` | 203.0.113.1 |
| ISP-R1 | `S 10.10.20.0/24` | 203.0.113.1 |
| ISP-R1 | `C 198.51.100.1/32` | Loopback0 |

## 7. Connectivity and Regression Tests

Run from the named Alpine Linux host:

| Test ID | Source | Command | Recorded result |
|---|---|---|---|
| VAL-01 | PC-HQ | `ping -c 5 198.51.100.1` | PASS, 5/5, 0% loss |
| VAL-02 | PC-BRANCH | `ping -c 5 198.51.100.1` | PASS, 5/5, 0% loss |
| REG-01 | PC-HQ | `ping -c 5 10.10.20.10` | PASS, 5/5, 0% loss |
| REG-02 | PC-BRANCH | `ping -c 5 10.10.10.10` | PASS, 5/5, 0% loss |

**Evidence:** Refer to the four ping screenshots and three router routing-table screenshots in [Section 9 of the change record](Change-Management-Record.md#9-evidence).

## 8. Troubleshooting Notes

**CML link not running:** During implementation, the R1-HQ Gi0/3 link initially remained down even though the interface had an IP address and `no shutdown`. CML Workbench link statistics showed the corresponding link was not running. Enabling it restored the operational link state.

**Missing return path:** End-host pings to the simulated external loopback initially failed despite successful forwarding toward the ISP. ISP-R1 required static routes back to both `10.10.10.0/24` and `10.10.20.0/24`.

**Routing principle:** The router selects the longest matching prefix. A specific route to an internal LAN takes precedence over the default route.

## 9. Configuration and Documentation Status

- [x] Router interface and routing configuration reviewed against lab results
- [x] R1-HQ routing table verified
- [x] R2-BRANCH routing table verified
- [x] ISP-R1 routing table verified
- [x] Four connectivity and regression tests passed
- [x] Running configuration saved to startup configuration on all three routers
- [x] Supporting screenshots collected in CHG-NET-001
- [ ] Configuration persistence after router restart tested
- [ ] Persistent Alpine host settings independently validated

## 10. Source Notes and Editorial Changes

This guide was derived from `Initial Configuration.txt`. It retains the original device roles, interface addresses, static-routing intent, Linux host commands, and verification commands while reorganizing them into an implementation sequence.

Editorial corrections made:

1. Added the R1-HQ default route to `203.0.113.2`, verified in the lab but absent from the source notes.
2. Added the ISP-R1 return route to `10.10.10.0/24`, verified in the lab but absent from the source notes.
3. Corrected inconsistent router prompts in the source notes (including R1-HQ prompts within the R2-BRANCH section).
4. Removed command-entry fragments and duplicate prompt text for readability.
5. Clearly separated temporary Alpine IP configuration from the proposed persistent alternative.

**Limitations:** This document is not a full `show running-config` backup. Do not use it as proof of host persistence or router restart testing. Consult the saved CML topology and the CHG-NET-001 evidence for the final verified state.
