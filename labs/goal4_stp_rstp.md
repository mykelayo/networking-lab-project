# Goal 4: STP/RSTP

This goal will focus on understanding and manipulating STP/RSTP behavior to ensure a loop-free Layer 2 topology, optimize convergence, and demonstrate redundancy in action.

### Objective
- Verify default STP/RSTP behavior in the topology.
- Identify the current root bridge and port roles.
- Optimize STP by electing the preferred root bridge (MLSW1 as primary, MLSW2 as secondary).
- Configure Rapid Per-VLAN Spanning Tree (Rapid-PVST) for faster convergence.
- Enable advanced features: PortFast, BPDUGuard, and Root Guard on appropriate ports.
- Verify loop prevention and fast recovery.

Now that we have redundant links (MLSW1 ↔ MLSW2 EtherChannel, single trunks from ASW), STP will block redundant paths to prevent loops while allowing quick failover.

### Why These Configurations?
- **Predictable Root Bridge**: Manually setting the root prevents suboptimal defaults (e.g., ASW becoming root due to lowest MAC).
- **Rapid-PVST**: Provides per-VLAN spanning tree with much faster convergence (~seconds vs. 30–50s in legacy STP).
- **Root Guard**: Prevents unexpected switches (e.g., a rogue device) from becoming root.
- **PortFast + BPDUGuard**: Keeps end devices from participating in STP while protecting against loops if someone plugs in a switch.
- **Testing Failover**: Proves the value of redundancy, critical for enterprise networks and CCNA understanding.

### Our Current Topology Recap
- Redundant paths:
  - MLSW1 ↔ MLSW2 (EtherChannel Po1 — treated as one logical link by STP)
  - ASW → MLSW1 (Fa0/1 trunk)
  - ASW → MLSW2 (Fa0/2 trunk)
- This creates a classic triangle loop. STP will block one ASW uplink.

### Steps in Packet Tracer

#### 1. Verify Default Behavior (Before Changes)
On **all switches**, run and screenshot:
```
show spanning-tree root
show spanning-tree vlan 10 detail
```
![ASW spanning-tree vlan 10](https://github.com/mykelayo/networking-lab-project/blob/main/topology/spanning-tree-vlan-10-before.png)

Note the current root bridge (the switch with lowest MAC, which is ASW).

#### 2. Configure Rapid-PVST Globally
On **MLSW1, MLSW2, and ASW**:
```
conf
spanning-tree mode rapid-pvst
```

#### 3. Elect Preferred Root Bridges
We want **MLSW1 as primary root** for all VLANs, **MLSW2 as secondary**.

On **MLSW1** (Primary Root):
```
conf
spanning-tree vlan 10,20,30,99 root primary
```
(This automatically sets priority to 24576)

On **MLSW2** (Secondary Root):
```
conf
spanning-tree vlan 10,20,30,99 root secondary
```
(This sets priority to 28672)

On **ASW**: No need, it will have default priority (32768)

#### 4. Add Root Guard on Downlink Ports
Protect distribution-to-access links from superior BPDUs.

On **MLSW1** (port to ASW):
```
conf
interface FastEthernet0/4
 spanning-tree guard root
```

On **MLSW2** (port to ASW):
```
conf
interface FastEthernet0/4
 spanning-tree guard root
```

#### 5. Add PortFast and BPDUGuard on Access Ports

On **ASW**:
```
conf t
! PC1 and PC2
 interface range FastEthernet0/3 - 4
 spanning-tree portfast
 spanning-tree bpduguard enable

! Server
 interface FastEthernet0/7
 spanning-tree portfast
 spanning-tree bpduguard enable

! LWAP (Lightweight AP)
 interface FastEthernet0/5
 spanning-tree portfast
 spanning-tree bpduguard enable
```

#### 6. Save Configurations
On all switches:
```
end
write memory
```

### Verification Commands

1. **Root Bridge Confirmation**
   On MLSW1:
   ```
   show spanning-tree vlan 10
   ```
   ![MLSW Root Bridge Confirmation](https://github.com/mykelayo/networking-lab-project/blob/main/topology/spanning-tree-vlan-10-after.png)

2. **Port Roles Summary**
   ```
   show spanning-tree summary
   ```
	![MLSW Port Roles Summary](https://github.com/mykelayo/networking-lab-project/blob/main/topology/spanning-tree-sum.png)

4. **Interface Status**
   ```
   show interfaces trunk
   ```
   ![MLSW Interface Status](https://github.com/mykelayo/networking-lab-project/blob/main/topology/spanning-tree-trunk.png)

### What We Achieved
- Predictable, optimized spanning-tree topology with MLSW1 as root.
- Rapid convergence using RSTP (Rapid-PVST mode).
- Protected access ports with PortFast + BPDUGuard.
- Foundation for HSRP, consistent root ensures stable forwarding.

### Potential Issues & Tips
- **Slow convergence?** Ensure all switches are in `rapid-pvst` mode.
- **Wrong root elected?** Check priorities with `show span active`, lower number wins.
- **EtherChannel issues?** STP treats Po1 as single link.
- **Port stuck in blocking?** Normal, one ASW uplink will be Alternate (discarding).
- **PT simulation**: RSTP convergence is fast in real gear, but PT may show slight delay.