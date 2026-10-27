# INC-L3-01 — HQ Inter-VLAN Connectivity Failure

**Project:** Operation Comeback 2027

**Environment:** Cisco Modeling Labs (CML)

**Technologies:** Cisco IOS, VLANs, IEEE 802.1Q, Router-on-a-Stick, ARP, ICMP, SSH

**Incident Status:** Resolved and validated

**Related Competency:** TTP 7.5 — Inter-VLAN Routing

---

## 1. Incident Overview

This incident involved a connectivity failure between two VLANs in a simulated enterprise network.

A workstation in VLAN 20 (OPERATIONS), identified as PC-OPS1, was unable to communicate with its default gateway or reach PC-CORP1, a workstation in VLAN 10 (CORPORATE).

The network used a Router-on-a-Stick architecture, with a Cisco router providing inter-VLAN routing through IEEE 802.1Q subinterfaces.

The investigation focused on verifying router subinterface configuration, switch trunk operation, gateway reachability, and ARP resolution.

**Root Cause:** An incorrect IEEE 802.1Q VLAN identifier was configured on the router subinterface responsible for VLAN 20.

**Resolution:** The VLAN encapsulation was corrected, connectivity was successfully restored, and the router configuration was saved.

**Outcome:** Inter-VLAN connectivity was validated through successful ICMP tests with zero packet loss.

## 2. Lab Environment and Network Topology

The incident was investigated in Cisco Modeling Labs (CML), using a simulated enterprise network with two VLANs and a Cisco router configured for inter-VLAN routing.

### 2.1 Network Components

| Device | Role | Description |
|---|---|---|
| R1 | Layer 3 Gateway | Cisco router providing inter-VLAN routing |
| SW01 | Layer 2 Switch | Cisco switch connecting both VLANs to R1 |
| PC-CORP1 | Corporate Workstation | Endpoint assigned to VLAN 10 |
| PC-OPS1 | Operations Workstation | Endpoint assigned to VLAN 20 |

### 2.2 VLAN and IP Addressing Plan

| Device / Interface | VLAN | IP Address | Default Gateway |
|---|---|---|---|
| PC-CORP1 | 10 | 10.20.0.10/24 | 10.20.0.1 |
| PC-OPS1 | 20 | 10.20.1.10/26 | 10.20.1.1 |
| R1 Gi0/0.10 | 10 | 10.20.0.1/24 | N/A |
| R1 Gi0/0.20 | 20 | 10.20.1.1/26 | N/A |

### 2.3 Logical Network Topology

```text
PC-CORP1 (VLAN 10)
10.20.0.10/24
       |
       | Access VLAN 10
       |
      SW01
       |
       | 802.1Q Trunk
       | Allowed VLANs: 10,20
       |
       R1
       | Gi0/0.10 - 10.20.0.1/24
       | Gi0/0.20 - 10.20.1.1/26
       |
      SW01
       |
       | Access VLAN 20
       |
PC-OPS1 (VLAN 20)
10.20.1.10/26
```

The router and switch are connected through a single IEEE 802.1Q trunk link. The diagram repeats SW01 to illustrate both VLAN paths; it represents one physical switch.

### 2.4 Inter-VLAN Routing Architecture

The network uses a Router-on-a-Stick architecture.

R1 provides Layer 3 connectivity between VLAN 10 and VLAN 20 through subinterfaces configured on GigabitEthernet0/0.

Each router subinterface requires the correct IEEE 802.1Q VLAN identifier and an IP address within the corresponding VLAN subnet.

All IP addresses in this report belong to an isolated, simulated lab environment and do not represent production infrastructure.

## 3. Reported Symptoms and Initial Connectivity Test

### 3.1 Reported Connectivity Failure

PC-OPS1, assigned to VLAN 20 (OPERATIONS), could not communicate with PC-CORP1 in VLAN 10 (CORPORATE).

During the initial investigation, PC-OPS1 was also unable to reach its own default gateway, `10.20.1.1`.

This indicated that troubleshooting should begin with local VLAN connectivity and gateway reachability before investigating inter-VLAN routing.

### 3.2 Initial Gateway Connectivity Test

**Source device:** PC-OPS1
**Destination:** R1 VLAN 20 gateway (`10.20.1.1`)
**Test method:** ICMP echo requests

**Command executed:**

```bash
ping 10.20.1.1
```

**Recorded result:**

```text
183 packets transmitted, 0 packets received, 100% packet loss
```

### 3.3 Initial Assessment

The test confirmed that PC-OPS1 could not communicate with its default gateway.

Because the workstation and gateway belong to the same IP subnet, successful communication requires Layer 2 connectivity and ARP resolution.

The failed ICMP test established the symptom but did not independently identify the root cause.

Further investigation was required to examine the router subinterface configuration, the switch trunk, and the workstation's ARP neighbor table.

## 4. Router Configuration Investigation

### 4.1 Router Access and Diagnostic Command

**Device:** R1
**Access method:** SSH using PuTTY
**Platform:** Cisco IOS

During the investigation, the router's running configuration was examined to verify the physical trunk interface and the subinterfaces responsible for inter-VLAN routing.

**Command executed:**

```text
show running-config
```

### 4.2 Recorded Router Configuration

The following configuration excerpt was recorded during the original troubleshooting session:

```cisco
interface GigabitEthernet0/0
 description TO_SW01_TRUNK
 no ip address
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/0.10
 description CORPORATE_GATEWAY
 encapsulation dot1Q 10
 ip address 10.20.0.1 255.255.255.0
!
interface GigabitEthernet0/0.20
 description OPERATIONS_GATEWAY
 encapsulation dot1Q 30
 ip address 10.20.1.1 255.255.255.192
```

### 4.3 Configuration Analysis

The router was configured with two subinterfaces for inter-VLAN routing:

- `GigabitEthernet0/0.10` served VLAN 10 (CORPORATE).
- `GigabitEthernet0/0.20` was intended to serve VLAN 20 (OPERATIONS).

The IP address configured on `GigabitEthernet0/0.20` was correct for the VLAN 20 subnet. However, the IEEE 802.1Q encapsulation was configured as:

```cisco
encapsulation dot1Q 30
```

This associated the subinterface with VLAN 30 instead of VLAN 20.

The numeric suffix `.20` in the subinterface name is an administrative identifier. It does not automatically assign the subinterface to VLAN 20.

The `encapsulation dot1Q` command determines which tagged VLAN traffic the subinterface processes.

### 4.4 Initial Troubleshooting Hypothesis

The configuration suggested a VLAN tagging mismatch between the switch trunk and the router subinterface.

If SW01 forwarded VLAN 20 traffic correctly while R1 expected VLAN 30 tags on the corresponding subinterface, the VLAN 20 workstation would be unable to communicate with its gateway.

The next investigation step was to verify the operational trunk configuration on SW01 before confirming the root cause.

## 5. Switch Trunk Verification

### 5.1 Switch Access and Diagnostic Command

**Device:** SW01
**Access method:** SSH using PuTTY
**Platform:** Cisco IOS

The switch trunk was examined to determine whether VLAN 20 traffic was correctly transported between SW01 and R1.

**Command executed:**

```text
show interfaces trunk
```

### 5.2 Recorded Trunk Output

The following output was recorded during the original troubleshooting session:

```text
Port        Mode             Encapsulation  Status        Native vlan
Gi0/0       on               802.1q         trunking      1

Port        Vlans allowed on trunk
Gi0/0       10,20

Port        Vlans allowed and active in management domain
Gi0/0       10,20

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/0       10,20
```

### 5.3 Trunk Analysis

The diagnostic output confirmed that:

- Interface `Gi0/0` was operating as an IEEE 802.1Q trunk.
- VLANs 10 and 20 were allowed on the trunk.
- Both VLANs were active on the switch.
- Both VLANs were in the spanning-tree forwarding state and were not pruned.

These findings supported the conclusion that SW01 was correctly transporting VLAN 20 traffic across the trunk.

However, an operational trunk does not independently guarantee end-to-end connectivity.

### 5.4 Troubleshooting Conclusion

The switch trunk configuration was consistent with the intended network design.

Combined with the incorrect `encapsulation dot1Q 30` configuration identified on R1, this evidence strengthened the hypothesis that the router's VLAN tagging mismatch was responsible for the connectivity failure.

Further validation through ARP troubleshooting was required to establish how the mismatch affected communication with the VLAN 20 gateway.

## 6. ARP Investigation

### 6.1 ARP Neighbor Table Verification

**Source device:** PC-OPS1
**Operating system:** Linux
**Target gateway:** 10.20.1.1

After the failed ICMP connectivity test, the workstation's ARP neighbor table was examined to determine whether PC-OPS1 could resolve the Layer 2 address of its default gateway.

**Command executed:**

```bash
ip neigh show
```

### 6.2 Recorded ARP Output

The following output was recorded before the configuration correction:

```text
10.20.1.1 dev eth0 used 0/0/0 probes 6 FAILED
```

The `FAILED` neighbor state indicated that the workstation had been unable to resolve the gateway's MAC address.

### 6.3 ARP Failure Analysis

PC-OPS1 and its default gateway were configured within the same IPv4 subnet.

Before sending an ICMP echo request to the gateway, the workstation needed to resolve the gateway's MAC address using ARP.

The unresolved ARP entry was consistent with the VLAN tagging mismatch identified on R1:

- SW01 forwarded VLAN 20 traffic over the IEEE 802.1Q trunk.
- R1's intended VLAN 20 subinterface was configured with `encapsulation dot1Q 30`.
- As a result, the router subinterface did not process frames tagged for VLAN 20 as intended.
- PC-OPS1 could not complete ARP resolution for its default gateway.

### 6.4 Diagnostic Conclusion

The failed gateway ping, unresolved ARP entry, operational switch trunk, and incorrect router encapsulation collectively supported the root-cause diagnosis.

**Confirmed root cause:** The router subinterface `GigabitEthernet0/0.20` was configured with VLAN 30 encapsulation instead of VLAN 20.

The next step was to correct the IEEE 802.1Q VLAN identifier and validate restored connectivity.

## 7. Configuration Correction

### 7.1 Corrective Action

The investigation identified an incorrect IEEE 802.1Q VLAN identifier on R1 subinterface `GigabitEthernet0/0.20`.

**Incorrect configuration:**

```cisco
encapsulation dot1Q 30
```

**Required configuration:**

```cisco
encapsulation dot1Q 20
```

The corrective action was to associate the router subinterface with VLAN 20, matching the VLAN transported by the switch trunk.

### 7.2 Configuration Commands

The following Cisco IOS command sequence reconstructs the corrective action. The exact command-entry transcript was not captured during the original incident.

```cisco
configure terminal
interface GigabitEthernet0/0.20
encapsulation dot1Q 20
end
```

The correction changed the VLAN tag associated with the subinterface without changing its IPv4 address or subnet mask.

### 7.3 Post-Correction Configuration Verification

The router configuration was verified after the correction using the following command:

```cisco
do show running-config interface GigabitEthernet0/0.20
```

The recorded verification confirmed the corrected encapsulation and gateway addressing:

```cisco
interface GigabitEthernet0/0.20
 description OPERATIONS_GATEWAY
 encapsulation dot1Q 20
 ip address 10.20.1.1 255.255.255.192
```

### 7.4 Expected Network Behavior

With the corrected encapsulation, R1 could process VLAN 20 frames arriving through the IEEE 802.1Q trunk.

This restored the intended Layer 2 association between the VLAN 20 workstation and its router gateway.

Connectivity tests were then required to confirm that ARP resolution, gateway reachability, and inter-VLAN communication had recovered.

## 8. Post-Correction Connectivity Validation

### 8.1 Default Gateway Reachability

**Source device:** PC-OPS1
**Destination:** 10.20.1.1 (VLAN 20 default gateway)
**Protocol:** ICMP

After correcting the IEEE 802.1Q encapsulation on R1, the gateway connectivity test was repeated.

```bash
ping 10.20.1.1
```

The recorded output showed successful ICMP echo replies, including sequence numbers 0 through 3.

This confirmed that PC-OPS1 could communicate with its default gateway after the correction.

### 8.2 Inter-VLAN Connectivity Test

**Source device:** PC-OPS1 (VLAN 20)
**Destination:** PC-CORP1 (VLAN 10)
**Destination IP:** 10.20.0.10

**Command executed:**

```bash
ping -c 4 10.20.0.10
```

**Recorded result:**

```text
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 6.846/7.705/8.526 ms
```

The successful test confirmed that traffic could be routed between VLAN 20 and VLAN 10.

### 8.3 ARP Neighbor Table Verification

After connectivity was restored, the ARP neighbor table on PC-OPS1 was examined again.

**Command executed:**

```bash
ip neigh show
```

**Recorded result (MAC address redacted for publication):**

```text
10.20.1.1 dev eth0 lladdr [REDACTED] used 0/0/0 probes 1 STALE
```

Unlike the earlier `FAILED` entry, the neighbor table now contained a resolved MAC address for the gateway.

The `STALE` state is a normal Linux neighbor-cache state indicating that the entry exists but its reachability has not been recently confirmed. It does not, by itself, indicate a connectivity failure.

### 8.4 Validation Summary

| Validation Check | Before Correction | After Correction |
|---|---|---|
| VLAN 20 gateway ping | Failed | Successful |
| Gateway ARP resolution | FAILED | MAC address resolved |
| VLAN 20 to VLAN 10 ping | Unavailable | 4/4 replies, 0% loss |
| Router VLAN 20 encapsulation | VLAN 30 | VLAN 20 |

The post-correction tests demonstrated that the configuration change restored gateway reachability and inter-VLAN communication.

These results were recorded in the simulated CML environment and do not represent production network measurements.

## 9. Configuration Persistence and Incident Closure

### 9.1 Saving the Router Configuration

After validating restored connectivity, the corrected running configuration was saved to the router's startup configuration.

**Device:** R1

**Command executed:**

```cisco
copy running-config startup-config
```

**Recorded confirmation:**

```text
[OK]
```

This confirmed that Cisco IOS reported a successful configuration save.

The router was not rebooted during the incident. Therefore, configuration persistence was confirmed through the save operation, not through a post-reboot validation test.

### 9.2 Incident Documentation in GLPI

The troubleshooting process was also documented in GLPI, an IT service management platform used in the lab environment.

**Ticket reference:** INC-L3-01 / GLPI Ticket #5

The service desk record documented the reported connectivity failure, investigation, identified root cause, corrective action, and successful validation.

The ticket was resolved, accepted by the simulated requester, and closed on October 25, 2026.

No personal information, user credentials, or private service desk details are included in this public report.

### 9.3 Final Incident Status

**Status:** Resolved and closed

**Root cause:** Incorrect IEEE 802.1Q encapsulation on router subinterface `GigabitEthernet0/0.20`.

**Corrective action:** Changed the VLAN encapsulation identifier from 30 to 20.

**Validation:** Gateway reachability and inter-VLAN connectivity were restored, with zero packet loss recorded during the four-packet inter-VLAN test.

**Configuration persistence:** Running configuration successfully saved to startup configuration.

**Service management:** Incident documented and closed in GLPI following requester acceptance.

## 10. Lessons Learned and Preventive Measures

### 10.1 Key Lessons Learned

This incident reinforced the importance of systematic troubleshooting across Layer 2 and Layer 3 network components.

**Key technical lessons:**

- A router subinterface name does not automatically determine its IEEE 802.1Q VLAN association.
- The `encapsulation dot1Q` configuration must match the VLAN identifier transported by the switch trunk.
- An operational switch trunk does not guarantee successful gateway communication.
- ARP neighbor states can provide valuable evidence when investigating local subnet connectivity failures.
- Configuration corrections should be followed by gateway and inter-VLAN connectivity tests.
- Successful troubleshooting includes documenting the incident and saving the corrected configuration.

### 10.2 Recommended Preventive Measures

The following measures are recommended for future lab deployments and network configuration changes:

1. **Configuration review:** Verify VLAN identifiers, router subinterfaces, IP addressing, and trunk settings before deployment.
2. **Standardized naming:** Use consistent VLAN names and interface descriptions to reduce configuration errors.
3. **Pre-change and post-change testing:** Establish a connectivity baseline and repeat relevant tests after changes.
4. **Configuration backup:** Save validated configurations and maintain version-controlled configuration templates with sensitive information removed.
5. **Troubleshooting documentation:** Record commands, observed results, root causes, corrective actions, and validation evidence.
6. **Monitoring:** Consider monitoring interface status and gateway reachability to helpdetect connectivity failures.

These are recommendations for future improvement and should not be interpreted as controls that were already implemented during this incident.

### 10.3 Professional Skills Demonstrated

This lab demonstrates practical experience with:

- Cisco IOS configuration analysis and troubleshooting.
- VLAN segmentation and IEEE 802.1Q trunking.
- Router-on-a-Stick inter-VLAN routing.
- ARP and ICMP diagnostic interpretation.
- Root-cause analysis and corrective action validation.
- Network incident documentation and IT service management using GLPI.

### 10.4 Evidence and Documentation Limitations

This report is based on recorded troubleshooting notes and command outputs from a simulated Cisco Modeling Labs environment.

The corrective command sequence in Section 7 was reconstructed from the documented configuration change because the original command-entry transcript was not captured.

Screenshots from the original troubleshooting session are not available. Any screenshots collected during a subsequent lab verification will be identified as later evidence rather than original incident captures.

The IP addresses are from an isolated simulated environment. Personal information, authentication credentials, and private infrastructure details are excluded from this public report.
