# TechPeak Enterprise Network - Troubleshooting Workbook & Incident Log

**Incident**: Monday Morning Post-Change Window Outage  
**Role**: Lead Incident & Network Triage Engineer  
**Credentials**: Console & Enable Password: `Lab#2026` | Domain: `techpeak.lab`  
**Topology File**: `TechPeak-Troubleshooting-Student.pkt`  

---

## Rules of Engagement
- **Show before Change**: Prove root cause with non-destructive `show` commands before altering state.
- **Root-Cause Remediation**: Fix underlying protocol/configuration faults; do not mask symptoms with workarounds, static routes, or ACL bypasses.
- **Minimal Footprint**: Modify only the incorrect statements on affected devices.
- **Persistence**: Save with `write memory` after every verified remediation.
- **Final Acceptance**: Full reachability across all branches, data centres, and simulated internet (`203.0.113.1`).

---

## Ticket Progress Matrix

| Ticket # | Description | Domain | Affected Device | Status | Root Cause Summary |
|---|---|---|---|---|---|
| **1** | PC1 cannot reach VLAN 10 gateway | HQ LAN / Access | `HQ-SRV-SW` / `PC1` | **RESOLVED** | Cascading failure from Ticket 3: DHCP unreachable due to trunk pruning |
| **2** | VLAN 20 users on floor 4 (PC6) have no connectivity | HQ LAN / Access | TBD | OPEN | Pending Triage |
| **3** | HQ servers (`10.10.2.x`) unreachable from users | HQ Server Farm | `HQ-SRV-SW` | **RESOLVED** | Trunk uplinks (Gig0/1-2) pruned VLAN 30 (`allowed vlan 40` only) |
| **4** | PC4 is dead | HQ Access | TBD | OPEN | Pending Triage |
| **5** | Finance (PC5) cannot reach its gateway | HQ Access | TBD | OPEN | Pending Triage |
| **6** | WAN-B (R4) lost its link to HQ-CORE2 | Core / WAN | TBD | OPEN | Pending Triage |
| **7** | Lagos office is isolated from WAN | WAN / Branch | TBD | OPEN | Pending Triage |
| **8** | HQ cannot reach branch networks (routers up) | Routing / OSPF-EIGRP | TBD | OPEN | Pending Triage |
| **9** | Branch offices cannot reach Internet (HQ can) | WAN / Edge NAT | TBD | OPEN | Pending Triage |
| **10** | DC-B servers unreachable from Lagos | DC / WAN | TBD | OPEN | Pending Triage |
| **11** | Manchester users cannot reach gateway | Branch LAN | TBD | OPEN | Pending Triage |
| **12** | HQ-ACC1 switch cannot be managed from remote subnets | Switch Management | TBD | OPEN | Pending Triage |
| **13** | Console messages on HQ-CORE2 mention HSRP | Core Redundancy | TBD | OPEN | Pending Triage |
| **14** | DCA-SRV1 cannot be reached from other sites | Data Centre A | TBD | OPEN | Pending Triage |
| **15** | DCA-SW2 has lost uplink redundancy | Data Centre A | TBD | OPEN | Pending Triage |
| **16** | PC3 cannot reach its gateway | HQ Access | TBD | OPEN | Pending Triage |

---

## Detailed Incident Triage Logs

### Ticket 3: HQ Servers (`10.10.2.x`) Unreachable from Users
- **Symptom**: All HQ servers in subnet `10.10.2.0/24` (DHCP, DNS, Intranet) are completely unreachable across the enterprise. Direct ping from default gateway `HQ-CORE1` to server `10.10.2.10` fails (0/5 replies).
- **Impact**: High / Enterprise-wide outage. Secondary services (DHCP address allocation) failing network-wide.
- **Affected Device**: `HQ-SRV-SW`
- **Diagnostic Commands Used**:
  - `HQ-CORE1# show ip int brief | include Vlan` — Verified `Vlan30` is configured with `10.10.2.2/24` and status `up/up`.
  - `HQ-CORE1# ping 10.10.2.10` — Failed (0/5). Confirmed L2 failure within same subnet.
  - `HQ-SRV-SW# show vlan brief` — Port `Fa0/1` assigned to `VLAN 30 (HQ-SERVERS)`.
  - `HQ-SRV-SW# show interfaces trunk` — **Identified Defect**:
    ```text
    Port        Vlans allowed on trunk
    Gig0/1      40
    Gig0/2      40
    ```
- **Root Cause**:
  Uplink trunk interfaces `Gig0/1` and `Gig0/2` on switch `HQ-SRV-SW` had restrictive VLAN filtering configured (`switchport trunk allowed vlan 40`). VLAN 30 was omitted, blocking all server farm traffic across the core distribution layer.
- **Remediation Commands**:
  ```ios
  HQ-SRV-SW# configure terminal
  HQ-SRV-SW(config)# interface range Gig0/1 - 2
  HQ-SRV-SW(config-if-range)# switchport trunk allowed vlan add 30
  HQ-SRV-SW(config-if-range)# end
  HQ-SRV-SW# write memory
  ```
- **Verification**:
  - `HQ-SRV-SW# show interfaces trunk` confirmed allowed VLANs: `30, 40`.
  - `HQ-CORE1# ping 10.10.2.10` succeeded with 100% success rate (`!!!!!`).

---

### Ticket 1: PC1 Cannot Reach VLAN 10 Gateway
- **Symptom**: PC1 fails to obtain network configuration via DHCP (receives APIPA `169.254.x.x`), rendering it unable to communicate with its default gateway (`10.10.0.1`).
- **Impact**: Floor 1 / Finance user isolated from corporate network.
- **Affected Devices**: `HQ-ACC1`, `HQ-CORE1`, `HQ-SRV-SW`
- **Diagnostic Commands Used**:
  - `PC1> ipconfig /all` — Returned APIPA auto-configuration address (`169.254.x.x`).
  - `HQ-ACC1# show vlan brief` — Verified `Fa0/1` is active in `VLAN 10 (FINANCE)`.
  - `HQ-ACC1# show interfaces trunk` — Verified uplinks `Gig0/1` & `Gig0/2` permit and forward VLAN 10.
  - `HQ-CORE1# show run | begin interface Vlan10` — Verified `ip helper-address 10.10.2.10` points to HQ DHCP server.
  - `PC1 (Static IP Test)`: Temporarily assigned `10.10.0.50/24` with GW `10.10.0.1`; pings to `10.10.0.1` and `10.10.0.2` succeeded, proving L2/L3 path from PC1 to `HQ-CORE1` was intact.
- **Root Cause**:
  Cascading failure triggered by Ticket 3. PC1 was fully operational at Layer 2, but its DHCP relay requests forwarded by `HQ-CORE1` to `10.10.2.10` were dropped at `HQ-SRV-SW` due to trunk pruning.
- **Remediation**:
  Resolved via remediation of Ticket 3 (`HQ-SRV-SW` trunk allowed list update).
- **Verification**:
  - Toggled PC1 IP configuration to DHCP: received `10.10.0.x`, subnet mask `255.255.255.0`, default gateway `10.10.0.1`.
  - `PC1> ping 10.10.0.1` — Success (4/4 replies).
