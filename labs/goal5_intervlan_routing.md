# Goal 5: Inter-VLAN Routing

With VLANs fully established, trunks configured, and Layer 2 stability achieved through **Rapid-PVST** and deterministic root bridge election (Goal 4), we now implement **inter-VLAN routing**. This goal focuses on enabling Layer 3 communication across VLANs on **MLSW1** and **MLSW2** using **Switch Virtual Interfaces (SVIs)**, and demonstrating **router-on-a-stick** inter-VLAN routing via **CME-R**. 

The end devices in different VLANs (e.g., a PC in Data VLAN pinging WLC in Wireless VLAN) will be able to communicate.

### Objective
- Activate and verify SVIs for VLANs 10, 20, 30, and 99 on both MLSWs.
- Confirm SVIs transition from down/down to up/up after VLANs and active ports exist.
- Enable **ip routing** globally on both multilayer switches (we have already done that, confirming again because it is required for L3 forwarding).
- Validate **router-on-a-stick** inter-VLAN routing via CME-R subinterfaces as an alternative path (useful for voice traffic).
- Verify full inter-VLAN connectivity with pings from end devices.
- Observe routing tables, ARP resolution, and connectivity paths.
- Prepare for **HSRP configuration** (Goal 12) by documenting current default gateways.

### Why This Configuration?
- **SVIs on Multilayer Switches (Preferred Method)**:
  - Distributed routing: Each MLSW performs local inter-VLAN forwarding (faster, no bottleneck).
  - Leverages hardware ASIC switching for line-rate performance.
  - Eliminates bottlenecks created by a single router forwarding all VLAN traffic.
  - Aligns with modern design (Layer 3 to the access layer when possible).
  - Enables future HSRP for gateway redundancy without requiring a separate router.
- **Router-on-a-Stick on CME-R**:
  - Classic method using subinterfaces and 802.1Q trunking.
  - Demonstrates encapsulation, subinterface configuration, and how a router can route between VLANs.
  - Provides a path for voice traffic (phones can reach CME-R directly).
- **No Routing Protocol Yet**: We are using only directly connected routes. Dynamic routing (static/default → OSPF/EIGRP) comes in Goals 6 to 7.
- **Redundancy**: Both MLSWs have SVIs with IPs (.2 and .3). Later, HSRP will create virtual gateways (.1) with MLSW1 active for VLANs 10/20 and MLSW2 active for 30/99, matching our STP root election.

### Current State
- VLANs 10, 20, 30, 99 created and named on all switches.
- Access ports assigned (PCs in VLAN 10, phones voice VLAN 20, LWAP & Server in VLAN 99).
- Trunks configured with allowed VLANs and native VLAN 99.
- SVIs already configured in Goal 2 with IPs:
  - MLSW1: .2 in each VLAN
  - MLSW2: .3 in each VLAN
- CME-R subinterfaces already configured:
  - Gi0/0.20 → 10.0.20.254/24
  - Gi0/0.99 → 10.0.99.254/24
- End devices have static IPs and default gateways pointing to future HSRP virtual IP (.1) but for now, they won't have full connectivity until we adjust or use SVI IPs temporarily.

### Steps in Packet Tracer

#### 1. Enable IP Routing on Multilayer Switches
On **MLSW1** and **MLSW2** (this is critical, without this, SVIs won't route. we have done this while configuring the ip addresses): MLSW-ip-route.png
```
conf t
ip routing
end
write memory
```

Verify:
```
show ip route
```

![MLSW1 IP Route](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-ip-route.png>)
![MLSW2 IP Route](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW2-ip-route.png>)

→ Routes are **Connected** for all four VLAN subnets (C entries).

#### 2. Verify SVIs Come Up
On **MLSW1** and **MLSW2**:
```
show ip interface brief | include Vlan
```

![MLSW1 VLAN Interfaces](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-vlan-interfaces.png>)
![MLSW2 VLAN Interfaces](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW2-vlan-interfaces.png>)

→ All VLAN interfaces are **up/up**.

#### 3. Temporarily Adjust Host Default Gateways
Since HSRP isn't configured yet, hosts pointing to .1 won't reach the network.

**Temporary we will fix that for testing** (then revert or automate with DHCP later):
- On PC1 and PC2: Change default gateway to **10.0.10.2** (MLSW1)
- On Server: Change to **10.0.99.3** (MLSW2)

#### 4. No Changes Needed on CME-R
Subinterfaces and encapsulation already configured in Goal 2. The trunk from ASW Fa0/6 carries VLANs 20 and 99, so CME-R can route between Voice and Management if needed.

#### 5. Save Configurations
On MLSW1 and MLSW2:
```
end
write memory
```

### Verification Commands

#### On MLSW1 and MLSW2
1. **SVI Status**
   ```
   show ip interface brief
   ```

![MLSW1 SVI Status](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-SVI-status.png>)
![MLSW2 SVI Status](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW2-SVI-status.png>)


2. **Routing Table**
   ```
   show ip route
   ```
![MLSW1 Routing Table](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-routing-table.png>)

3. **ARP Table (after pings)**
   ```
   show arp | include Vlan
   ```

![MLSW1 ARP Table](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-arp-table.png>)

   → Entries from different VLANs (proof of inter-VLAN communication)

#### Inter-VLAN Ping Tests
From **PC1 (10.0.10.10, VLAN 10)** → Command Prompt:
- ping 10.0.99.50 (Server in VLAN 99)
- ping 10.0.99.10 (WLC in VLAN 99)

![PC1/Server Inter-Vlan Communication](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/pc1-srv-ping.png>)
![PC1/WLC Inter-Vlan Communication](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/pc1-wlc-ping.png>)

From **Server**:
- ping 10.0.20.254 (CME-R in VLAN 20)

![Server/CME-R Inter-Vlan Communication](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/srv-cme-ping.png>)

#### Router-on-a-Stick Verification (on CME-R)
```
show ip interface brief
```

![CME-R IP Interface](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/cme-r-ip-int.png>)

```
show ip route
```

![CME-R IP Route](<https://github.com/mykelayo/networking-lab-project/blob/main/topology/cme-r-ip-route.png>)

### What We Achieved
1. **Full Inter-VLAN Connectivity**
   - Devices in different VLANs communicate via SVIs on MLSWs.
   - PCs, laptops, phones, server, and WLC can all ping each other across VLANs.
2. **Demonstrated Both Routing Methods**
   - **Distributed Layer 3 (SVIs on MLSWs)**: Fast, line-rate routing.
   - **Router-on-a-Stick (CME-R)**: 802.1Q subinterface routing for specific VLANs.
3. **Routing Tables Populated**
   - Connected routes for VLANs visible on both MLSWs and CME-R.
   - ARP entries confirm Layer 3 MAC-to-IP resolution across VLANs.
4. **Foundation for Redundancy and HSRP**
   - SVIs ready for virtual gateway deployment in Goal 12.
   - Matches STP root election (traffic flows efficiently across uplinks).
5. **Network Verification and Documentation**
   - All VLANs verified up/up.
   - Default gateways temporarily adjusted to enable full connectivity.
   - Inter-VLAN routing confirmed without dynamic routing protocols.