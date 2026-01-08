# Goal 2: Subnetting and IP Configuration

Now that the topology is designed and devices are basically set up, we'll move to subnetting and IP addressing. This lab focuses on planning the IP address scheme, calculating subnets using Variable Length Subnet Masking (VLSM), assigning static IP addresses to interfaces, subinterfaces, loopbacks, and hosts, and verifying basic Layer 3 functionality. We'll use a private address space for the internal network and simulate public IPs for the WAN link to ISP-R. No dynamic routing or VLANs yet, that's in later goals, so we'll treat potential trunk links as access ports temporarily for IP assignment where needed (we'll convert them to trunks in Goal 3). This allows basic connectivity testing via pings within local segments.

This lab integrates skills like loopback interfaces for future OSPF router IDs, IP CEF for routing optimization, and basic hardening (e.g., no IP domain-lookup to prevent DNS resolution delays).

### Objective
- Design an IP addressing scheme using VLSM to efficiently allocate addresses for VLANs, point-to-point links, management, and loopbacks.
- Perform manual subnet calculations to understand the process (no tools needed in PT, but explained here for CCNA prep).
- Assign IPs to router interfaces, switch SVIs (preparing for inter-VLAN routing), subinterfaces on CME-R (demonstrating router-on-a-stick for voice/management), and end devices.
- Verify Layer 3 connectivity within local segments using pings and other commands.
- Document all assignments in tables for clarity.

This ensures a scalable addressing plan that avoids waste, supports redundancy (e.g., HSRP virtual gateways), and prepares for advanced features like OSPF (stable router IDs via loopbacks) and IPv6 (reserved space).

### Why This IP Scheme?
- **Efficiency with VLSM**: VLSM allows subdividing a large network into smaller subnets of varying sizes, reducing address waste. For example, host VLANs get /24 (254 usable IPs) for growth, while P2P links get /30 (2 usable IPs) since only two devices are needed.
- **Scalability and Organization**: VLAN-specific subnets make logical sense (e.g., 10.0.*.0 for VLAN (VLANID)), easy to remember and filter in ACLs. Management VLAN separates admin traffic for security.
- **Redundancy**: Virtual gateways (.1) for HSRP, with physical SVIs on MLSW1 (.2) and MLSW2 (.3) for failover.
- **Feature Coverage**: 
  - Subinterfaces on CME-R demonstrate 802.1Q encapsulation and router-on-a-stick, useful for scenarios without multilayer switches.
  - Loopbacks provide stable, always-up interfaces for OSPF router IDs (not affected by physical link failures).
  - Private IPs (RFC 1918) for internal, public simulation for WAN to prepare for NAT/PAT.
- **Best Practices**: Reserves space for expansion (e.g., more VLANs in 10.0.40.0–10.0.98.0), uses nice round numbers for readability.
- **Personalization**: Timezone from Goal 1 (WAT) is configured; passwords are simple here but will be strong in production.

### Detailed Subnetting Process
We'll start with the class A private network 10.0.0.0/16 (65,534 usable hosts). Using VLSM, we allocate based on needs, choosing "nice" numbers rather than sequential for memorability.

#### VLSM Explanation
VLSM builds on CIDR by allowing different mask lengths. Example calculation for a /24 from /16:
- Base: 10.0.0.0/16 = 10.0.0.0 – 10.0.255.255
- To get a /24: Borrow 8 bits (24-16=8), creating 256 possible /24 subnets (2^8).
- First /24: 10.0.0.0/24 (network 10.0.0.0, usable 10.0.0.1–254, broadcast 10.0.0.255).
- But we skip to 10.0.10.0/24 for VLAN 10 alignment.

Manual calculation for P2P /30:
- From a /24 (e.g., 10.0.1.0/24 = 256 addresses).
- Borrow 6 bits for /30 (30-24=6), creating 64 /30 subnets (2^6).
- First: 10.0.1.0/30 (network 10.0.1.0, usable 10.0.1.1–2, broadcast 10.0.1.3).
- Second: 10.0.1.4/30 (usable 10.0.1.5–6).

Requirements and Allocations:
- VLANs: 4 x /24 (254 hosts each) for growth beyond current ~10 devices.
- P2P: 3 x /30 (2 hosts each).
- Loopbacks: 5 x /32 (1 host each), from a separate block (1.1.1.0/24 simulation) for distinct router IDs.
- Reserved: Remaining in 10.0.0.0/16 for future (e.g., IPv6 dual-stack).

#### Subnet Details Table
| Subnet Name       | Network     | Netmask         | First Usable   | Last Usable   | Broadcast   | Total Hosts |
|-------------------|-------------|-----------------|----------------|---------------|-------------|-------------|
| VLAN10_Data       | 10.0.10.0   | 255.255.255.0   | 10.0.10.1      | 10.0.10.254   | 10.0.10.255 | 254         |
| VLAN20_Voice      | 10.0.20.0   | 255.255.255.0   | 10.0.20.1      | 10.0.20.254   | 10.0.20.255 | 254         |
| VLAN30_Wireless   | 10.0.30.0   | 255.255.255.0   | 10.0.30.1      | 10.0.30.254   | 10.0.30.255 | 254         |
| VLAN99_Management | 10.0.99.0   | 255.255.255.0   | 10.0.99.1      | 10.0.99.254   | 10.0.99.255 | 254         |
| P2P_MLSW1-EdgeR   | 10.0.1.0    | 255.255.255.252 | 10.0.1.1       | 10.0.1.2      | 10.0.1.3    | 2           |
| P2P_MLSW2-EdgeR   | 10.0.1.4    | 255.255.255.252 | 10.0.1.5       | 10.0.1.6      | 10.0.1.7    | 2           |
| WAN_EdgeR-ISPR    | 203.0.113.0 | 255.255.255.252 | 203.0.113.1    | 203.0.113.2   | 203.0.113.3 | 2           |

#### Loopbacks Table
| Subnet Name   | Network   | Netmask         | First Usable   | Last Usable   | Broadcast | Total Hosts |
|---------------|-----------|-----------------|----------------|---------------|-----------|-------------|
| MLSW1_Lo0     | 1.1.1.1   | 255.255.255.255 | 1.1.1.1        | 1.1.1.1       | N/A       | 1           |
| MLSW2_Lo0     | 2.2.2.2   | 255.255.255.255 | 2.2.2.2        | 2.2.2.2       | N/A       | 1           |
| CME-R_Lo0     | 3.3.3.3   | 255.255.255.255 | 3.3.3.3        | 3.3.3.3       | N/A       | 1           |
| Edge-R_Lo0    | 4.4.4.4   | 255.255.255.255 | 4.4.4.4        | 4.4.4.4       | N/A       | 1           |
| ISP-R_Lo0     | 5.5.5.5   | 255.255.255.255 | 5.5.5.5        | 5.5.5.5       | N/A       | 1           |

#### Specific Device IP Assignments Table
| Device       | Interface/Subinterface | IP Address      | Mask            | Notes                          |
|--------------|------------------------|-----------------|-----------------|--------------------------------|
| MLSW1        | VLAN 10 SVI            | 10.0.10.2       | 255.255.255.0   | HSRP physical (Active)         |
| MLSW1        | VLAN 20 SVI            | 10.0.20.2       | 255.255.255.0   |                                |
| MLSW1        | VLAN 30 SVI            | 10.0.30.2       | 255.255.255.0   |                                |
| MLSW1        | VLAN 99 SVI            | 10.0.99.2       | 255.255.255.0   |                                |
| MLSW1        | Fa0/1 (P2P to Edge-R) | 10.0.1.1        | 255.255.255.252 | No switchport                   |
| MLSW1        | Lo0                    | 1.1.1.1         | 255.255.255.255 | OSPF RID                       |
| MLSW2        | VLAN 10 SVI            | 10.0.10.3       | 255.255.255.0   | HSRP physical (Standby)        |
| MLSW2        | VLAN 20 SVI            | 10.0.20.3       | 255.255.255.0   |                                |
| MLSW2        | VLAN 30 SVI            | 10.0.30.3       | 255.255.255.0   |                                |
| MLSW2        | VLAN 99 SVI            | 10.0.99.3       | 255.255.255.0   |                                |
| MLSW2        | Fa0/1 (P2P to Edge-R) | 10.0.1.5        | 255.255.255.252 | No switchport                   |
| MLSW2        | Lo0                    | 2.2.2.2         | 255.255.255.255 | OSPF RID                       |
| Edge-R       | Gi0/0 (to MLSW1)       | 10.0.1.2        | 255.255.255.252 |                                |
| Edge-R       | Gi0/1 (to MLSW2)       | 10.0.1.6        | 255.255.255.252 | Redundancy                     |
| Edge-R       | Serial0/3/0 (to ISP-R) | 203.0.113.1     | 255.255.255.252 |                                |
| Edge-R       | Lo0                    | 4.4.4.4         | 255.255.255.255 | OSPF RID                       |
| CME-R        | Fa0/0.20 (Voice)       | 10.0.20.254     | 255.255.255.0   | Dot1Q encap, backup gateway    |
| CME-R        | Fa0/0.99 (Mgmt)        | 10.0.99.254     | 255.255.255.0   | Dot1Q encap                    |
| CME-R        | Lo0                    | 3.3.3.3         | 255.255.255.255 | OSPF RID                       |
| ISP-R        | Serial0/1/0 (to Edge-R)| 203.0.113.2     | 255.255.255.252 | Clock rate if DCE              |
| ISP-R        | Lo0                    | 5.5.5.5         | 255.255.255.255 | Test loopback                  |
| WLC          | Management             | 10.0.99.10      | 255.255.255.0   | Gateway 10.0.99.1 (virtual)    |
| PC1          | Ethernet               | 10.0.10.10      | 255.255.255.0   | Gateway 10.0.10.1              |
| PC2          | Ethernet               | 10.0.10.11      | 255.255.255.0   | Gateway 10.0.10.1              |
| Laptop       | Wireless               | 10.0.30.10      | 255.255.255.0   | Gateway 10.0.30.1 (temp static)|
| Server       | Ethernet               | 10.0.99.50      | 255.255.255.0   | Gateway 10.0.99.1              |

![Logical Topology](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/logical-topology.png>)

### Steps in Packet Tracer
1. **Plan and Document**: Review subnetting and tables above. Calculate one manually on paper (e.g., /30 binary: 10.0.1.0 = 00001010.00000000.00000001.00000000, mask /30 = last two bits host → usables .1 and .2).
2. **Configure Switch SVIs** (on MLSW1 and MLSW2 for inter-VLAN):
   - On MLSW1/MLSW2: `conf t`
     - `ip routing` (enable L3 routing)
     - `interface vlan 10`, `ip address 10.0.10.2/24` (MLSW1) or `10.0.10.3/24` (MLSW2), `no shut`
     - Repeat for VLAN 20: 10.0.20.2/24 or .3
     - VLAN 30: 10.0.30.2/24 or .3
     - VLAN 99: 10.0.99.2/24 or .3
     - Note: VLANs not created yet (Goal 3), but SVIs can be configured, they'll be down/down until VLANs exist and ports are active.
3. **Configure Router Interfaces**:
   - On Edge-R:
     - `int gi0/0`, `ip add 10.0.1.2 255.255.255.252`, `no shut` (to MLSW1)
     - `int gi0/1`, `ip add 10.0.1.6 255.255.255.252`, `no shut` (to MLSW2)
     - `int serial0/3/0`, `ip add 203.0.113.1 255.255.255.252`, `no shut`
     - `int lo0`, `ip add 4.4.4.4 255.255.255.255`
   - On CME-R:
     - `int fa0/0`, `no shut` (main int, no IP)
     - `int fa0/0.20`, `encapsulation dot1Q 20`, `ip add 10.0.20.254 255.255.255.0`
     - `int fa0/0.99`, `encapsulation dot1Q 99`, `ip add 10.0.99.254 255.255.255.0`
     - `int lo0`, `ip add 3.3.3.3 255.255.255.255`
   - On ISP-R:
     - `int serial0/1/0`, `ip add 203.0.113.2 255.255.255.252`, `no shut`, `clock rate 64000` (DCE)
     - `int lo0`, `ip add 5.5.5.5 255.255.255.255`
   - On MLSW1/MLSW2 (P2P interfaces—not SVIs):
     - MLSW1: `int fa0/1`, `no switchport`, `ip add 10.0.1.1 255.255.255.252`, `no shut`
     - MLSW2: `int fa0/1`, `no switchport`, `ip add 10.0.1.5 255.255.255.252`, `no shut`
     - `int lo0`, `ip add 1.1.1.1 255.255.255.255` (MLSW1) or `2.2.2.2 255.255.255.255` (MLSW2)
4. **Additional Hardening (All Routers/Switches)**:
   - `ip domain-name lab.local`
   - `no ip domain-lookup` (prevents hanging on typos)
   - `ip cef` (enables Cisco Express Forwarding for faster routing)
   - `no ipv6 cef` (disables IPv6 CEF since not using yet)
5. **Configure WLC Management**:
   - Access WLC GUI (click device > GUI tab) or CLI.
   - Set Management Interface: IP 10.0.99.10/24, Gateway 10.0.99.1, VLAN 99.
   - Save and reboot if needed.

![WLC Management Interface](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/wlc-management-interface.png>)
6. **Configure End Devices** (Manual IPs for now—DHCP in Goal 8):
   - On PCs/Laptop/Server: Click device > Desktop > IP Configuration > Static.
     - Assign as per table, including default gateway (e.g., PC1: IP 10.0.10.10, Mask 255.255.255.0, Gateway 10.0.10.1).
     - Optional: DNS 10.0.99.50 (server IP) for future.
   - For IP-Phones: Click > Config > Network > IPv4: Static, assign (e.g., IP-Phone1: 10.0.20.10/24, Gateway 10.0.20.1).
7. **Save Configs**: On all devices, `end`, `wr`.

### Verification
- `show ip interface brief` on each device: Confirm IPs, status up/up (or down for SVIs until Goal 3).

![MLSW1 IP Interface Brief](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-ip-interface-brief.png>)
- `show run | section interface` : Verify configs.
- `show arp` : Check MAC-IP bindings after pings.
- `show ip route` : Static routes only (connected subnets); no dynamic yet.
- Basic pings (within same subnet only—no inter-VLAN routing yet):
  - From MLSW1: Ping 10.0.1.2 (Edge-R)

![MLSW1 to Edge-R Ping Test](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-to-EDGE-R.png>)
  - From Edge-R: Ping 10.0.1.1 (MLSW1) and 10.0.1.5 (MLSW2).

![Edge-R to MLSW1 Ping Test](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/Edge-R-to-MLSW1.png>)
  
![Edge-R to MLSW2 Ping Test](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/Edge-R-to-MLSW2.png>)
  - From ISP-R: Ping 203.0.113.1 (Edge-R).

![ISP-R to Edge-R Ping Test](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/ISP-R-to-Edge-R.png>)
  - Hosts (post-Goal 3 for full VLANs): PC1 ping PC2.

![PC1 to PC2 Ping Test](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/PC1-to-PC2.png>)

### What We Achieved
- Efficient, documented IP plan with VLSM.
- Basic L3 connectivity in segments.
- Preparation for routing (SVIs, subints, loopbacks).
- Demonstrated hardening and optimization.