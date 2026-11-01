# Packet-Tracer-Project-6---FHRP-HSRP-

![Project 6 Screenshot](https://github.com/pinoechalle-lang/Packet-Tracer-Project-6---FHRP-HSRP-/blob/main/Screenshot%202026-10-30%20000423.png)

## Requirement
---

**Company:** Larkspur Logistics, a regional distribution company.

Larkspur Logistics is opening a new branch office with three departments. The branch uses two routers and two access switches. During the planning meeting, the operations manager raised one concern:

> "In our last office, one router failure cut off the whole floor, and nobody could reach the other departments until IT showed up. We can't have that here."

The network engineer (you) is asked to build a branch network where **a single router or router-link failure does not interrupt communication between departments**.

---
### Business Requirements

| # | Requirement |
|---|---|
| B1 | Three departments must be isolated into their own VLANs. |
| B2 | Departments must be able to communicate with each other (inter-VLAN routing). |
| B3 | A single router or router-link failure must not interrupt inter-VLAN communication. |
| B4 | Failover must be automatic, with no manual intervention. |
| B5 | Hosts must not need reconfiguration when a failover happens. |

### Technical Requirements

| # | Requirement |
|---|---|
| T1 | Use router-on-a-stick (802.1Q subinterfaces) on both routers for inter-VLAN routing. |
| T2 | Use HSRP version 2 with a single virtual gateway per VLAN. |
| T3 | R1 must be the **Active** router and R2 the **Standby** router for all VLANs. |
| T4 | Enable `preempt` on both routers so R1 reclaims the Active role after recovery. |
| T5 | Use interface tracking so R1 gives up the Active role when it loses a path that R2 still has. |
| T6 | Both access switches must be dual-homed, with one trunk to each router. |
| T7 | Trunks must use native VLAN 72 and allow only VLANs 10, 20, 30, and 72. |
| T8 | Hosts must use the HSRP virtual IP as their default gateway. |

### Constraint (Packet Tracer Limitation)

Packet Tracer does not support a configurable tracking `decrement`. It always uses the default of **10**. The priority gap between the two routers therefore has to be **smaller than 10**, or one failed tracked interface will never push the Active router below the Standby router. This lab uses **155 (R1)** and **150 (R2)**.

---

| Link | From | To |
|---|---|---|
| 1 | R1 g0/1 | SW-A1 g0/1 |
| 2 | R1 g0/2 | SW-A2 g0/1 |
| 3 | R2 g0/1 | SW-A1 g0/2 |
| 4 | R2 g0/2 | SW-A2 g0/2 |
| 5 | SW-A1 f0/1 | PCA-10 |
| 6 | SW-A2 f0/1 | PCA-20 |
| 7 | SW-A2 f0/2 | PCA-30 |

## Addressing Plan

| VLAN | Department | Network | Virtual Gateway (HSRP) | R1 (Active) | R2 (Standby) |
|---|---|---|---|---|---|
| 10 | Administration | 192.168.10.0/24 | 192.168.10.254 | 192.168.10.252 | 192.168.10.253 |
| 20 | Sales | 192.168.20.0/24 | 192.168.20.254 | 192.168.20.252 | 192.168.20.253 |
| 30 | Engineering | 192.168.30.0/24 | 192.168.30.254 | 192.168.30.252 | 192.168.30.253 |
| 72 | Native (unused) | N/A | N/A | N/A | N/A |

---

## HSRP Design

| Setting | R1 | R2 |
|---|---|---|
| Version | 2 | 2 |
| Priority (all groups) | 155 | 150 |
| Preempt | Enabled | Enabled |
| Group 10 tracks | g0/2.20, g0/2.30 | g0/2.20, g0/2.30 |
| Group 20 tracks | g0/1.10 | g0/1.10 |
| Group 30 tracks | g0/1.10 | g0/1.10 |

### Why track the other side?

Each router's subinterfaces are split across two switches. If R1 loses the link to one switch, it can still be Active on the other side, so hosts on that side would send traffic to a router that cannot reach the rest of the network. Tracking makes R1 lower its priority (155 to 145) when it loses the *other* side, so R2 (150) preempts and takes over all VLANs.

---

## Acceptance Tests

| # | Test | Expected Result |
|---|---|---|
| 1 | Run `show standby brief` on R1 and R2 (all links up) | R1 is Active (155) and R2 is Standby (150) for groups 10, 20, 30. |
| 2 | Ping between PCA-10, PCA-20, and PCA-30 | All pings succeed. |
| 3 | Remove the R1 to SW-A1 link | R1's priority drops to 145 on groups 20 and 30. R2 becomes Active for all groups. |
| 4 | Ping between all PCs again | All pings succeed through R2. |
| 5 | Restore the link | R1 preempts and returns to Active (155). |
| 6 | Repeat steps 3 to 5 with the R1 to SW-A2 link | R1's priority drops to 135 on group 10 (two tracked interfaces lost). R2 becomes Active, and pings succeed. |

## Verification Commands

| Command | Device | Purpose |
|---|---|---|
| `show standby brief` | R1, R2 | Priority, state, and Active/Standby peer per group. |
| `show standby <interface>` | R1, R2 | Full HSRP detail, including track state, for one subinterface. |
| `show ip interface brief` | R1, R2 | Confirms interface and subinterface status. |
| `show interfaces trunk` | SW-A1, SW-A2 | Confirms trunk ports, native VLAN, and allowed VLANs. |
| `show vlan brief` | SW-A1, SW-A2 | Confirms VLAN-to-port assignment. |

---

## Known Limitations

- Each access switch is a single point of failure for the hosts attached to it. HSRP protects the gateway, not the switch.
- The switches are not interconnected, so there is no Layer 2 loop and no STP dependency in this design.
- On real IOS, the tracking `decrement` can be set manually, and `preempt delay` is recommended. These are not available or reliable in Packet Tracer.

## Files

| File | Description |
|---|---|
| `P6 - FHRP (HSRP).pkt` | Packet Tracer project file |
| `P6 - FHRP (HSRP).txt` | Switch configurations (SW-A1, SW-A2) |
| `P6 - Router_Configuration.txt` | Router configurations (R1, R2) |
