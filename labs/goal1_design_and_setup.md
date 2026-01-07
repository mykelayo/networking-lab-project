# Goal 1: Design Topology and Basic Setup

This is the foundation of the entire project. In this initial lab, we carefully design an small campus network topology, build it in Cisco Packet Tracer (PT), establish physical connectivity, and apply basic device configurations. No IP addressing or advanced features yet, those come later. The focus here is on creating a stable, well-documented starting point that prevents issues (like crashes or misconfigurations) in future goals.

### Objective
- Design a scalable, redundant small topology that supports all planned CCNA features (VLANs, inter-VLAN routing, HSRP, wireless with WLC, voice with CME, NAT, dynamic routing, security, etc.).
- Add and label all required devices in Packet Tracer.
- Cable devices according to the physical topology with proper link types.
- Perform initial device hardening and basic configuration (hostnames, passwords, banners, console settings, interface activation).
- Verify physical connectivity (link lights) and basic CLI access.
- Document the topology with clear diagrams and notes for future reference.

This ensures a clean, professional starting point before we layer on complexity.

### Why This Topology?
This design mimics a small-to-medium enterprise campus network with distribution/core and access layers:

- **Redundancy**:
  - Dual multilayer switches (MLSW1 & MLSW2) for gateway redundancy (HSRP later).
  - EtherChannel between MLSW1 ↔ MLSW2 for link aggregation and loop prevention.
  - Dual uplinks from Edge-R to both MLSWs (future routing redundancy).
- **Scalability & Feature Support**:
  - Access Switch (ASW) handles end devices (PCs, IP phones with daisy-chained PCs, lightweight AP, server).
  - Wireless LAN Controller (WLC-3504) for centralized wireless management (enterprise best practice over autonomous APs).
  - CME-R (2811) dedicated to voice services (CME/telephony-service) with router-on-a-stick subinterfaces.
  - Edge-R (2911) as border router for NAT/PAT and external connectivity.
  - ISP-R simulates the internet for WAN testing.
- **Layer Separation**:
  - Layer 2: ASW + MLSWs (VLANs, trunks, STP).
  - Layer 3: MLSWs (SVIs for inter-VLAN), Edge-R (NAT, dynamic routing), CME-R (subinterfaces).

- **Real-World Alignment**: Follows Cisco's hierarchical design (access → distribution → core/edge).

### Topology Diagrams

#### Physical Topology
![Physical Topology in Packet Tracer](https://github.com/mykelayo/networking-lab-project/blob/main/topology/physical-topology.png)

#### Logical Topology (High-Level Overview)
![Logical Topology Overview](https://github.com/mykelayo/networking-lab-project/blob/main/topology/logical-topology.png)

#### Text-Based Physical Connections Table
| Source Device       | Source Port       | Destination Device | Destination Port     | Link Type              | Purpose                          |
|---------------------|-------------------|--------------------|----------------------|------------------------|----------------------------------|
| MLSW1               | Fa0/1             | Edge-R             | Gi0/0                | Copper Straight        | P2P routed link                  |
| MLSW2               | Fa0/1             | Edge-R             | Gi0/1                | Copper Straight        | P2P routed link (redundancy)     |
| MLSW1               | Fa0/2–3           | MLSW2              | Fa0/2–3              | Copper Straight        | EtherChannel trunk (core link)   |
| MLSW1               | Fa0/5             | WLC                | Ethernet1            | Copper Straight        | WLC management/trunk             |
| ASW                 | Fa0/1             | MLSW1              | Fa0/4                | Copper Straight        | Access-to-distribution trunk     |
| ASW                 | Fa0/2             | MLSW2              | Fa0/4                | Copper Straight        | Access-to-distribution trunk     |
| ASW                 | Fa0/3             | IP-Phone1          | Ethernet             | Copper Straight        | Voice + Data (PC1 daisy-chained)  |
| IP-Phone1           | PC Port           | PC1                | Ethernet             | Copper Straight        | Daisy-chain PC behind phone       |
| ASW                 | Fa0/4             | IP-Phone2          | Ethernet             | Copper Straight        | Voice + Data (PC2 daisy-chained)  |
| IP-Phone2           | PC Port           | PC2                | Ethernet             | Copper Straight        | Daisy-chain PC behind phone       |
| ASW                 | Fa0/5             | LWAP               | Ethernet             | Copper Straight (PoE)  | Lightweight AP connection        |
| ASW                 | Fa0/6             | CME-R              | Gi0/0                | Copper Straight        | Trunk for subinterfaces          |
| ASW                 | Fa0/7             | Server             | Ethernet             | Copper Straight        | DNS/Syslog/DHCP server           |
| Edge-R              | Serial0/3/0       | ISP-R              | Serial0/1/0          | Serial DCE/DTE         | WAN simulation                   |

### Devices Summary Table
| Device   | Model       | Role                              | Key Features Used Later                  |
|----------|-------------|-----------------------------------|------------------------------------------|
| MLSW1    | 3560        | Distribution/Core (Primary)       | SVIs, HSRP Active (VLAN 10/20), OSPF     |
| MLSW2    | 3560        | Distribution/Core (Standby)        | SVIs, HSRP Standby/Active (VLAN 30/99)   |
| ASW      | 2960        | Access Layer                      | VLANs, voice VLAN, port security         |
| CME-R    | 2811        | Voice Router                      | CME, subinterfaces, DHCP pools           |
| Edge-R   | 2911        | Border Router                     | NAT/PAT, dynamic routing, ACLs           |
| ISP-R    | 1941        | Internet Simulator                | Static routes, loopback                  |
| WLC      | 3504        | Wireless Controller               | WLANs, CAPWAP, Option 43                 |
| LWAP     | LAP-PT      | Lightweight Access Point          | Registered to WLC                        |
| PC1/PC2  | PC-PT       | End Devices (Data VLAN)           | Testing, ping sources                    |
| IP-Phone1/2 | IP Phone 7960 | Voice Endpoints                | CME registration, voice VLAN             |
| Laptop   | Laptop-PT   | Wireless Client                   | Associates to WLAN                        |
| Server   | Server-PT   | Utility Server                    | DNS, Syslog, DHCP (optional)             |

### Steps in Packet Tracer

#### 1. Create New Project and Add Devices
- Open Cisco Packet Tracer
- Add devices exactly as per the table above.
- **Label every device** clearly:
  - Right-click device → Inspect → Notes or Label → Enter name (e.g., "MLSW1").

#### 2. Cable the Topology
- Use the connections table above.
- Tips:
  - Serial link: Place DCE cable end on ISP-R → auto-prompt for `clock rate 64000`.
  - Phone daisy-chain: Connect ASW → Phone Ethernet port, then Phone PC port → PC.
  - Power on all devices (click power button if off).

#### 3. Basic Configuration on All Devices
Apply this on **every router and switch** (MLSW1, MLSW2, ASW, CME-R, Edge-R, ISP-R):

```
enable
configure terminal

hostname <DEVICE-NAME> 

enable secret cisco

username admin privilege 15 secret cisco

service password-encryption

no ip domain-lookup     
ip domain-name lab.local

clock timezone WAT 1   

banner motd #
**********************************************************
   Welcome to My Networking Lab Project
   Unauthorized access prohibited!
   CCNA 200-301 Prep Homelab
**********************************************************
#

line con 0
 login local
 exec-timeout 30 0
 logging synchronous
line vty 0 15
 login local
 exec-timeout 30 0
 logging synchronous
 transport input ssh 

! Bring up all interfaces
interface range fa0/1 - 24 , gi0/0 - 1
 no shutdown

end
write memory
```

#### 4. WLC Initial Setup (Limited in PT)
- Click WLC → CLI tab or GUI tab.
- If wizard appears, set:
  - Management IP: Leave for Goal 2.
  - System Name: WLC
- Skip advanced config for now.

### Verification
- **Link Lights**: All connections should show green (up/up).
![Link Lights](https://github.com/mykelayo/networking-lab-project/blob/main/topology/logical-topology.png)
- **CLI Access**: Console into each device → `show run | include hostname` → Confirm correct name. 
![MLSW1 hostname](https://github.com/mykelayo/networking-lab-project/blob/main/topology/hostname.png)

### What We Achieved
- Professional, redundant, feature-rich topology built and documented.
- All devices configured with secure basic settings.
- Clean starting point for progressive configuration.
- Visual and textual documentation for portfolio impact.