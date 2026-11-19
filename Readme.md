# Multi-Site Enterprise Network Design & Implementation

A comprehensive multi-site enterprise network built and simulated in **Cisco Packet Tracer**. This architecture connects a centralized Headquarters (HQ) with four remote branch offices and a simulated public Internet edge, incorporating Layer-2 switching, dynamic Layer-3 routing, high availability, centralized application services, perimeter NAT/PAT, departmental access controls, and Site-to-Site IPsec VPN encryption.

---

## Architecture Overview

```text
                        [ INTERNET-SERVER ]
                          (203.0.113.10)
                                 |
                           [ ISP Router ]
                           (200.1.1.2/30)
                                 |
                           [  EDGE-RTR  ]  <-- NAT/PAT Overload (200.1.1.1)
                                 |
                       [ Central / WAN Router ]  <-- OSPF + Site-to-Site IPsec VPN
                      /     |           |      \
                     /      |           |       \
          [ Core01 ]---[ Core02 ]       |        \
         (HSRP Active) (HSRP Stby)      |         \
              \           /             |          \
         [ HQ Access Switches ]    [Branch-1]  [Branch-2]  [Branch-3]  [Branch-4]
         (VLANs 10, 11, 12, 20)   (.13.0/24)  (.14.0/24)  (.15.0/24)  (.16.0/24)
```

---

## IP Addressing & VLAN Scheme

### Headquarters (HQ) Subnets & Gateways
| VLAN ID | Department / Role | Network Subnet | Core01 SVI | Core02 SVI | HSRP Virtual Gateway |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Sales | `192.168.10.0/24` | `192.168.10.252` | `192.168.10.253` | `192.168.10.254` |
| **VLAN 11** | HR | `192.168.11.0/24` | `192.168.11.252` | `192.168.11.253` | `192.168.11.254` |
| **VLAN 12** | Accounts | `192.168.12.0/24` | `192.168.12.252` | `192.168.12.253` | `192.168.12.254` |
| **VLAN 20** | Servers | `172.10.10.0/24` | `172.10.10.252` | `172.10.10.253` | `172.10.10.254` |

### Remote Branches & Infrastructure Links
| Segment / Link | Network / IP Assignment | Notes |
| :--- | :--- | :--- |
| **Branch-1 LAN** | `192.168.13.0/24` | DHCP via Helper Address; IPsec VPN to HQ Sales |
| **Branch-2 LAN** | `192.168.14.0/24` | DHCP via Helper Address; IPsec VPN to HQ Sales |
| **Branch-3 LAN** | `192.168.15.0/24` | DHCP via Helper Address; IPsec VPN to HQ Sales |
| **Branch-4 LAN** | `192.168.16.0/24` | DHCP via Helper Address; IPsec VPN to HQ Sales |
| **Core01 ↔ Central Router** | `10.1.1.0/24` | Primary OSPF routed uplink |
| **Core02 ↔ Central Router** | `10.2.2.0/24` | Redundant OSPF routed uplink |
| **EDGE-RTR ↔ ISP** | `200.1.1.1` ↔ `200.1.1.2` | Outside WAN interface for PAT (`200.1.1.1`) |
| **External Web/Test Server** | `203.0.113.10` | Simulated public Internet host |

### Centralized Application Servers (VLAN 20)
| Service | IP Address | Protocols / Details |
| :--- | :--- | :--- |
| **Web Server** | `172.10.10.1` | HTTP (`http://www.company.com`) |
| **DNS Server** | `172.10.10.2` | Resolves `www.company.com` → `172.10.10.1` |
| **Email Server** | `172.10.10.3` | SMTP / POP3 (`@company.com`) |

---

## Key Implemented Technologies

### 1. Layer-2 Switching & Loop Prevention
* **VLAN Segmentation:** Logically isolates HQ departments (`Sales`, `HR`, `Accounts`) and centralized infrastructure (`Server`) across shared physical switches.
* **IEEE 802.1Q Trunking:** Carries tagged traffic for VLANs `10, 11, 12, 20` across inter-switch links.
* **Spanning Tree Protocol (PVST):** Prevents Layer-2 switching loops across redundant switch topologies by maintaining a loop-free forwarding tree.
* **LACP EtherChannel:** Bundles multiple physical Ethernet links into a single logical `Port-Channel` (`Po1`) for increased bandwidth and link-level fault tolerance.

### 2. Inter-VLAN Routing & High Availability
* **Switched Virtual Interfaces (SVIs):** Multilayer core switches (`Core01` and `Core02`) provide Layer-3 gateway routing between all internal VLANs.
* **HSRP Gateway Redundancy:** Provides First-Hop Redundancy using `.254` virtual gateways. `Core01` acts as the Active router (`Priority 110` with `preempt`), while `Core02` acts as Standby (`Priority 100`), ensuring automatic failover and failback.
* **Redundant Core Uplinks:** Both `Core01` (`10.1.1.0/24`) and `Core02` (`10.2.2.0/24`) maintain independent OSPF adjacencies with the Central Router.

### 3. Enterprise WAN & Dynamic Routing
* **OSPF Dynamic Routing:** Dynamically exchanges routes across HQ and all four remote branches (`Branch-1` through `Branch-4`) with full neighbor adjacencies (`FULL` state).
* **Default Route Propagation:** `EDGE-RTR` routes unknown external traffic to the ISP (`200.1.1.2`), and OSPF redistributes the default route (`0.0.0.0/0` Type E2) down to all remote branch routers.

### 4. Centralized Network & Application Services
* **Centralized DHCP & DHCP Relay:** Automatically assigns IP addresses, subnet masks, default gateways, and DNS server addresses to all clients. Remote branch routers use `ip helper-address` to forward broadcast DHCP requests across Layer-3 boundaries.
* **DNS, HTTP & Email:** Hosts internal corporate web services (`www.company.com`), hostname resolution, and inter-departmental email communication using SMTP and POP3.

### 5. Network Security & Perimeter Translation
* **Extended ACLs (Department Isolation):** Enforces strict segmentation between `Sales`, `HR`, and `Accounts` (blocking direct peer-to-peer department traffic) while permitting necessary access to centralized servers in `VLAN 20`.
* **Switchport Security:** Restricts access ports to `maximum 1` MAC address using `sticky` MAC learning and `shutdown` violation mode. Unused switch ports are administratively disabled.
* **NAT / PAT (Overload):** Configured on `EDGE-RTR` to translate all internal HQ and branch private subnets (`192.168.10.0/24`–`192.168.16.0/24` and `172.10.10.0/24`) to a single public-facing outside IP (`200.1.1.1`).
* **Site-to-Site IPsec VPN:** Secures inter-site traffic between HQ Sales (`192.168.10.0/24`) and all four branch networks (`192.168.13.0/24`–`192.168.16.0/24`) using ISAKMP (`QM_IDLE ACTIVE`) and ESP Tunnel Mode (`esp-aes` encryption + `esp-sha-hmac` integrity).

---

## Verification & Testing Guide

Use the following Cisco IOS CLI commands and functional tests to verify each subsystem in Cisco Packet Tracer:

| Feature | Target Device | Verification Command / Action | Expected Result |
| :--- | :--- | :--- | :--- |
| **VLANs & Trunks** | Access / Core Switches | `show vlan brief`<br>`show interfaces trunk` | VLANs `10, 11, 12, 20` active; trunks using `802.1q` |
| **STP & EtherChannel** | Core / Dist Switches | `show spanning-tree vlan 10`<br>`show etherchannel summary` | Loop-free Root/Designated/Blocking ports; `Po1(SU)` with bundled `(P)` ports |
| **SVIs & HSRP** | `Core01`, `Core02` | `show ip interface brief`<br>`show standby brief` | SVIs `up/up`; `Core01` = `Active` (110), `Core02` = `Standby` (100) |
| **HSRP Failover** | `Core01` → `Core02` | Shut down `Vlan10` on `Core01`, then run `show standby brief` on `Core02` | `Core02` transitions to `Active`; restores to `Standby` when `Core01` recovers |
| **OSPF Routing** | Central / Branch Routers | `show ip ospf neighbor`<br>`show ip route ospf` | All peers in `FULL` state; `O` and `O*E2` routes populated |
| **DHCP & Relay** | HQ & Branch PCs | `ipconfig` (or Desktop → DHCP)<br>`show running-config` (Branch) | Valid IP lease received; `ip helper-address` configured on branch interfaces |
| **DNS, Web & Email** | Client PCs | Browser: `http://www.company.com`<br>Email client: Send/Receive | Hostname resolves to `172.10.10.1`, webpage loads, SMTP/POP3 mail succeeds |
| **ACL Isolation** | Sales PC → HR PC / Server | `ping <HR-PC-IP>` vs `ping 172.10.10.1`<br>`show access-lists` | Inter-department ping **fails** (blocked by ACL); server ping **succeeds** |
| **Port Security** | Access Switch | `show port-security interface fa0/x`<br>`show interfaces status` | `Secure-up`, `Sticky MAC` learned; rogue MAC triggers `Secure-shutdown` |
| **Internet & PAT** | `EDGE-RTR` & Client PCs | `ping 203.0.113.10`<br>`show ip nat translations` | External server reachable; multiple inside locals mapped to `200.1.1.1` |
| **IPsec VPN** | Central Router | `show crypto isakmp sa`<br>`show crypto ipsec sa` | 4 tunnels `QM_IDLE ACTIVE`; `#pkts encrypt` & `#pkts decrypt` increment on ping |

---

## How to Run

1. Clone this repository or download the `.pkt` topology file:
   ```bash
   git clone https://github.com/<your-username>/<your-repo-name>.git
   ```
2. Open the `.pkt` file using **Cisco Packet Tracer** (v8.0 or later recommended).
3. Allow 30–60 seconds for STP convergence, OSPF neighbor adjacencies, and HSRP elections to stabilize (or click **Fast Forward Time** in Packet Tracer).
4. Follow the **Verification & Testing Guide** above to inspect routing tables, test failover, and verify encrypted VPN traffic.