# Goal 2: Subnetting and IP Configuration
Now that the topology is designed and devices are basically set up, we'll move to subnetting and IP addressing. This lab focuses on planning the IP address scheme, calculating subnets, and assigning static IP addresses to interfaces, loopbacks, and hosts. We'll use a private address space for the internal network and simulate public IPs for the WAN link to ISP-R. No dynamic routing or VLANs yet, that's later, so we'll treat trunk links as access ports temporarily for IP assignment where needed (we'll change them to trunks in Goal 3). This allows basic connectivity testing via pings.

### Objective
- Design an IP addressing scheme using subnetting to efficiently allocate addresses for VLANs, point-to-point links, management, and loopbacks. 
- Assign IPs to router interfaces, switch SVIs (even though VLANs aren't configured yet, we'll create them in Goal 3, but planning ahead), subinterfaces on CME-R (for router-on-a-stick in voice VLAN), and end devices. 
- Verify Layer 3 connectivity within local segments.

This ensures all devices have IPs before we add VLANs, trunks, and routing, making troubleshooting easier step by step.

### Why This IP Scheme?
- **Efficiency with VLSM**: We'll use Variable Length Subnet Masking (VLSM) on a /16 network to create subnets of different sizes, larger for host VLANs (e.g., /24 for up to 254 hosts), smaller for point-to-point links (/30 for 2 hosts), and /32 for loopbacks (stable OSPF router IDs).
- **Scalability**: Pre-plans for VLANs (Data, Voice, Wireless, Management) to support inter-VLAN routing later. Management VLAN keeps admin traffic separate.
- **Redundancy**: Gateways will use HSRP virtual IPs (e.g., .1 for virtual gateway in each subnet), with MLSW1/MLSW2 as .2/.3.
- **Feature Coverage**: Subinterfaces on CME-R 802.1Q encapsulation (even if not strictly needed with MLSW SVIs, it shows router-on-a-stick). Loopbacks for OSPF stability.
- **Private Addressing**: Using 10.0.0.0/16 internally; simulate public with 203.0.113.0/24 for ISP side (NAT).

#### Subnetting Process
We'll subnet 10.0.0.0/16 (65,534 hosts available). Requirements:
- 4 host VLANs: Data (VLAN 10, need ~50 hosts → /24), Voice (VLAN 20, ~20 hosts → /26, but /24 for simplicity), Wireless (VLAN 30, ~50 hosts → /24), Management (VLAN 99, ~10 devices → /28, but /24 for growth).
- Point-to-point links: MLSW1-Edge-R (/30), MLSW2-Edge-R (/30), Edge-R-ISP-R (/30).
- Loopbacks: /32 for each router/MLSW (stable IDs).
- Reserve space for future (e.g., IPv6 in Goal 14).

Step-by-step subnetting:
1. Start with largest subnets first (VLSM best practice).
   - VLAN 10 (Data): 10.0.10.0/24 (hosts 10.0.10.1-254, gateway 10.0.10.1 virtual).
   - VLAN 20 (Voice): 10.0.20.0/24 (hosts 10.0.20.1-254, gateway 10.0.20.1).
   - VLAN 30 (Wireless): 10.0.30.0/24 (hosts 10.0.30.1-254, gateway 10.0.30.1).
   - VLAN 99 (Management): 10.0.99.0/24 (hosts 10.0.99.1-254, gateway 10.0.99.1).
2. Point-to-point:
   - MLSW1-Edge-R: 10.0.1.0/30 (MLSW1: 10.0.1.1, Edge-R: 10.0.1.2).
   - MLSW2-Edge-R: 10.0.1.4/30 (MLSW2: 10.0.1.5, Edge-R: 10.0.1.6).
   - Edge-R-ISP-R: 203.0.113.0/30 (Edge-R: 203.0.113.1, ISP-R: 203.0.113.2), simulates public.
3. Loopbacks (for OSPF router IDs):
   - MLSW1 Lo0: 1.1.1.1/32
   - MLSW2 Lo0: 2.2.2.2/32
   - CME-R Lo0: 3.3.3.3/32
   - Edge-R Lo0: 4.4.4.4/32
   - ISP-R Lo0: 5.5.5.5/32 (for testing).

For CME-R: Since it's trunked to ASW, we'll add subinterfaces for VLANs 20 (Voice) and 99 (Management) for router-on-a-stick, even if MLSWs handle main routing. CME-R Gi0/0.20: 10.0.20.254/24 (backup gateway for CME), Gi0/0.99: 10.0.99.254/24.

WLC Management: 10.0.99.10/24 (configured in WLC interface).

Hosts:
- PC1 (Data): 10.0.10.10/24, gateway 10.0.10.1
- PC2 (Data): 10.0.10.11/24, gateway 10.0.10.1
- IP-Phone1 (Voice): DHCP later
- IP-Phone2 (Voice): DHCP later
- Laptop (Wireless): DHCP later, manual for now: 10.0.30.10/24
- Server (Management): 10.0.99.50/24

![Logical Topology](https://github.com/mykelayo/networking-lab-project/blob/main/topology/logical-topology.png)

```
- VLAN 10 (Data): 10.0.10.0/24, Gateway: 10.0.10.1 (HSRP Virtual)
- VLAN 20 (Voice): 10.0.20.0/24, Gateway: 10.0.20.1
- VLAN 30 (Wireless): 10.0.30.0/24, Gateway: 10.0.30.1
- VLAN 99 (Management): 10.0.99.0/24, Gateway: 10.0.99.1
- P2P MLSW1-Edge-R: 10.0.1.0/30
- P2P MLSW2-Edge-R: 10.0.1.4/30
- P2P Edge-R-ISP-R: 203.0.113.0/30
- Loopbacks: As above
```

### Steps in Packet Tracer
1. **Plan and Document**: Review the subnetting above.

2. **Configure Switch SVIs** (on MLSW1 and MLSW2—prep for inter-VLAN):
   - On MLSW1/MLSW2: `conf t`
     - `ip routing` (enable L3 routing)
     - `interface vlan 10`, `ip address 10.0.10.2/24` (MLSW1) or `10.0.10.3/24` (MLSW2), `no shut`
     - Repeat for VLAN 20: 10.0.20.2/24 or .3
     - VLAN 30: 10.0.30.2/24 or .3
     - VLAN 99: 10.0.99.2/24 or .3
     - Note: VLANs not created yet (Goal 3), but SVIs can be configured, they'll be down until then.

3. **Configure Router Interfaces**:
   - On Edge-R:
     - `int gi0/0`, `ip add 10.0.1.2/30`, `no shut` (to MLSW1)
     - `int gi0/1`, `ip add 10.0.1.6/30`, `no shut` (to MLSW2)
     - `int serial0/3/0`, `ip add 203.0.113.1/30`, `no shut`
     - `int lo0`, `ip add 4.4.4.4 255.255.255.255`
   - On CME-R:
     - `int fa0/0`, `no shut` (main int, no IP)
     - `int fa0/0.20`, `encapsulation dot1Q 20`, `ip add 10.0.20.254/24`
     - `int fa0/0.99`, `encapsulation dot1Q 99`, `ip add 10.0.99.254/24`
     - `int lo0`, `ip add 3.3.3.3 255.255.255.255`
   - On ISP-R:
     - `int serial0/1/0`, `ip add 203.0.113.2/30`, `no shut`, `clock rate 64000` (DCE)
     - `int lo0`, `ip add 5.5.5.5 255.255.255.255`
   - On MLSW1/MLSW2 (P2P interfaces, not SVIs):
     - MLSW1: `int fa0/1`, `no switchport`, `ip add 10.0.1.1/30`, `no shut`
     - MLSW2: `int fa0/1`, `no switchport`, `ip add 10.0.1.5/30`, `no shut`
     - `int lo0`, `ip add 1.1.1.1/32` (MLSW1) or `2.2.2.2/32` (MLSW2)

4. **Additional Hardening (All Routers/Switches)**:
   - `ip domain-name lab.local`
   - `no ip domain-lookup`
   - `ip cef` (optimize routing)
   - `no ipv6 cef` (disable, not using yet)

5. **Configure WLC Management**:
   - Access WLC GUI (click device > GUI tab) or CLI.
   - Set Management Interface: IP 10.0.99.10/24, Gateway 10.0.99.1, VLAN 99.
   - Save and reboot if needed.
 
   ![WLC Management Interface](https://github.com/mykelayo/networking-lab-project/blob/main/topology/wlc-management-interface.png)   

6. **Configure End Devices** (Manual IPs for now, DHCP in Goal 8):
   - On PCs/Laptop/Server: Click device > Desktop > IP Configuration > Static.
     - Assign as per hosts above.

7. **Save Configs**: On all devices, `end`, `wr`.

### Verification
   - `show ip interface brief` on each device: Confirm IPs assigned, interfaces up/up.

   ![MLSW1 IP Interface Brief](https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-ip-interface-brief.png)
   - Basic pings (within same subnet only, no routing yet):
   - From MLSW1: Ping 10.0.1.2 (Edge-R)

   ![MLSW1 to Edge-R Ping Test](https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-to-EDGE-R.png)
   - From Edge-R: Ping 10.0.1.1 and 10.0.1.5.

   ![Edge-R to MLSW1 Ping Test](https://github.com/mykelayo/networking-lab-project/blob/main/topology/Edge-R-to-MLSW1.png)

   ![Edge-R to MLSW2 Ping Test](https://github.com/mykelayo/networking-lab-project/blob/main/topology/Edge-R-to-MLSW2.png)
   - From ISP-R: Ping 203.0.113.1

   ![ISP-R to Edge-R Ping Test](https://github.com/mykelayo/networking-lab-project/blob/main/topology/ISP-R-to-Edge-R.png)

   - Hosts: Once VLANs/trunked, ping within VLAN (e.g., PC1 ping PC2).
   
   ![PC1 to PC2 Ping Test](https://github.com/mykelayo/networking-lab-project/blob/main/topology/PC1-to-PC2.png)

### Potential Issues and Tips
- SVIs Down: Normal until VLANs created (Goal 3), they'll come up then.
- Subnet Overlaps: Double-check calculations to avoid.
- Why Loopbacks Early? They don't depend on physical links, perfect for router IDs in dynamic routing.
- Why Subinterfaces on CME-R? Demonstrates encapsulation, useful for voice traffic handling.